# Day 12 — Cheatsheet: Container Security

## Non-root

```dockerfile
# Dockerfile
RUN groupadd --gid 1000 appgroup \
 && useradd --uid 1000 --gid appgroup --shell /usr/sbin/nologin --create-home appuser
USER appuser          # must come before CMD/ENTRYPOINT
```

```bash
docker run --user 1000:1000 myimage        # override/enforce at run time
```

```yaml
# Kubernetes
securityContext:
  runAsNonRoot: true    # refuses to start if effective user is root
  runAsUser: 1000
```

## Read-only root filesystem

```bash
docker run --read-only --tmpfs /tmp --tmpfs /var/run myimage
```

```yaml
# Kubernetes
securityContext:
  readOnlyRootFilesystem: true
volumes:
  - name: tmp
    emptyDir: {}
```

## Capabilities

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myimage   # start at zero, add back only what's proven needed
docker run --privileged myimage                                 # NEVER in production — disables all isolation
```

```yaml
# Kubernetes
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

Common capabilities:

```
CAP_NET_BIND_SERVICE   bind ports < 1024 without full root
CAP_CHOWN              change file ownership
CAP_DAC_OVERRIDE       bypass file permission checks
CAP_SYS_ADMIN          admin grab-bag (mount, etc.) — red flag if requested
CAP_SYS_PTRACE         trace/inspect other processes
CAP_NET_RAW            raw sockets / packet crafting
```

## Seccomp

```bash
docker run --rm alpine grep Seccomp /proc/1/status     # "Seccomp: 2" = filtering active (Docker default)
docker run --security-opt seccomp=unconfined myimage    # disables filtering — debugging only, never prod
docker run --security-opt seccomp=./profile.json myimage  # custom allow/deny syscall list
```

```yaml
# Kubernetes 1.19+
securityContext:
  seccompProfile:
    type: RuntimeDefault    # or: type: Localhost, localhostProfile: profiles/custom.json
```

## Trivy

```bash
trivy image myapp:latest                                   # full scan
trivy image --severity HIGH,CRITICAL myapp:latest           # filter noise
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest   # CI gate (nonzero exit fails build)
trivy image --ignore-unfixed myapp:latest                   # hide CVEs with no fix available yet
trivy image --format json myapp:latest > scan.json          # machine-readable output
trivy fs .                                                   # scan a repo/filesystem directly
trivy config Dockerfile                                       # IaC misconfig scan
trivy image --skip-db-update myapp:latest                    # skip DB refresh (airgapped/rate-limited CI)
```

## Grype

```bash
syft myapp:latest -o json > sbom.json      # generate SBOM first (Syft, Grype's sibling tool)
grype sbom:sbom.json                        # scan the SBOM
grype myapp:latest                          # or scan the image directly (runs Syft internally)
grype myapp:latest --fail-on high           # CI gate
```

## Docker Scout

```bash
docker scout cves myapp:latest
docker scout cves --only-severity critical,high myapp:latest
docker scout compare myapp:latest --to myapp:previous     # diff CVEs between two tags
docker scout recommendations myapp:latest                  # suggests smaller/patched base images
```

## Cosign

```bash
# Key-pair signing
cosign generate-key-pair                                       # writes cosign.key + cosign.pub
cosign sign --key cosign.key myregistry/myapp@sha256:<digest>
cosign verify --key cosign.pub myregistry/myapp@sha256:<digest>

# Keyless signing (OIDC via Fulcio, logged to Rekor)
cosign sign myregistry/myapp@sha256:<digest>
cosign verify \
  --certificate-identity "https://github.com/org/repo/.github/workflows/ci.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  myregistry/myapp@sha256:<digest>

# Attestations (SBOM, scan results, provenance)
cosign attest --key cosign.key --predicate sbom.json --type spdx myregistry/myapp@sha256:<digest>
cosign verify-attestation --key cosign.pub myregistry/myapp@sha256:<digest>
```

Rule of thumb: **always sign and verify by digest (`sha256:...`), never by tag** — tags are mutable and can be repointed after signing.

## Admission-time enforcement (Kubernetes)

```yaml
# Kyverno ClusterPolicy (abridged) — reject Pods with unverified images
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
