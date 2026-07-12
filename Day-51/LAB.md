# Day 51 — Lab: K8s Upgrades & Cluster Management

**Goal:** Practice a real minor-version cluster upgrade end-to-end using minikube, check for deprecated APIs beforehand, and drill cordon/drain/PDB behavior safely — the assigned hands-on activity, broken into concrete steps.

**Prerequisites:** `minikube` installed, `kubectl`, `helm`. Install `pluto`:
```bash
brew install FairwindsOps/tap/pluto   # macOS
# or download a release binary from github.com/FairwindsOps/pluto/releases
```
Install Velero CLI (for the stretch section):
```bash
brew install velero
```

---

### Lab 1 — Check for deprecated/removed APIs before upgrading (core hands-on activity, part 1)

1. Start a cluster on an older minor version:
   ```bash
   minikube start --kubernetes-version=v1.28.0 -p upgrade-lab
   ```
2. Deploy something using a slightly old but still valid API, plus check the whole cluster:
   ```bash
   kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
   pluto detect-all-in-cluster --target-versions k8s=v1.29.0
   ```
3. Also scan a local directory of manifests (create one with an intentionally old PDB API version to see `pluto` catch it):
   ```bash
   mkdir -p /tmp/pluto-test
   cat <<'EOF' > /tmp/pluto-test/old-pdb.yaml
   apiVersion: policy/v1beta1
   kind: PodDisruptionBudget
   metadata:
     name: old-pdb
   spec:
     minAvailable: 1
     selector:
       matchLabels: { app: nginx }
   EOF
   pluto detect-files -d /tmp/pluto-test --target-versions k8s=v1.29.0
   ```
4. Convert the old manifest to a current API version and diff it:
   ```bash
   kubectl convert -f /tmp/pluto-test/old-pdb.yaml --output-version policy/v1 -o /tmp/pluto-test/new-pdb.yaml
   diff /tmp/pluto-test/old-pdb.yaml /tmp/pluto-test/new-pdb.yaml
   ```

**Success criteria:** `pluto` correctly flags the old PDB manifest as `REMOVED` relative to v1.29, and you can explain what would happen if you tried to `kubectl apply` the unconverted version against a 1.29+ cluster.

---

### Lab 2 — Upgrade minikube from 1.28 to 1.29 (core hands-on activity, part 2)

1. Confirm current version and upgrade:
   ```bash
   kubectl version --short
   minikube start --kubernetes-version=v1.29.0 -p upgrade-lab
   ```
   (minikube handles this as a control-plane-and-node upgrade together since it's a single-node dev cluster — note in your own words how this differs from EKS, where control plane and node groups are upgraded as two separate, ordered steps.)
2. Confirm the upgrade succeeded and the earlier Deployment survived:
   ```bash
   kubectl version --short
   kubectl get deployment nginx-deployment
   ```
3. Re-run `pluto detect-all-in-cluster` against the now-current cluster and confirm no unexpected regressions.

**Success criteria:** You can articulate why a real EKS upgrade requires the strict "control plane, then nodes" ordering even though minikube's single-node model hides that distinction.

---

### Lab 3 — Cordon, drain, and PodDisruptionBudget interaction

1. Deploy a 3-replica app with a tight PDB:
   ```bash
   kubectl create deployment web --image=nginx --replicas=3
   cat <<'EOF' | kubectl apply -f -
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: web-pdb
   spec:
     minAvailable: 3
     selector:
       matchLabels: { app: web }
   EOF
   ```
2. Try to drain the (only, in minikube) node and observe it hang/fail:
   ```bash
   kubectl drain upgrade-lab --ignore-daemonsets --delete-emptydir-data --timeout=30s
   ```
   Note the PDB-related error message.
3. Fix it by loosening the PDB, then retry:
   ```bash
   kubectl patch pdb web-pdb -p '{"spec":{"minAvailable":2}}'
   kubectl drain upgrade-lab --ignore-daemonsets --delete-emptydir-data --timeout=60s
   kubectl uncordon upgrade-lab
   ```

**Success criteria:** You can explain, from having watched it happen, why `minAvailable` equal to total replica count makes a node undrainable, and why that's the PDB working as designed rather than a bug.

---

### Cleanup

```bash
minikube delete -p upgrade-lab
rm -rf /tmp/pluto-test
```

### Stretch challenge

Install Velero against a local MinIO or real S3 bucket, take a backup of the `web` Deployment and its PDB before deleting the namespace, then restore from that backup and confirm both objects come back exactly as they were — including the (patched) `minAvailable: 2` value.
