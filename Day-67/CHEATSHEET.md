# Day 67 — Cheatsheet: Container & Registry Security

## Trivy

```bash
trivy image myapp:latest                                  # full scan, all severities
trivy image --severity HIGH,CRITICAL myapp:latest         # filter severities
trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 myapp:latest   # CI gate
trivy image --format sarif --output trivy.sarif myapp:latest   # SARIF for GitHub code scanning
trivy fs --severity CRITICAL .                             # scan filesystem/manifests, no image needed
trivy image --ignorefile .trivyignore myapp:latest         # apply accepted-risk exceptions
```

## Grype (Anchore)

```bash
grype myapp:latest                       # scan an image
grype sbom:./sbom.json                    # scan a pre-generated SBOM (no re-pull)
grype myapp:latest --fail-on critical     # CI gate
grype myapp:latest -o json > out.json     # machine-readable output
```

## ECR native scanning

```bash
# Enable enhanced (Inspector-based) continuous scanning account-wide
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{"scanFrequency":"CONTINUOUS_SCAN","repositoryFilters":[{"filter":"*","filterType":"WILDCARD"}]}]'

# Trigger a manual scan / read findings
aws ecr start-image-scan --repository-name myapp --image-id imageTag=latest
aws ecr describe-image-scan-findings --repository-name myapp --image-id imageTag=latest
```
| Mode | Engine | Coverage | Frequency |
|---|---|---|---|
| Basic | Clair | OS packages only | On push (point-in-time) |
| Enhanced | Amazon Inspector | OS + language deps | Continuous |

## Cosign — signing & verifying

```bash
cosign generate-key-pair                                  # key-based (cosign.key / cosign.pub)
cosign sign --key cosign.key myimg@sha256:<digest>          # sign by DIGEST, not tag
cosign sign --yes myimg@sha256:<digest>                     # keyless (needs OIDC in CI)

cosign verify --key cosign.pub myimg@sha256:<digest>
cosign verify \
  --certificate-identity="https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  myimg@sha256:<digest>                                     # keyless verify, pinned identity

cosign attest --yes --predicate sbom.cdx.json --type cyclonedx myimg@sha256:<digest>   # attach SBOM
cosign verify-attestation --type cyclonedx myimg@sha256:<digest>                        # pull it back
```

## Syft — SBOM generation

```bash
syft myapp:latest -o table                  # human-readable
syft myapp:latest -o spdx-json > sbom.spdx.json
syft myapp:latest -o cyclonedx-json > sbom.cdx.json
syft dir:. -o table                          # scan a filesystem, no image build needed
syft myapp:latest --select-catalogers "python,npm"   # filter package types
```

## GitHub Actions permissions needed for keyless signing

```yaml
permissions:
  id-token: write   # required to mint OIDC token for Fulcio
  contents: read
```

## Kubernetes admission — require signed images

```yaml
# Sigstore policy-controller
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
            subject: "https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main"
```

```yaml
# Kyverno verifyImages (alternative)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-signature
      match: { any: [{ resources: { kinds: ["Pod"] } }] }
      verifyImages:
        - imageReferences: ["myregistry.io/*"]
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
```
