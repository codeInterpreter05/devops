# Day 25 — Storage in K8s: Ephemeral Storage & Volume Snapshots

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Not every volume needs to be a PVC — sometimes you need scratch space that dies with the Pod, or a deliberate (and usually discouraged) escape hatch into the host node's own filesystem. And once you *do* have real persistent data, you need a backup story — `VolumeSnapshot` is how Kubernetes exposes point-in-time backup/restore for CSI-backed volumes natively, instead of relying entirely on external tooling.

## `emptyDir` — scratch space that lives and dies with the Pod

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - { name: scratch, mountPath: /cache }
  volumes:
    - name: scratch
      emptyDir:
        sizeLimit: 1Gi          # optional cap; without it, can consume unbounded node disk
```

An `emptyDir` volume is created fresh (empty, as the name says) when the Pod is scheduled to a node, and is **permanently deleted when the Pod is removed from that node** — not just restarted, but removed. It's the standard choice for:
- **Scratch/cache space** a container needs temporarily (a build cache, a temp-file directory for processing).
- **Sharing files between containers in the same Pod** — every container in a Pod can mount the same `emptyDir` and read/write each other's files, which is the standard sidecar pattern (e.g., a log-shipping sidecar reading files a main container writes).

By default, `emptyDir` is backed by the node's normal disk (or whatever storage is provisioned for kubelet's ephemeral storage). Setting `medium: Memory` backs it with `tmpfs` (RAM) instead — extremely fast, but counts against the Pod's memory limit and vanishes entirely on any restart (even faster to lose than the disk-backed default, since it never touches disk at all).

## `hostPath` — mounting the node's own filesystem into a Pod

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - { name: docker-sock, mountPath: /var/run/docker.sock }
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
        type: Socket
```

`hostPath` mounts a path from the **underlying node's** filesystem directly into the Pod — meaning the data is tied to that *specific node*, not to the Pod's identity, and definitely not portable if the Pod gets rescheduled elsewhere. Legitimate uses are narrow and almost always infrastructure-level: a monitoring/logging DaemonSet reading `/var/log` on every node, a CNI/CSI node plugin that needs access to host device files or sockets. It is **not** an appropriate general-purpose persistence mechanism for application data — that's what PVCs are for.

The security angle matters more than the persistence angle here: `hostPath` (especially with a broad path like `/` or write access) is one of the most direct routes to **container escape / node compromise**, since a compromised container with a writable `hostPath` mount can potentially modify files the node's own OS or kubelet relies on. This is exactly why Pod Security Standards (Day 26) restrict or forbid `hostPath` under the `Restricted` and `Baseline` profiles.

## `emptyDir` vs `hostPath`, side by side

| | `emptyDir` | `hostPath` |
|---|---|---|
| Data survives Pod deletion? | No | Yes (it's the node's real filesystem) |
| Tied to a specific node? | No (recreated fresh wherever scheduled) | Yes (only meaningful on that exact node) |
| Typical use | Scratch space, inter-container sharing within a Pod | Infra-level DaemonSets needing host device/log access |
| Security risk | Low | High — direct path to node filesystem access |
| Appropriate for app data persistence? | No | No (use a PVC) |

## VolumeSnapshots — point-in-time backup/restore for PVCs

Built on the same CSI extension model as everything else in this day — a CSI driver that supports snapshotting exposes `CreateSnapshot`/`DeleteSnapshot`/`CreateVolumeFromSnapshot` operations, wrapped in Kubernetes objects:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata: { name: ebs-snapshot-class }
driver: ebs.csi.aws.com
deletionPolicy: Delete            # or Retain — same idea as PV reclaim policy
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: data-snapshot }
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: data-pvc
```

```bash
kubectl get volumesnapshot
kubectl describe volumesnapshot data-snapshot     # readyToUse: true once complete
kubectl get volumesnapshotcontent                  # cluster-scoped, analogous to PV for snapshots
```

Restoring means creating a **new** PVC that references the snapshot as its data source — it does not overwrite the original volume:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: restored-pvc }
spec:
  storageClassName: gp3
  dataSource:
    name: data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: { storage: 10Gi }
```

On AWS, an EBS `VolumeSnapshot` maps directly to a real **EBS Snapshot** (stored in S3 under the hood, incremental after the first) — so this feature isn't a Kubernetes-only abstraction, it's a native Kubernetes API surface over a real cloud backup primitive, meaning your snapshots survive even full cluster deletion.

## Points to Remember

- `emptyDir` is scoped to the Pod's lifetime on a node — great for scratch space and inter-container file sharing within one Pod, useless for anything you need to survive Pod deletion.
- `hostPath` ties data to one specific node and is primarily a security risk, not a persistence solution — restrict it to infra-level DaemonSets with a real need for host filesystem/device access.
- `VolumeSnapshot`/`VolumeSnapshotClass`/`VolumeSnapshotContent` mirror the PVC/StorageClass/PV pattern, and require the underlying CSI driver to support the snapshot extension (not all do).
- Restoring from a snapshot always creates a **new** PVC via `dataSource` — it never mutates the PVC the snapshot was taken from.
- Snapshot `deletionPolicy` (`Retain` vs `Delete`) controls whether deleting the `VolumeSnapshot` object also deletes the underlying cloud snapshot — same safety consideration as PV reclaim policy.

## Common Mistakes

- Using `emptyDir` for anything expected to survive a pod restart/reschedule (e.g., "temporary" upload storage that turns out to matter) — data loss the moment the Pod moves or is deleted, with no warning.
- Reaching for `hostPath` as a quick persistence hack in early development, then carrying that pattern into production, where it silently breaks the moment the Pod gets rescheduled to a different node (or becomes a real security finding in an audit).
- Assuming `medium: Memory` `emptyDir` is just "a faster disk" — it's actually RAM, counted against the Pod's memory limit, and gone immediately (not just on Pod deletion, but on any container restart within the Pod, since tmpfs contents don't survive that either in some configurations — always verify for your exact setup).
- Forgetting that not every CSI driver implements the snapshot extension — attempting to create a `VolumeSnapshot` against a driver/StorageClass that doesn't support it will simply hang in a non-ready state with no useful top-level error until you check the driver's own logs.
- Treating a `VolumeSnapshot` as a full disaster-recovery solution on its own without testing restores — an untested backup (snapshot or otherwise) is not a real backup; periodically prove you can actually create a `restored-pvc` and mount it successfully.
