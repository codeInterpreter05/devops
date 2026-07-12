# Day 70 — Artifact Management: Registries and OCI Artifacts

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** CI/CD

## Brief

Every pipeline eventually produces something that needs to live somewhere durable and pullable: a container image, a Helm chart, a compiled binary, a Python wheel. Registries are the unglamorous but load-bearing infrastructure underneath every deploy — if your registry is unavailable, *nothing* can deploy, no matter how good your pipeline logic is. Understanding registry choices and the newer idea of "OCI artifacts" (using container registries to store things that aren't container images at all) is core infrastructure literacy for any CI/CD-focused role.

This day is split into three focused files:

1. **This file** — container registries (ECR, GCR, Docker Hub, GitHub Packages) and OCI artifacts, including Helm charts stored as OCI.
2. **[02-README-Package-Managers-Semver.md](02-README-Package-Managers-Semver.md)** — Nexus/Artifactory for private package registries, and semantic versioning in CI.
3. **[03-README-Provenance-SLSA.md](03-README-Provenance-SLSA.md)** — artifact signing/provenance and the SLSA framework's maturity levels.

## Container registries — the options and their tradeoffs

| Registry | Notes |
|---|---|
| **Docker Hub** | The original, still the default unauthenticated pull target for most public base images (`node`, `alpine`, `postgres`). Free tier has aggressive pull-rate limits for anonymous/unauthenticated pulls — a very common cause of mysterious CI failures ("pull rate limit exceeded") when many CI runners share an egress IP. |
| **Amazon ECR** | AWS-native, tightly integrated with IAM (pull/push permissions via IAM policy, not a separate registry-level auth system), native vulnerability scanning (Day 67), and pairs naturally with ECS/EKS. Regional by default (`ECR Private`), with a separate `ECR Public` for public-facing images. |
| **Google GCR / Artifact Registry** | GCR is being phased out in favor of **Artifact Registry**, which (notably) supports more than containers — Maven, npm, Python, and generic artifacts in one product, foreshadowing the "one registry for everything" trend. |
| **GitHub Container Registry (GHCR)** / **GitHub Packages** | Tightly coupled to GitHub repo permissions — an image's visibility and access naturally follows the repo's. Convenient when your source and CI are already on GitHub, avoiding a separate registry-auth setup entirely. |

**The interview-relevant point:** the *choice* usually isn't about raw features (they mostly overlap) — it's about **where your identity/auth boundary already lives**. If you're all-in on AWS IAM, ECR's IAM-native model avoids a second identity system. If you're all-in on GitHub, GHCR avoids the same duplication. Fighting your existing identity model to use a "better" registry is rarely worth the operational friction.

## OCI artifacts — registries as generic artifact storage

The **OCI (Open Container Initiative) distribution spec** — the API contract that "push"/"pull" implement — turns out to be a genuinely generic content-addressable blob store with manifests, not something inherently tied to container images. This realization led to the **OCI Artifacts** convention: you can push *anything* (a Helm chart, a WASM binary, an SBOM, a Cosign signature — which is itself stored as an OCI artifact, tying back to Day 67) to a container registry, tagged and versioned exactly like an image, using the same push/pull/auth machinery you already operate for images.

This matters practically because it means **one registry, one auth model, one set of operational runbooks** for every artifact type your pipelines produce — instead of a container registry for images, a separate Helm chart museum for charts, and yet another system for anything else.

### Helm charts as OCI artifacts

Helm historically used its own chart repository format (a static `index.yaml` served over HTTP, `helm repo add`). Since Helm 3.8+, OCI-based chart storage is natively supported and is now the recommended approach:

```bash
# Package a chart and push it to an OCI-compliant registry (ECR here)
helm package ./mychart --version 1.4.0
aws ecr get-login-password | helm registry login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

helm push mychart-1.4.0.tgz oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts

# Pull/install directly from the OCI registry — no `helm repo add` needed at all
helm install myrelease oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts/mychart --version 1.4.0
```

Notice there's no separate `helm repo add` step for OCI-based charts — you reference the `oci://` URL directly, because the registry itself (not a separate chart-index file) is the source of truth for what versions exist.

### ORAS — the generic OCI client

**ORAS** (OCI Registry As Storage) is the general-purpose CLI for pushing/pulling arbitrary files as OCI artifacts, for anything that doesn't have its own OCI-aware tooling (like Helm now does natively):

```bash
# Push an arbitrary file (e.g., a Terraform module tarball, a config bundle) as an OCI artifact
oras push 123456789012.dkr.ecr.us-east-1.amazonaws.com/configs/app-config:v1.2.0 \
  ./config-bundle.tar.gz:application/vnd.oci.image.layer.v1.tar+gzip

# Pull it back down
oras pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/configs/app-config:v1.2.0

# List the tags for an artifact
oras repo tags 123456789012.dkr.ecr.us-east-1.amazonaws.com/configs/app-config
```

Under the hood, `oras push` constructs a valid OCI manifest with your file(s) as one or more layers and an appropriate media type — the registry stores it exactly like it would a container image layer, because as far as the registry's storage/API is concerned, it *is* one.

## Points to Remember

- Registry choice is usually driven by which identity/auth system you're already standardized on (AWS IAM → ECR, GitHub → GHCR) more than by feature differences between registries, which mostly overlap.
- Docker Hub's anonymous pull-rate limiting is a very real, common CI failure mode — mitigate by authenticating pulls (even a free account raises the limit significantly) or mirroring frequently-used base images into your own registry.
- OCI Artifacts is the realization that the container registry push/pull/manifest API is a generic, content-addressable artifact store — not something inherently specific to container images.
- Helm 3.8+ supports OCI registries natively (`oci://` references, `helm push`/`helm install oci://...`) — no separate `helm repo add`/`index.yaml` chart-repo machinery needed anymore.
- **ORAS** is the generic tool for pushing/pulling *anything* as an OCI artifact when the tool you're using (unlike Helm) has no built-in OCI awareness of its own.

## Common Mistakes

- Hitting Docker Hub's anonymous pull-rate limit in CI and misdiagnosing it as a network/registry outage, rather than recognizing the actual cause (shared CI runner egress IPs exhausting the free unauthenticated pull quota) and fixing it by authenticating pulls or mirroring the base image internally.
- Standing up a completely separate system for Helm chart distribution (a classic chart museum) in an org that already operates a container registry with OCI support — duplicating auth/access-control work that OCI-based chart storage would have avoided entirely.
- Assuming ECR/GCR/GHCR are functionally interchangeable and picking based on marketing rather than which identity/access-control system already governs the rest of the org's infrastructure — leading to a second, parallel auth system nobody wanted to maintain.
- Forgetting that OCI artifacts still need proper tagging/versioning discipline (see Day 70 file 2 on semver) — pushing arbitrary blobs to a registry doesn't automatically give you good version hygiene; that's still on you to enforce.
- Not realizing Cosign signatures (Day 67) and SBOM attestations are themselves stored as OCI artifacts in the same registry as the image — expecting them to live in some separate signature-specific service and being confused when `cosign verify` is just doing another registry pull under the hood.
