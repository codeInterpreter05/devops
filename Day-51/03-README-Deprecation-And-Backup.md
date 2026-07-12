# Day 51 — K8s Upgrades & Cluster Management: API Deprecations & Backup

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Kubernetes | **Flag:** —

## Brief

The two things most likely to turn a routine Kubernetes upgrade into an incident are: manifests using an API version that got **removed** (not just deprecated — actually removed) in the target version, and having no reliable way to **roll back** cluster state if the upgrade goes wrong. `pluto` and `kubectl-convert` address the first problem; Velero addresses the second. Together they're what makes "we upgraded and something broke" recoverable instead of catastrophic.

## Kubernetes API deprecation policy — why this is a recurring problem

Kubernetes follows a formal **API deprecation policy**: a stable (GA, `v1`) API, once deprecated, must remain supported for at least **12 months or 3 releases (whichever is longer)** before removal. Beta APIs get shorter windows, alpha APIs can be removed at any time with no guarantee. The catch: this policy guarantees *some* notice, but it doesn't guarantee anyone reads the release notes — manifests referencing a since-removed `apiVersion` (classic examples: `extensions/v1beta1` and `apps/v1beta1` for Deployments, both removed in 1.16; `policy/v1beta1` for PodDisruptionBudget, removed in 1.25; `batch/v1beta1` for CronJob, removed in 1.25) will simply **fail to apply** against a cluster that has moved past the removal version — `kubectl apply` returns a "no matches for kind" error, and anything relying on GitOps auto-sync from an old manifest silently stops reconciling.

This is precisely why checking for deprecated/removed APIs is a **mandatory pre-upgrade step**, not an optional nice-to-have — it's the single most common cause of "the cluster upgraded fine, but half our Helm charts are now broken."

## `pluto` — finding deprecated/removed APIs before you upgrade

`pluto` scans your live cluster (or your Helm charts / raw manifests / directories in a repo) for API versions that are deprecated or already removed **relative to a target Kubernetes version**, and reports exactly which resources need fixing before you upgrade.

```bash
# Scan a live cluster for deprecated/removed APIs, targeting a specific future version
pluto detect-all-in-cluster --target-versions k8s=v1.29.0

# Scan Helm releases specifically (catches charts installed via Helm, not just raw manifests)
pluto detect-helm -o wide

# Scan a directory of raw YAML manifests (e.g., your GitOps repo) before merging an upgrade PR
pluto detect-files -d ./k8s-manifests --target-versions k8s=v1.29.0
```
Output flags each finding as `DEPRECATED` (still works, but scheduled for removal) or `REMOVED` (will already fail against the target version) — this distinction is what tells you whether something is a "fix soon" or a "fix before you upgrade or this breaks immediately."

## `kubectl-convert` — migrating manifests to the new API version

Once `pluto` tells you *what* needs fixing, `kubectl convert` (a `kubectl` plugin, install via `kubectl krew install convert` or download from the `kubernetes/kubectl` releases) rewrites a manifest from an old `apiVersion` to a current one automatically:

```bash
kubectl convert -f old-deployment.yaml --output-version apps/v1 -o converted-deployment.yaml
```
It handles the mechanical field renames/restructuring between API versions (which isn't always a trivial find-and-replace — some fields moved location or changed structure between versions, not just the `apiVersion` string itself). You should still **review the diff** rather than blindly trusting the conversion, since some semantic defaults changed between versions too (a known example: `apps/v1` Deployments require `spec.selector` to be explicitly set and immutable, whereas the older `extensions/v1beta1` would default/infer it — a converted manifest needs that filled in correctly).

## Velero — cluster backup and disaster recovery

Velero backs up **Kubernetes API objects** (as JSON, stored in an object store like S3) and, optionally, the **contents of persistent volumes** (via provider-specific snapshot APIs or its own file-system backup mode, Restic/Kopia-based), giving you a way to restore cluster state after a bad upgrade, an accidental `kubectl delete namespace`, or a full disaster-recovery scenario (region loss, cluster deletion).

```bash
# Install Velero into the cluster (once), pointing at an S3 bucket for backup storage
velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket my-velero-backups --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 --secret-file ./credentials-velero

# Take an on-demand backup of everything
velero backup create pre-upgrade-backup

# Or scope it to a namespace / label selector
velero backup create pre-upgrade-backup --include-namespaces production --selector app=my-app

# Schedule recurring backups (cron syntax)
velero schedule create daily-backup --schedule="0 2 * * *" --ttl 720h0m0s

# List / inspect backups
velero backup get
velero backup describe pre-upgrade-backup --details

# Restore from a backup (e.g., after a failed upgrade or accidental deletion)
velero restore create --from-backup pre-upgrade-backup
```

**Why "before every cluster upgrade" is the standard operating rule**: a control plane upgrade is largely non-destructive to existing objects (etcd data isn't touched by a normal minor-version bump), but a bad upgrade combined with a bad manifest migration, a botched node group cutover, or an admission webhook that starts rejecting objects post-upgrade can still leave you needing to restore specific namespaces/objects to a known-good state quickly. Velero backups (plus EKS's own automatic etcd management, which you don't control directly) are the practical, actionable safety net an operator can pull the trigger on themselves.

**What Velero is not**: it is not a continuous replication/high-availability solution, and restoring persistent volume data depends on your storage provider's snapshot support being configured correctly — test restores periodically (in a scratch cluster/namespace) rather than assuming a backup you've never restored from will work when you actually need it.

## Points to Remember

- Kubernetes guarantees a minimum deprecation window (12 months / 3 releases for GA APIs) before removal, but that only means "eventually breaks," not "safe to ignore" — checking before every upgrade is mandatory practice, not optional.
- `pluto` scans live clusters, Helm releases, or raw manifest directories and reports `DEPRECATED` vs. `REMOVED` relative to a target version — run it against your *next* target version before starting the upgrade.
- `kubectl convert` mechanically rewrites manifests to a new `apiVersion`, but always review the diff — some fields changed semantics/requirements between API versions, not just names.
- Velero backs up Kubernetes objects (always) and PV data (if configured with snapshot/Restic support) to an external object store — it's the practical rollback mechanism for a bad upgrade, not a replacement for etcd's own durability.
- Test Velero restores periodically in a non-production scratch environment — an untested backup is not a verified backup.

## Common Mistakes

- Upgrading the control plane without first scanning for removed APIs, then discovering GitOps sync is silently broken because manifests reference an `apiVersion` that no longer exists on the new control plane.
- Treating `kubectl convert`'s output as final without reviewing it — missing a newly-required field (like an explicit, immutable `spec.selector` on Deployments) that the old API version used to infer automatically.
- Running `pluto` once, long before the actual upgrade, and not re-running it right before cutover — new manifests can be added to the repo in the interim that reintroduce a deprecated API version.
- Installing Velero but never actually running a test restore — discovering during a real incident that the PV snapshot plugin was misconfigured, or that IAM permissions for the S3 bucket were insufficient, at the worst possible time.
- Assuming a Velero backup captures everything needed to fully recreate a cluster from scratch (it doesn't back up cluster-level infrastructure like the EKS control plane itself, IAM roles, VPC config) — Velero restores workloads/objects into an *existing* cluster, it isn't a full infrastructure-as-code replacement.
