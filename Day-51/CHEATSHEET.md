# Day 51 — Cheatsheet: K8s Upgrades & Cluster Management

## EKS upgrade order and commands

```bash
# 1. Control plane FIRST — one minor version at a time, cannot skip
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.29
aws eks describe-update --name my-cluster --update-id <ID>

# 2. Add-ons — check/update compatibility with new control plane
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.29
aws eks update-addon --cluster-name my-cluster --addon-name vpc-cni --addon-version <VERSION>

# 3. Node groups — only after control plane is upgraded
aws eks update-nodegroup-version --cluster-name my-cluster --nodegroup-name my-nodegroup
```

Version skew rule: kubelet version <= control plane version, never newer. Control plane upgrades: one minor version at a time.

## Node upgrade strategies

| Strategy | Risk | Rollback |
|---|---|---|
| Rolling in-place node group | modifies live infra | must roll forward/retry |
| Blue/green node group | old group untouched during cutover | scale new group to 0, keep old |
| Blue/green full cluster | most conservative, 2x infra cost | shift traffic back to old cluster |

## Cordon / Drain / PDB

```bash
kubectl cordon <node>                                            # stop new scheduling only
kubectl uncordon <node>                                           # reverse cordon
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=300s   # cordon + safe evict
kubectl get nodes                                                 # SchedulingDisabled = cordoned
```

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: my-app-pdb }
spec:
  minAvailable: 2       # OR maxUnavailable: 1 (mutually exclusive)
  selector:
    matchLabels: { app: my-app }
```

PDB protects only **voluntary** disruptions (drain, autoscaler scale-down) via the Eviction API — not node failure/involuntary kills.
`kubectl delete pod` bypasses PDBs entirely; `kubectl drain` does not.

## pluto — deprecated/removed API scanning

```bash
pluto detect-all-in-cluster --target-versions k8s=v1.29.0     # scan live cluster
pluto detect-helm -o wide                                      # scan Helm releases
pluto detect-files -d ./manifests --target-versions k8s=v1.29.0  # scan a directory
```
Output: `DEPRECATED` (still works, will be removed) vs `REMOVED` (already broken on target version).

## kubectl-convert

```bash
kubectl krew install convert                                     # install plugin
kubectl convert -f old.yaml --output-version apps/v1 -o new.yaml
diff old.yaml new.yaml   # always review — semantics can change, not just apiVersion string
```

## Velero

```bash
velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket my-velero-backups --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 --secret-file ./credentials-velero

velero backup create pre-upgrade-backup
velero backup create ns-backup --include-namespaces production --selector app=my-app
velero schedule create daily-backup --schedule="0 2 * * *" --ttl 720h0m0s
velero backup get
velero backup describe pre-upgrade-backup --details
velero restore create --from-backup pre-upgrade-backup
velero restore get
```

## Common removed API versions (know these cold)

| Old | Removed in | Current |
|---|---|---|
| `extensions/v1beta1` (Deployment) | 1.16 | `apps/v1` |
| `apps/v1beta1`/`v1beta2` (Deployment) | 1.16 | `apps/v1` |
| `policy/v1beta1` (PodDisruptionBudget) | 1.25 | `policy/v1` |
| `batch/v1beta1` (CronJob) | 1.25 | `batch/v1` |
| `networking.k8s.io/v1beta1` (Ingress) | 1.22 | `networking.k8s.io/v1` |
