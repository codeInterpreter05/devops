# Day 119 — Resources: Zero Trust & mTLS

## Primary (assigned)

- **SPIFFE documentation** (spiffe.io/docs) — the assigned starting point. Covers the SPIFFE ID/SVID specification, the concepts pages, and links straight into SPIRE's own docs for the reference implementation.

## Deepen your understanding

- **NIST SP 800-207 — Zero Trust Architecture** (csrc.nist.gov) — the closest thing to an authoritative spec for zero trust principles; worth skimming directly rather than only secondhand summaries, since interviewers who know it will notice if your framing drifts from it.
- **BeyondCorp research papers, Google** ("BeyondCorp: A New Approach to Enterprise Security" and the follow-up papers, available via research.google) — the original source material for the BeyondCorp model, written by the team that built it.
- **SPIRE Kubernetes Quickstart tutorial** (github.com/spiffe/spire-tutorials, `k8s/quickstart`) — the exact manifests used in today's Lab 2; also has a follow-on tutorial demonstrating federation between two SPIRE deployments.
- **Istio documentation — Security** (istio.io/latest/docs/concepts/security and .../tasks/security/authentication) — covers `PeerAuthentication`, `AuthorizationPolicy`, and how Istio's identity model maps directly onto SPIFFE.
- **cert-manager documentation** (cert-manager.io/docs) — `Issuer`/`ClusterIssuer` types, the `Certificate` CRD reference, and the `csi-driver-spiffe` project docs for avoiding Secret-based key storage entirely.

## Reference / lookup

- `spire-server entry create --help` / `spire-agent api fetch --help` — the exact flags for registration entries and SVID fetching, faster to check locally than searching docs mid-task.
- **Kubernetes NetworkPolicy reference** (kubernetes.io/docs/concepts/services-networking/network-policies) — the authoritative field reference for `podSelector`, `policyTypes`, `ingress`/`egress` rule shapes.
- **Istio `PeerAuthentication` / `AuthorizationPolicy` API reference** (istio.io/latest/docs/reference/config/security) — exact field-level docs for both CRDs.

## Practice

- **Today's Lab** (`LAB.md`) — default-deny NetworkPolicy micro-segmentation, a real SPIRE server/agent deployment issuing SVIDs, cert-manager-issued mTLS between plain services, and Istio STRICT mTLS with identity-based `AuthorizationPolicy`, all on a local `kind` cluster.
- **Istio's own `httpbin`/`sleep` sample apps** (in the Istio release tarball under `samples/`) — the standard, disposable way to drill mTLS/AuthorizationPolicy behavior without building your own test services.
