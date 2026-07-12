# Day 70 — Lab: Artifact Management and Registries

**Goal:** Push a Helm chart to an OCI registry, version it with a git tag, and pull it into ArgoCD — the day's assigned hands-on activity — plus hands-on practice with a private package proxy and SLSA-style provenance.

**Prerequisites:**
- Docker installed, and either a real AWS account with ECR access, or a local registry (`registry:2` image) for a lower-cost stand-in.
- `helm` (3.8+) installed.
- A local Kubernetes cluster (`kind`/`minikube`) with ArgoCD installed for the core lab (`kubectl create namespace argocd && kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`).
- `oras` CLI installed for Lab 2.

---

### Lab 1 — Registry basics: run a local proxy-style registry

1. Run a local registry as a stand-in for Nexus/Artifactory's proxy role:
   ```bash
   docker run -d -p 5000:5000 --name local-registry registry:2
   ```
2. Pull a public image, retag it to point at your local registry, and push:
   ```bash
   docker pull alpine:3.19
   docker tag alpine:3.19 localhost:5000/alpine:3.19
   docker push localhost:5000/alpine:3.19
   ```
3. Delete the local copy and re-pull it from your "cache" instead of Docker Hub directly:
   ```bash
   docker rmi alpine:3.19 localhost:5000/alpine:3.19
   docker pull localhost:5000/alpine:3.19
   ```

**Success criteria:** You can explain, using this exercise, why a shared internal registry cache reduces both public-registry outage risk and Docker Hub rate-limit exposure for an entire org's CI fleet.

---

### Lab 2 — Push an arbitrary artifact with ORAS

1. Create a dummy config bundle:
   ```bash
   mkdir -p /tmp/day70 && cd /tmp/day70
   echo '{"env":"staging","replicas":3}' > config.json
   tar czf config-bundle.tar.gz config.json
   ```
2. Push it as an OCI artifact to your local registry:
   ```bash
   oras push localhost:5000/configs/app-config:v1.0.0 \
     config-bundle.tar.gz:application/vnd.oci.image.layer.v1.tar+gzip --plain-http
   ```
3. Pull it back down into a clean directory and verify contents match:
   ```bash
   mkdir pulled && cd pulled
   oras pull localhost:5000/configs/app-config:v1.0.0 --plain-http
   ```

**Success criteria:** You've pushed and pulled a non-container-image file through the same registry API used for container images, and can explain why this works (OCI distribution spec is a generic manifest+blob store).

---

### Lab 3 — The core hands-on activity: Helm chart as an OCI artifact, versioned with git, pulled by ArgoCD

1. Scaffold a minimal Helm chart:
   ```bash
   helm create day70-chart
   cd day70-chart
   ```
2. Tag it with a real git version and set the chart version to match:
   ```bash
   git init && git add . && git commit -m "initial chart"
   git tag v1.0.0
   sed -i.bak "s/^version:.*/version: 1.0.0/" Chart.yaml
   ```
3. Package and push to your local (or ECR) OCI registry:
   ```bash
   helm package . --version 1.0.0
   helm push day70-chart-1.0.0.tgz oci://localhost:5000/helm-charts --plain-http
   ```
4. Point ArgoCD at the OCI chart. Create an `Application` manifest:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: day70-app
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: localhost:5000/helm-charts/day70-chart
       chart: day70-chart
       targetRevision: "1.0.0"
       helm: {}
     destination:
       server: https://kubernetes.default.svc
       namespace: default
     syncPolicy:
       automated: {}
   ```
   ```bash
   kubectl apply -f day70-app.yaml
   ```
5. Bump the chart version, re-tag in git (`v1.1.0`), re-package, re-push, and update `targetRevision` — confirm ArgoCD picks up the new version.

**Success criteria:** ArgoCD successfully syncs an application whose Helm chart source is an OCI registry reference, and you can demonstrate a version bump flowing through git tag → chart version → registry push → ArgoCD sync.

---

### Cleanup

```bash
docker rm -f local-registry
kind delete cluster --name <your-cluster-name>   # if you created one just for this lab
rm -rf /tmp/day70
```

### Stretch challenge

Add a `cosign attest` step generating an SLSA-style provenance predicate for the Helm chart's OCI artifact (treat the chart tarball's digest the same as an image digest), then write the `cosign verify-attestation` command you'd run to confirm it before letting ArgoCD sync it — sketch, in a comment, how you'd wire that verification into ArgoCD's sync process as a pre-sync check.
