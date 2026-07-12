# Day 70 — Cheatsheet: Artifact Management and Registries

## Container registry auth quick reference

```bash
# ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# GHCR
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Docker Hub
docker login -u USERNAME
```

## Helm + OCI

```bash
helm registry login <registry> -u <user> --password-stdin
helm package ./mychart --version 1.4.0
helm push mychart-1.4.0.tgz oci://<registry>/helm-charts
helm install myrelease oci://<registry>/helm-charts/mychart --version 1.4.0   # no `helm repo add` needed
helm pull oci://<registry>/helm-charts/mychart --version 1.4.0
```

## ORAS — generic OCI artifact push/pull

```bash
oras push <registry>/configs/app-config:v1.0.0 \
  file.tar.gz:application/vnd.oci.image.layer.v1.tar+gzip
oras pull <registry>/configs/app-config:v1.0.0
oras repo tags <registry>/configs/app-config
oras manifest fetch <registry>/configs/app-config:v1.0.0
```

## Nexus/Artifactory client config

```bash
# pip via a group repo (hosted + proxy combined)
pip install --index-url https://nexus.example.com/repository/pypi-group/simple mypkg

# npm
npm config set registry https://artifactory.example.com/artifactory/api/npm/npm-group/

# Docker proxy (fixes Docker Hub rate limits org-wide)
docker pull nexus.example.com/docker-proxy/library/node:20
```

## Semantic Versioning

```
MAJOR.MINOR.PATCH   e.g. 2.4.1
MAJOR -> breaking changes
MINOR -> new backward-compatible features
PATCH -> backward-compatible bug fixes

^2.4.0   -> >=2.4.0 <3.0.0   (npm caret: minor/patch updates allowed)
~2.4.0   -> >=2.4.0 <2.5.0   (npm tilde: patch updates only)
```

## Conventional Commits -> semantic-release

```
feat: ...            -> MINOR bump
fix: ...              -> PATCH bump
feat!: ...            -> MAJOR bump
BREAKING CHANGE: ...   (in commit body) -> MAJOR bump
```

```yaml
permissions:
  contents: write
steps:
  - uses: actions/checkout@v4
    with: { fetch-depth: 0 }
  - run: npx semantic-release
```

## SLSA provenance

```yaml
# GitHub Actions native provenance attestation
permissions:
  id-token: write
  attestations: write
  contents: read
steps:
  - uses: actions/attest-build-provenance@v1
    with:
      subject-name: myregistry.io/myapp
      subject-digest: sha256:<digest>
```

```bash
# Cosign equivalent
cosign attest --yes --predicate provenance.json --type slsaprovenance myimage@sha256:<digest>
cosign verify-attestation --type slsaprovenance myimage@sha256:<digest>
```

| SLSA Level | Guarantee |
|---|---|
| 0 | No guarantees, manual builds |
| 1 | Provenance exists (self-reported by build script) |
| 2 | Provenance generated + signed by hosted build platform |
| 3 | Isolated, tamper-resistant, non-forgeable provenance |
