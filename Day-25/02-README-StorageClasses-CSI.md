# Day 25 — Storage in K8s: StorageClasses, Dynamic Provisioning & CSI Drivers

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Manually pre-creating a PersistentVolume for every PVC an application might ever request (static provisioning) doesn't scale operationally — nobody wants to hand-provision an EBS volume every time a developer creates a new PVC. `StorageClass` plus dynamic provisioning via CSI drivers is what makes storage in Kubernetes feel as elastic as compute: you ask for storage, and it materializes automatically from the underlying cloud, on demand.

## `StorageClass`: the template for on-demand storage

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com          # which CSI driver handles this class
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete                  # default if omitted
volumeBindingMode: WaitForFirstConsumer   # important — see below
allowVolumeExpansion: true
```

When a PVC names `storageClassName: gp3`, the StorageClass's `provisioner` field tells Kubernetes *which CSI driver* to call to actually create a new volume — no admin pre-provisioning required. Clusters typically have a **default StorageClass** (annotated `storageclass.kubernetes.io/is-default-class: "true"`) used automatically when a PVC omits `storageClassName` entirely.

### `volumeBindingMode` — a subtle but important field

- **`Immediate`** (older default): the PV is provisioned as soon as the PVC is created, *before* any Pod using it is scheduled. Problem: for zone-locked storage like EBS (an EBS volume lives in exactly one AZ), the volume might get created in `us-east-1a` while the scheduler later decides to place the Pod in `us-east-1b` for unrelated reasons (resource availability) — resulting in a Pod that can never actually mount its volume.
- **`WaitForFirstConsumer`** (the correct choice for zonal cloud storage, and the modern default for most cloud CSI StorageClasses): delays provisioning until a Pod using the PVC is actually scheduled, so the volume is created in the **same AZ the scheduler already chose** for the Pod. This single setting eliminates an entire class of "PVC bound but Pod stuck in Pending forever" incidents caused by AZ mismatches.

```bash
kubectl get storageclass                                    # see default (marked with (default))
kubectl describe storageclass gp3
kubectl get pvc data-pvc -o jsonpath='{.spec.storageClassName}'
```

## Access modes: what "can be mounted by multiple pods" actually means

Access modes describe how a PVC can be mounted **across nodes**, not how many pods on the *same* node can use it:

- **`ReadWriteOnce` (RWO)** — the volume can be mounted read-write by only **one node** at a time. (Kubernetes 1.22+ actually lets multiple Pods *on that same node* share an RWO mount — worth knowing since this quietly changed from the older "one Pod only" understanding many people still repeat.) This is what block storage like EBS provides — a single EC2 instance attaches an EBS volume, period.
- **`ReadOnlyMany` (ROX)** — many nodes can mount it simultaneously, but strictly read-only.
- **`ReadWriteMany` (RWX)** — many nodes can mount it simultaneously, read-write. Requires a storage backend that supports concurrent multi-node access — **block storage like EBS cannot do this**; you need a network filesystem like **EFS** (NFS-based) or Azure Files.
- **`ReadWriteOncePod` (RWOP)**, newer — an even stricter guarantee than RWO: only a **single Pod** in the entire cluster can mount it read-write, not just a single node. Useful for workloads (certain databases) where even accidental double-mounting from two pods that happen to land on the same node would corrupt data.

**The single most common storage architecture mistake** is requesting `ReadWriteMany` on an EBS-backed StorageClass to let multiple Pods (e.g., replicas of a Deployment) share one volume — EBS fundamentally does not support this; the PVC will either fail to bind or the second Pod trying to mount it will fail to schedule/attach. RWX needs EFS/NFS-class storage, full stop.

## CSI: the plugin interface for storage drivers

Like CRI for container runtimes, **CSI (Container Storage Interface)** is a standard gRPC interface that decouples Kubernetes from any specific storage backend's implementation details. A CSI driver has (typically) two pieces:

- **Controller plugin** — runs as a Deployment, talks to the cloud API to actually create/delete/attach/detach volumes (e.g., calls `CreateVolume`/`AttachVolume` against the AWS EBS API).
- **Node plugin** — runs as a DaemonSet (one per node, naturally — see Day 23), handles the node-local `mount`/`unmount`/format operations the kubelet delegates to it.

```bash
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl get pods -n kube-system -l app=ebs-csi-node          # one per node
kubectl get csidrivers                                          # every CSI driver registered in the cluster
kubectl get volumeattachments                                    # which PV is attached to which node, right now
```

On EKS, the **AWS EBS CSI driver** is installed as an add-on (either via `eksctl create addon` or Helm) and requires an **IAM role** (commonly granted via IRSA, see Day 26) so its controller pod can actually call the AWS EBS API on your behalf — a very common first-time setup gap is forgetting this IAM permission, which causes PVCs to sit `Pending` forever with a clear `AccessDenied` reason in `kubectl describe pvc` events once you look.

## Points to Remember

- `StorageClass.provisioner` names the CSI driver responsible for creating volumes on demand — this is what makes "just create a PVC and storage appears" possible, with zero manual admin steps.
- `volumeBindingMode: WaitForFirstConsumer` should be the default choice for any zonal cloud block storage (EBS, managed disks) — it avoids provisioning a volume in the wrong AZ relative to where the Pod actually gets scheduled.
- Access modes describe cross-node mount concurrency, not cross-pod: RWO now permits multiple pods on the *same* node since Kubernetes 1.22; RWOP is the newer, stricter single-pod-cluster-wide guarantee.
- RWX (`ReadWriteMany`) requires network filesystem storage (EFS/NFS/Azure Files) — block storage (EBS) fundamentally cannot support it, no matter how the StorageClass is configured.
- CSI drivers have a controller component (cloud API calls) and a node component (local mount/attach), mirroring the CRI's split between "decide" and "execute" — and on EKS, the controller needs real IAM permissions to function.

## Common Mistakes

- Requesting `ReadWriteMany` against an EBS-backed StorageClass expecting it to "just work" for a scaled-out Deployment — it structurally cannot; the fix is either EFS-backed storage for RWX or redesigning so each replica gets its own RWO volume (e.g., via a StatefulSet's `volumeClaimTemplates`).
- Leaving `volumeBindingMode: Immediate` on a zonal storage class and being confused by pods stuck `Pending` with a volume-node-affinity-conflict error — the fix is `WaitForFirstConsumer`, not manually pinning pods to a zone.
- Forgetting to grant IAM permissions (IRSA) to the EBS/EFS CSI controller's ServiceAccount on EKS, then spending time debugging "PVC won't bind" as a Kubernetes problem when it's actually an AWS `AccessDenied` surfaced only in PVC events.
- Assuming there's always a default StorageClass — a fresh self-managed cluster (e.g., `kubeadm` without a cloud provider integration configured) has none by default, and PVCs with no explicit `storageClassName` will sit `Pending` indefinitely with no provisioner to satisfy them.
- Confusing `allowVolumeExpansion` (lets you grow a PVC's size later with `kubectl edit pvc`) with something that also shrinks volumes — Kubernetes does not support shrinking a PVC's requested storage size; expansion is one-directional.
