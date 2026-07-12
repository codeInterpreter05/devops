# Day 70 — Resources: Artifact Management and Registries

## Primary (assigned)

- **SLSA framework** (slsa.dev) — free, the assigned starting point for this day. The official spec covering all four levels, the threat model each level addresses, and how provenance is generated/verified.

## Deepen your understanding

- **OCI Distribution Spec** (github.com/opencontainers/distribution-spec) — the actual API contract behind every `docker push`/`docker pull`, and the basis for understanding why OCI Artifacts work at all.
- **Helm documentation: "Use OCI-based registries"** (helm.sh/docs) — the official guide to `helm push`/`helm install oci://...` and migrating off classic chart repositories.
- **ORAS project documentation** (oras.land) — the generic OCI artifact client's full reference, useful whenever you need to store something in a registry that has no OCI-aware tooling of its own.
- **Semantic Versioning spec** (semver.org) — the actual, precise specification (not just the "MAJOR.MINOR.PATCH" summary) including pre-release and build metadata rules.
- **Conventional Commits spec** (conventionalcommits.org) — the commit message convention `semantic-release` and similar tools rely on to automate version bumps.

## Reference / lookup

- **Sonatype Nexus Repository docs** and **JFrog Artifactory docs** — official references for configuring hosted/proxy/group repositories per package ecosystem (PyPI, npm, Maven, Docker).
- **GitHub docs: "Using artifact attestations"** — first-party reference for `actions/attest-build-provenance` and verifying attestations with the `gh attestation` CLI.

## Practice

- Push a real Helm chart to a free-tier ECR (or a local `registry:2` container) as done in this day's lab, and wire it into a real ArgoCD `Application` — this is the single most useful hands-on exercise for internalizing OCI-as-generic-storage.
- Set up `semantic-release` against a small throwaway repo with a mix of `feat:`/`fix:`/`feat!:` commits and watch it compute version bumps automatically — seeing it work end to end demystifies the "magic" faster than reading about it.
