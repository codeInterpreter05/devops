# Day 28 — Lab: Helm & Kustomize

**Goal:** Package a real application as a Helm chart with per-environment values files (the assigned hands-on activity), practice hooks and OCI push/pull, then build an equivalent Kustomize base+overlay setup and compare the two approaches directly.

**Prerequisites:** minikube running, `helm`, `kustomize` (or `kubectl` v1.14+, which bundles it), and `helm-docs` (optional, for the stretch challenge).

```bash
minikube start --driver=docker
helm version
kustomize version || kubectl kustomize --help
```

---

### Lab 1 — The core hands-on activity: package your app as a Helm chart with dev/staging/prod values

This is the assigned hands-on activity — do it for real, using a simple nginx-based stand-in if you don't have your internship app handy locally.

1. Scaffold a chart:
   ```bash
   helm create myapp
   cd myapp
   rm -rf templates/tests   # trim the default scaffold's extras for this lab
   ```
2. Simplify `templates/deployment.yaml` to something you fully understand line-by-line (replace the generated one with a minimal version referencing `.Values.replicaCount`, `.Values.image.repository`, `.Values.image.tag`, and `.Values.resources` — reuse the pattern from file 1 of this day's notes).
3. Create three environment values files:
   ```bash
   cat <<'EOF' > values-dev.yaml
   replicaCount: 1
   image: { repository: nginx, tag: "1.25" }
   resources: { requests: { cpu: 50m, memory: 64Mi }, limits: { memory: 128Mi } }
   EOF
   cat <<'EOF' > values-staging.yaml
   replicaCount: 2
   image: { repository: nginx, tag: "1.25" }
   resources: { requests: { cpu: 100m, memory: 128Mi }, limits: { memory: 256Mi } }
   EOF
   cat <<'EOF' > values-prod.yaml
   replicaCount: 4
   image: { repository: nginx, tag: "1.25" }
   resources: { requests: { cpu: 250m, memory: 256Mi }, limits: { memory: 512Mi } }
   EOF
   ```
4. Lint and render (never skip this step before installing):
   ```bash
   helm lint .
   helm template myapp-dev . -f values-dev.yaml
   helm template myapp-prod . -f values-prod.yaml | grep -A2 replicas
   ```
5. Install into three separate namespaces, one per environment:
   ```bash
   for env in dev staging prod; do
     kubectl create namespace $env --dry-run=client -o yaml | kubectl apply -f -
     helm install myapp . -f values-$env.yaml -n $env --atomic --timeout 3m
   done
   kubectl get deployments -A -l app.kubernetes.io/instance=myapp
   ```
6. Confirm the environments actually differ:
   ```bash
   for env in dev staging prod; do
     echo "== $env =="; kubectl get deployment myapp -n $env -o jsonpath='{.spec.replicas}{"\n"}'
   done
   ```

**Success criteria:** Three running releases (`dev`/`staging`/`prod`) from the **same chart**, each with a different replica count and resource allocation, confirmed via `kubectl get deployment -o jsonpath`.

---

### Lab 2 — Upgrade, rollback, and a `pre-upgrade` hook

1. Add a simple `pre-upgrade` hook Job (echoing a "migration" message) to `templates/migrate-job.yaml`, following file 1's hook annotation pattern.
2. Bump the image tag in `values-prod.yaml` to `1.26` and upgrade:
   ```bash
   helm upgrade myapp . -f values-prod.yaml -n prod --atomic --timeout 3m
   kubectl get jobs -n prod    # confirm the hook Job ran
   helm history myapp -n prod
   ```
3. Roll back and confirm the image tag reverts:
   ```bash
   helm rollback myapp 1 -n prod
   kubectl get deployment myapp -n prod -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
   ```

**Success criteria:** You've watched a hook Job execute automatically during `helm upgrade`, and confirmed `helm rollback` reverts both the image tag and (if you'd changed it) the replica count to the exact previous revision's values.

---

### Lab 3 — OCI push/pull (local registry, no cloud account needed)

1. Run a local OCI registry for practice:
   ```bash
   docker run -d -p 5000:5000 --name local-registry registry:2
   ```
2. Package and push your chart:
   ```bash
   helm package .
   helm push myapp-0.1.0.tgz oci://localhost:5000/helm-charts
   ```
3. Pull and install directly from the OCI registry:
   ```bash
   helm pull oci://localhost:5000/helm-charts/myapp --version 0.1.0
   helm install myapp-from-oci oci://localhost:5000/helm-charts/myapp --version 0.1.0 -n dev --dry-run --debug
   ```

**Success criteria:** You've pushed a chart to an OCI registry and pulled/dry-run-installed it back, confirming the full `oci://` workflow end-to-end without any special chart-repository infrastructure.

---

### Lab 4 — Equivalent setup in Kustomize, for comparison

1. Build the same three-environment structure using Kustomize instead:
   ```bash
   mkdir -p kustomize-myapp/base kustomize-myapp/overlays/{dev,staging,prod}
   ```
   Write `base/deployment.yaml`, `base/kustomization.yaml`, and per-overlay `kustomization.yaml` + patch files following file 3's pattern, targeting `replicas` and `resources` per environment exactly like the Helm values files did.
2. Render and diff against the Helm-rendered output:
   ```bash
   kubectl kustomize kustomize-myapp/overlays/prod > /tmp/kustomize-prod.yaml
   helm template myapp-prod ./myapp -f myapp/values-prod.yaml > /tmp/helm-prod.yaml
   diff /tmp/kustomize-prod.yaml /tmp/helm-prod.yaml   # expect differences in labels/structure, similar core config
   ```

**Success criteria:** You have a working Kustomize overlay set producing equivalent per-environment Deployments to the Helm chart, and can articulate concretely (from direct experience, not just reading file 3) which approach felt simpler for this specific task.

---

### Cleanup

```bash
for env in dev staging prod; do helm uninstall myapp -n $env; kubectl delete namespace $env; done
helm uninstall myapp-from-oci -n dev 2>/dev/null
docker rm -f local-registry
```

### Stretch challenge

Run `helm-docs` against your chart (add `# --` comments to `values.yaml` first) and confirm it generates an accurate `README.md` values table automatically. Then intentionally add a new value to `values.yaml` without a comment and observe how `helm-docs` handles the gap — a good check on whether you're relying on it correctly as living documentation rather than a one-time generator.
