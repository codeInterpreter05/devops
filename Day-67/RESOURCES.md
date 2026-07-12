# Day 67 — Resources: Container & Registry Security

## Primary (assigned)

- **Sigstore documentation** (docs.sigstore.dev) — free, the assigned starting point for this day. Covers Cosign, Fulcio, and Rekor and how the keyless signing flow fits together.

## Deepen your understanding

- **Trivy documentation** (aquasecurity.github.io/trivy) — full CLI reference, CI integration guides (GitHub Actions, GitLab CI, Jenkins), and the `.trivyignore` format.
- **Anchore's Syft and Grype docs** (github.com/anchore/syft, github.com/anchore/grype) — how SBOM generation and vulnerability scanning are designed to work together as a pair.
- **AWS ECR image scanning documentation** — the official comparison of basic vs. enhanced scanning, and how to wire enhanced-scanning findings into EventBridge/Security Hub.
- **SLSA framework** (slsa.dev) — while formally a Day 70 topic, it's worth a first skim now since signing and SBOMs are two of the building blocks SLSA levels are built on.

## Reference / lookup

- **CycloneDX and SPDX specifications** — the two SBOM standards; useful to know which fields each format captures when a compliance team asks for a specific one.
- **cosign CLI reference** (docs.sigstore.dev/cosign/overview) — full flag reference for `sign`, `verify`, `attest`, and `verify-attestation`.

## Practice

- Wire the full scan → sign → SBOM chain into a real GitHub Actions workflow against a throwaway public repo, using keyless signing end to end (no local key files) — this is the most realistic rehearsal for how this looks in a production pipeline, and directly extends this day's lab.
- **kind** + Sigstore's `policy-controller` Helm chart — spin up a disposable local cluster and practice writing `ClusterImagePolicy` resources until rejecting an unsigned image at admission feels routine.
