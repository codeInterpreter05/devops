# Day 28 — Helm & Kustomize: OCI Chart Registries & Helm Best Practices

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** (none)

## Brief

Helm charts used to require a separate, special-purpose "chart repository" (an index.yaml plus a directory of packaged `.tgz` files, served over plain HTTP) — extra infrastructure just to distribute charts. Since Helm 3.8, charts can be pushed to and pulled from **any OCI-compliant registry** — the same registries you already use for container images (ECR, GHCR, Docker Hub, Harbor, ACR) — collapsing chart distribution into infrastructure you already operate and secure.

## Traditional chart repositories vs OCI registries

```bash
# Traditional model: a chart repo is just an HTTP-served index.yaml + tarballs
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install redis bitnami/redis
```

The traditional model requires standing up (or trusting a third party to run) a chart-repository server, and has no native concept of image-registry-style access control, vulnerability scanning, or per-artifact signing that container registries have matured around for years.

```bash
# OCI model: charts are pushed/pulled exactly like container images
helm package ./mychart                                    # produces mychart-1.4.0.tgz
helm push mychart-1.4.0.tgz oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts
helm install myapp oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts/mychart --version 1.4.0
helm pull oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts/mychart --version 1.4.0
```

Note there's no `helm repo add` step for OCI at all — you reference the `oci://` URL directly at install/pull time, since OCI registries don't use Helm's index.yaml concept; each chart version is just another **artifact** (with its own media type) alongside your container images in the same registry namespace.

## Why OCI registries matter operationally

- **One registry, one set of access controls** — the same IAM policies, network restrictions, and audit logging you already have for ECR/container images apply automatically to your Helm charts, instead of maintaining a separate chart-repo's auth model.
- **Image scanning tooling increasingly supports OCI artifacts generally**, not just container images — meaning chart artifacts can be pulled into the same vulnerability/policy scanning pipeline as your images, rather than being an unscanned blind spot.
- **Chart provenance and signing** — OCI's content-addressable, digest-based model pairs naturally with `cosign`/Sigstore-based signing, which is increasingly a supply-chain-security requirement (see also the SBOM/signing themes from software-supply-chain-security phases of a broader DevOps curriculum).
- **Simplifies multi-cluster/multi-account setups** — replicate a single ECR repository across regions/accounts using the exact same tooling (ECR replication, pull-through cache) you'd already use for images, and your charts inherit that same replication story for free.

## Helm best practices worth internalizing

- **Pin chart versions explicitly** in any automated pipeline (`--version 1.4.0`), never `helm install` without a version pin against a mutable chart source — an unpinned `helm upgrade` in CI can silently pick up a newer chart version with breaking changes.
- **Lint and template-render in CI before ever applying**:
  ```bash
  helm lint ./mychart
  helm template ./mychart -f values-prod.yaml | kubeval -           # or kubeconform, validates against k8s schemas
  ```
- **Keep secrets out of `values.yaml`** — plaintext secrets checked into a chart's values files end up in git history and in Helm's release-history storage (Secrets, by default) forever. Use `helm-secrets` (SOPS-encrypted values), External Secrets Operator, or a sealed-secrets approach instead, referencing existing cluster Secrets from templates rather than embedding raw values.
- **Use `--atomic` for safer upgrades in automation**:
  ```bash
  helm upgrade myapp ./mychart -f values-prod.yaml --atomic --timeout 5m
  ```
  `--atomic` automatically rolls back to the previous successful revision if the upgrade fails or times out, rather than leaving a partially-applied, half-upgraded release sitting in the cluster.
- **Use `helm-docs`** to auto-generate a chart's README (values table, description) directly from comments in `values.yaml` — keeps documentation from silently drifting out of sync with the actual chart as it evolves, since it's regenerated from the source of truth rather than hand-maintained separately.

```yaml
# values.yaml, annotated for helm-docs generation
# -- Number of replicas to run
replicaCount: 2
# -- Container image configuration
image:
  # -- Image repository
  repository: myapp
  # -- Image tag, defaults to the chart's appVersion if empty
  tag: ""
```

```bash
helm-docs                 # regenerates README.md from values.yaml comments + a README.md.gotmpl template
```

## Points to Remember

- Since Helm 3.8, `oci://` registries are a first-class chart distribution mechanism — no separate chart-repo server, no `helm repo add`, charts live alongside container images in the same registry.
- OCI-based chart distribution inherits whatever access control, scanning, and replication tooling you already have for container images — a real operational simplification, not just a syntax change.
- Never `helm install`/`upgrade` against an unpinned, mutable chart version in an automated pipeline — always pin an explicit version.
- `--atomic` is the standard safety flag for automated `helm upgrade` — it auto-rolls-back a failed/timed-out upgrade instead of leaving a half-applied release.
- Secrets belong in a proper secrets-management layer (SOPS/`helm-secrets`, External Secrets Operator), never committed in plaintext inside `values.yaml` files that end up in git and in Helm's own release-history storage.

## Common Mistakes

- Committing real credentials/API keys directly into a chart's `values-prod.yaml`, forgetting that Helm's default release-history storage (a Secret per revision, containing the full rendered values) means that data persists in the cluster indefinitely across every revision, not just in git.
- Running `helm upgrade` in CI without `--atomic` (or an equivalent rollback strategy), leaving a cluster in a half-upgraded, partially-rolled-out state after a failed deploy, requiring manual `helm rollback` intervention that could have been automatic.
- Treating OCI chart registries as requiring completely different tooling from container images — in practice it's the same `docker login`/registry credential flow (`helm registry login`) and largely the same mental model, just a different artifact type.
- Skipping `helm lint`/`helm template` validation in CI and only discovering a broken template (bad indentation, missing values key) when `helm upgrade` fails against the real cluster.
- Letting chart documentation (a hand-written README) drift out of sync with the actual `values.yaml` schema over time — `helm-docs`, regenerated as part of CI, keeps this from silently rotting.
