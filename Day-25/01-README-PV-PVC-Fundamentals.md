# Day 25 — Storage in K8s: PersistentVolumes & PersistentVolumeClaims

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Containers are ephemeral by design — anything written to a container's own filesystem disappears the moment it's recreated. For stateless apps that's fine; for anything that needs real persistence (a database, an uploaded-file store, a message queue's data directory) Kubernetes needs a storage abstraction that survives pod restarts and even pod deletion. That abstraction is the `PersistentVolume`/`PersistentVolumeClaim` pair, and "what happens to data in a PVC when you delete a Pod, versus when you delete the PVC itself" is a direct, frequently asked interview question that tests whether you actually understand the lifecycle, not just the acronyms.

This day is split into three files:

1. **This file** — PV/PVC binding lifecycle and reclaim policy.
2. **[02-README-StorageClasses-CSI.md](02-README-StorageClasses-CSI.md)** — StorageClasses, dynamic provisioning, access modes, and CSI drivers (EBS/EFS).
3. **[03-README-Ephemeral-Storage-Snapshots.md](03-README-Ephemeral-Storage-Snapshots.md)** — `emptyDir` vs `hostPath`, and VolumeSnapshots.

## The two-object model: PV and PVC

- **`PersistentVolume` (PV)** — a cluster-level resource representing an actual piece of storage (an EBS volume, an NFS export, a local disk). It exists independently of any Pod or namespace — think of it as the storage admin's inventory of "here is a disk that exists and is available."
- **`PersistentVolumeClaim` (PVC)** — a namespaced *request* for storage, made by a user/application: "I need 10Gi, ReadWriteOnce, at least this fast." A Pod never references a PV directly — it references a PVC, and the PVC is what gets **bound** to a matching PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3
```

```yaml
apiVersion: v1
kind: Pod
metadata: { name: app }
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: data
          mountPath: /var/lib/app-data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

This indirection (Pod → PVC → PV) is deliberate: it decouples *what an application needs* from *which specific piece of physical/cloud storage satisfies it* — the same Pod spec works whether the PVC is ultimately backed by an EBS volume, a local disk, or NFS, because the Pod only ever knows about the PVC.

## Binding lifecycle

1. A PVC is created, describing size/access-mode/`storageClassName` requirements.
2. The PV controller looks for an existing, unbound PV that satisfies those requirements exactly (static provisioning) — **or**, if `storageClassName` names a class with dynamic provisioning enabled (the overwhelmingly common case today, covered in file 2), a brand-new PV is created on demand to satisfy the claim.
3. Once matched, the PVC and PV are **bound 1:1** — a PV can never be shared across multiple PVCs, even if it has spare capacity.
4. The Pod's `volumeMounts` reference the PVC by name; the kubelet (via the CSI driver) handles actually attaching/mounting the underlying volume onto the node and into the container's filesystem namespace.

```bash
kubectl get pv                       # cluster-scoped, shows Status: Available/Bound/Released
kubectl get pvc                      # namespaced, shows Status: Pending/Bound
kubectl describe pvc data-pvc          # shows the exact PV it's bound to, and any binding errors
```

A PVC stuck in `Pending` almost always means: no PV/StorageClass satisfies the requested size/access-mode combination, or (for dynamic provisioning) the provisioner itself is failing — `kubectl describe pvc` always shows the specific event/error.

## Reclaim policy: what happens to the data

Every PV has a `persistentVolumeReclaimPolicy`, which governs what happens **once its bound PVC is deleted** (not when the Pod using it is deleted — Pod deletion never touches the PVC at all, by design):

- **`Retain`** (safest, common for anything you care about): the PV is **not** deleted or reused automatically — it moves to `Released` status, keeping the underlying data intact, and requires manual admin intervention (deleting or repurposing the PV) before that storage can be reused. This is the deliberate "don't ever silently destroy someone's data" default for statically provisioned volumes.
- **`Delete`** (the default for most dynamically-provisioned StorageClasses, e.g. AWS EBS `gp3`): when the PVC is deleted, the underlying cloud storage resource (the actual EBS volume) is also deleted automatically. Convenient for disposable/dev environments, dangerous if assumed for production data without backups.
- **`Recycle`** (deprecated, don't use): used to scrub the volume (`rm -rf` its contents) and make it `Available` again for reuse; removed in favor of dynamic provisioning entirely.

```bash
kubectl get pv <name> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
kubectl patch pv <name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'   # change policy on an existing PV
```

## The direct answer to "what happens to data..."

- **Delete the Pod** → nothing happens to the PVC or its data at all. The PVC remains `Bound`, the underlying storage is untouched, and a new Pod referencing the same PVC will see exactly the same data (this is the entire point — it's how a StatefulSet pod comes back with its data intact, see Day 23).
- **Delete the PVC** → depends entirely on `persistentVolumeReclaimPolicy` on the bound PV: `Retain` keeps the data (PV becomes `Released`, needs manual cleanup); `Delete` destroys the underlying storage resource along with the data, generally irreversibly (unless you have snapshots — see file 3).

## Points to Remember

- Pods never reference PVs directly — they reference a PVC, and the PVC is what's bound 1:1 to a PV. This indirection is what makes Pod specs portable across different underlying storage backends.
- Deleting a Pod never touches its PVC or data — persistence across pod restarts/rescheduling is the entire reason PVCs exist.
- `persistentVolumeReclaimPolicy` (on the PV, not the PVC) is what determines whether deleting the *PVC* destroys the underlying data (`Delete`) or preserves it for manual recovery (`Retain`).
- A PVC stuck `Pending` is a matching/provisioning problem — check `kubectl describe pvc` for the exact reason before assuming it's a deeper cluster issue.
- `Recycle` is deprecated; don't use it or expect to see it in modern documentation/exam material as a real option.

## Common Mistakes

- Assuming deleting a Pod deletes its data — a common and understandable confusion, but PVCs are explicitly designed to outlive the Pods that use them.
- Not checking a PV's reclaim policy before deleting a PVC in production — with `Delete` policy (the common dynamic-provisioning default), this is an irreversible data-loss action with no confirmation prompt.
- Assuming a `Retain`-policy PV automatically becomes reusable after its PVC is deleted — it moves to `Released`, not `Available`; it requires manual intervention (typically deleting and recreating the PV object, or clearing `claimRef`) before anything can bind to it again.
- Forgetting that a PV can only ever be bound to one PVC at a time, even if the PV has far more capacity than the PVC requests — there's no capacity-sharing between multiple claims on a single PV.
- Treating "my Pod's data survived a restart" as proof backups aren't needed — PVC persistence protects against pod/node churn, not against accidental `kubectl delete pvc`, storage-backend failure, or application-level data corruption; none of those are covered by the PV/PVC model alone.
