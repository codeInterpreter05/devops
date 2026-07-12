# Day 12 — Container Security: Image Signing with Cosign

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Containers | **Flag:** ⚡ Interview-critical

## Brief

Hardening (file 1) limits what a running container can do, and scanning (file 2) catches known-vulnerable packages before deploy — but neither answers a different question: *is the image that's actually about to run the exact image you built and scanned, unmodified, from a source you trust?* Between "CI builds an image" and "a node pulls and runs it" there are multiple handoffs — push to a registry, pull by a CD system, pull by cluster nodes — and at every hop, without a verification step, nothing stops a tampered or substituted image from slipping through. Image signing closes that gap.

## The supply-chain problem signing solves

Two distinct guarantees are at stake, and it's worth keeping them separate:

- **Authenticity and integrity** — was this exact set of bytes produced by an identity I trust, and has it been altered since? This is what a cryptographic signature over the image content directly answers.
- **Provenance** — what build process, source commit, and pipeline actually produced this artifact? This is answered by attaching signed *attestations* (metadata about the build) alongside the signature, not by the signature alone — frameworks like SLSA (Supply-chain Levels for Software Artifacts) formalize how much of this a given build pipeline can actually prove.

A detail that trips people up: container image **tags** (`myapp:latest`, `myapp:v1.2.0`) are mutable pointers. Anyone with push access to the repository can repoint a tag at a completely different image without anything visibly changing from a consumer's perspective — `myapp:v1.2.0` today is not guaranteed to be the same bytes as `myapp:v1.2.0` yesterday. The **content digest** (`sha256:<hash>`) is what's actually immutable and content-addressed — it's derived from the image's contents, so any change to the image produces a different digest. Signing and verification should always bind to the digest, never the tag, because verifying a mutable pointer defeats the entire purpose: an attacker who can repoint the tag can point it at an unsigned or differently-signed image, and a tag-based verification would never notice.

## Cosign

Cosign is part of the **Sigstore** project, alongside **Fulcio** (a certificate authority that issues short-lived signing certificates based on OIDC identity instead of long-lived keys) and **Rekor** (a public, append-only transparency log recording signing events). Sigstore's goal is making signing practical for ordinary CI pipelines without every team having to run their own PKI.

Cosign supports two signing models:

**1. Traditional key-pair signing** — generate a keypair, protect the private key (in real deployments, backed by a KMS — AWS KMS, GCP KMS, HashiCorp Vault — rather than a bare file on a laptop or build agent), sign with it.

```bash
cosign generate-key-pair
# writes cosign.key (private, passphrase-encrypted) and cosign.pub (public)

cosign sign --key cosign.key myregistry/myapp@sha256:<digest>
cosign verify --key cosign.pub myregistry/myapp@sha256:<digest>
```

**2. Keyless signing** — Sigstore's headline feature, and increasingly the default for CI-driven pipelines. There is no long-lived private key to generate, store, rotate, or leak at all: Cosign requests a short-lived certificate from Fulcio, authenticated via an OIDC identity (a CI provider's native workflow identity token — GitHub Actions and GitLab CI both issue these automatically to jobs — or an interactive Google/GitHub login for local use), signs with an ephemeral key that's discarded the moment signing completes, and records the event in Rekor so the signing action is independently auditable later without having to trust Cosign's local output alone.

```bash
cosign sign myregistry/myapp@sha256:<digest>
# in CI: uses the pipeline's OIDC token automatically, no login prompt
# locally: opens an interactive OIDC login flow

cosign verify \
  --certificate-identity "https://github.com/org/repo/.github/workflows/ci.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  myregistry/myapp@sha256:<digest>
```

Keyless signing removes the "where do we securely store the signing key" problem entirely, at the cost of anchoring trust in Sigstore's public infrastructure and OIDC identity rather than a key you fully control — an acceptable and increasingly standard trade-off, and the reason `cosign verify` for a keyless signature always requires specifying *which* identity and *which* OIDC issuer you expect, rather than just "a valid signature exists." A valid signature from an untrusted identity should verify as untrusted, which is why both flags are mandatory in practice, not optional extras.

Signature storage is the other detail that makes Cosign practical: it doesn't produce a separate file you have to distribute alongside the image yourself. It pushes the signature back to the *same registry* as an OCI artifact, tagged with a predictable name derived from the image's digest. That means the signature travels automatically with the image through every pull, mirror, and replication, and any standard OCI-compliant registry can store it — no special registry feature required, no side channel to keep in sync.

## Admission-time enforcement

A signature nobody checks is documentation, not a control. The step that actually converts "we sign our images" into an enforced guarantee is admission-time verification in the deployment environment — in Kubernetes, this is what admission controllers like **Kyverno** and **OPA/Gatekeeper** are for. Both can call Cosign's verification logic against every Pod's image reference at admission time and reject anything that doesn't carry a valid signature from an expected identity or key.

```yaml
# Kyverno ClusterPolicy (abridged)
spec:
  rules:
    - name: verify-image-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences: ["myregistry/myapp:*"]
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/org/repo/.github/workflows/ci.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
```

Without a policy like this actually wired into the cluster's admission chain, signing is optional in the most literal sense — nothing stops an unsigned, unverified, or outright tampered image from being deployed by mistake or by an attacker with cluster access. The same mechanism generalizes past bare signatures: Cosign can attach arbitrary signed **attestations** (`cosign attest`) to an image — an SBOM, a vulnerability-scan result, SLSA provenance — and admission policies can require those too, e.g. "only deploy images Trivy scanned with zero CRITICAL findings," turning the scanning step from the previous file into an enforced gate rather than a report nobody reads.

## Points to Remember

- Signing answers "is this the exact artifact I trust, unmodified" — provenance (what built it) is a separate, related guarantee provided by attestations, not the signature alone.
- Always sign and verify by content **digest**, never by tag — tags are mutable pointers and can be repointed after signing.
- Keyless signing (Fulcio + OIDC + Rekor) removes long-lived private key management entirely and is the standard pattern in modern CI; key-pair signing is still valid, especially where you need signing independent of a specific CI provider's OIDC support.
- Cosign stores signatures as OCI artifacts in the same registry as the image, keyed off the image digest — they travel with the image automatically, no separate distribution mechanism needed.
- `cosign verify` on a keyless signature must pin both `--certificate-identity` and `--certificate-oidc-issuer` — a valid signature from an unexpected identity should not verify as trusted.
- Signing without admission-time enforcement (Kyverno/Gatekeeper policies) is not a control — it's a signature nobody is required to check.

## Common Mistakes

- Signing (or verifying) against a tag instead of a digest, which silently breaks the guarantee the moment the tag is repointed.
- Treating `cosign sign` as the finish line and never wiring up `cosign verify` anywhere enforced — signing with no admission-time check is security theater.
- Verifying keyless signatures without constraining `--certificate-identity`/`--certificate-oidc-issuer`, which effectively accepts a signature from *anyone* who's ever authenticated to Sigstore, not just your own pipeline.
- Storing a traditional key-pair private key as a bare file in a CI config or repo secret without KMS backing or rotation — the same "hardcoded credential" mistake as any other long-lived secret, just applied to a signing key.
- Assuming a scanned-clean image is also a signed/verified image — they're independent controls; a clean Trivy scan says nothing about whether the artifact was tampered with after the scan ran.
