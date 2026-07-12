# Day 67 — Container & Registry Security: Image Signing with Sigstore/Cosign

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** DevSecOps | **Flag:** ⚡ Interview-critical

## Brief

Scanning tells you an image is free of *known* vulnerabilities at scan time. Signing tells you something completely different and equally important: **this exact image was built by a trusted process and hasn't been tampered with since.** Without signing, nothing stops someone (an attacker with registry push access, a compromised CI runner, or a misconfigured mirror) from swapping a clean image for a malicious one with the identical tag. This is the core idea behind supply-chain security post-SolarWinds/post-Log4Shell, and it's why "verify signatures before running anything in the cluster" is now a standard interview question — it tests whether you understand *integrity*, not just *vulnerability* scanning.

## The problem with traditional signing (and why Sigstore exists)

Classic image signing (e.g., Docker Content Trust / Notary) required you to generate, distribute, and — critically — **protect a long-lived private signing key**. Lose that key or leak it, and every signature it ever produced is now untrustworthy; rotating it across every consumer is painful. Most teams found this operationally heavy enough that they simply didn't sign images at all.

**Sigstore** (a Linux Foundation project, with **Cosign** as its main CLI) solves this with **keyless signing**:

1. Instead of a long-lived private key, your CI job authenticates to an OIDC identity provider it already trusts (GitHub Actions' built-in OIDC token, Google/GitLab OIDC, etc.).
2. **Fulcio** (Sigstore's certificate authority) issues a short-lived (minutes) code-signing certificate bound to that verified identity — e.g., "this signing event came from workflow `build.yml` in repo `org/myapp` on the `main` branch."
3. The signature and certificate are recorded in **Rekor**, a public, append-only transparency log (conceptually like Certificate Transparency logs for TLS certs) — giving you an immutable, publicly auditable record of *who signed what, and when*, without anyone needing to manage key material at all.

This means the "key" a verifier trusts isn't a secret you protect — it's a *verifiable identity* plus a public, tamper-evident log entry. Nothing to leak, nothing to rotate.

## Signing an image with Cosign

```bash
# Keyless signing (what most CI pipelines use — relies on OIDC from the CI provider)
cosign sign myregistry.io/myapp@sha256:<digest>

# Traditional key-based signing (if you do need a long-lived key pair)
cosign generate-key-pair                      # produces cosign.key / cosign.pub
cosign sign --key cosign.key myregistry.io/myapp@sha256:<digest>

# Always sign by DIGEST, not by tag — tags are mutable and can be repointed after signing
```

**Why sign by digest, not tag:** a tag like `:latest` or `:v1.2.3` can be re-pushed to point at a different image at any time. The digest (`sha256:...`) is a cryptographic hash of the image's actual content — it's the only identifier that can't be silently swapped out from under a signature.

### GitHub Actions example — keyless signing in CI

```yaml
jobs:
  build-sign-push:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # required: lets the job mint an OIDC token for Fulcio
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: sigstore/cosign-installer@v3

      - name: Build and push
        id: build
        run: |
          docker build -t myregistry.io/myapp:${{ github.sha }} .
          docker push myregistry.io/myapp:${{ github.sha }}
          echo "digest=$(docker inspect --format='{{index .RepoDigests 0}}' myregistry.io/myapp:${{ github.sha }} | cut -d@ -f2)" >> "$GITHUB_OUTPUT"

      - name: Sign image (keyless)
        run: cosign sign --yes myregistry.io/myapp@${{ steps.build.outputs.digest }}
```

`permissions: id-token: write` is the piece people forget — without it, the job has no OIDC token to present to Fulcio and keyless signing fails outright.

## Verifying signatures

```bash
# Verify a keyless signature, pinned to the exact CI identity that should have produced it
cosign verify \
  --certificate-identity="https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  myregistry.io/myapp@sha256:<digest>
```

Pinning `--certificate-identity` and `--certificate-oidc-issuer` is what makes this meaningfully secure — verifying "a signature exists" alone is nearly worthless (anyone can produce *some* keyless signature via their own repo/CI). What matters is that the signature's embedded identity matches *your specific, trusted build pipeline* — no other workflow, repo, or branch should pass verification.

## Verifying signatures in Kubernetes admission

Signing is only enforcement if something *checks* the signature before a Pod is allowed to run. Two common mechanisms:

**1. Sigstore's `policy-controller`** (a Kubernetes admission webhook purpose-built for this):
```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
    - glob: "myregistry.io/**"
  authorities:
    - keyless:
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subject: "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main"
```
Any Pod trying to schedule an image matching `myregistry.io/**` that isn't signed by that exact identity is rejected at admission time — before it ever reaches a node.

**2. Kyverno's `verifyImages` rule** (if you're already using Kyverno for other policies — see Day 68):
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "myregistry.io/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
```

Both approaches mutate the Pod spec to pin the image to its verified digest (not just the tag it was submitted with) — closing the tag-mutability gap described above at the cluster level too.

## Points to Remember

- Scanning = "no known vulnerabilities right now." Signing = "this exact artifact came from a trusted build and wasn't tampered with." You need both — they answer different questions.
- Sigstore/Cosign's keyless flow trades a long-lived secret key for a short-lived certificate (Fulcio) bound to a verified OIDC identity, recorded in a public transparency log (Rekor) — nothing to leak or rotate.
- Always sign and verify by **digest**, never by tag — tags are mutable pointers, digests are immutable content hashes.
- `permissions: id-token: write` in GitHub Actions is required for keyless signing to obtain an OIDC token — a very common thing to forget and get a cryptic Fulcio auth failure for.
- Enforcement requires an admission-time check (`policy-controller` or Kyverno's `verifyImages`) pinned to a *specific* trusted identity — verifying "a signature exists" without checking *whose* signature is nearly meaningless.

## Common Mistakes

- Verifying only that a signature is *present*, without pinning `--certificate-identity`/`--certificate-oidc-issuer` — this passes for literally any signed image from any GitHub repo, not just your trusted pipeline, defeating the purpose.
- Signing/pushing by mutable tag and assuming the signature still applies after a later `docker push` re-points that tag to new content — it doesn't; the signature is bound to the digest that existed at signing time.
- Forgetting `id-token: write` in the CI job's `permissions` block, then being confused why keyless signing fails with an OIDC/identity error in an otherwise-correct pipeline.
- Enabling signature verification in a cluster's admission policy in "Audit" mode and never flipping it to "Enforce" — violations get logged but nothing is actually blocked, giving a false sense of security.
- Treating Rekor's transparency log as a private mechanism — by default, keyless-signing events are recorded in a *public* log. For private/internal images, teams either accept this (the log records the identity/hash, not the image contents) or run a private Sigstore instance if that public record is a compliance concern.
