# Day 25 — Resources: Storage in K8s

## Primary (assigned)

- **Kubernetes.io Concepts — Storage** (kubernetes.io/docs/concepts/storage/) — free, the assigned starting point. Covers PV/PVC, StorageClasses, access modes, and volume snapshots with the authoritative field-level detail.

## Deepen your understanding

- **AWS EBS CSI Driver GitHub repo** (github.com/kubernetes-sigs/aws-ebs-csi-driver) — README and examples directory cover IRSA setup, StorageClass parameters, and volume snapshot support specifically for EKS.
- **AWS EFS CSI Driver GitHub repo** (github.com/kubernetes-sigs/aws-efs-csi-driver) — the RWX counterpart; useful for understanding exactly why/how EFS differs from EBS at the provisioning level.
- **Kubernetes.io — CSI Volume Cloning and Snapshots** (kubernetes.io/docs/concepts/storage/volume-snapshots/) — the full snapshot/restore object model referenced in file 3.
- **"Kubernetes Patterns" by Bilgin Ibryam & Roland Huß** (O'Reilly) — the "Stateful Service" and storage-related chapters connect PV/PVC mechanics to real StatefulSet-based architecture decisions.

## Reference / lookup

- `kubectl explain persistentvolumeclaim.spec` / `kubectl explain storageclass` — always in sync with your cluster's installed API version.
- **Kubernetes API Reference** (kubernetes.io/docs/reference/generated/kubernetes-api) — exact semantics of `volumeBindingMode`, `persistentVolumeReclaimPolicy`, and `dataSource`.

## Practice

- **KillerCoda Kubernetes storage scenarios** (killercoda.com/kubernetes) — free browser-based labs covering PV/PVC lifecycle and StorageClasses without needing a real cloud account.
- Practice this day's lab end-to-end on a real EKS cluster if you have access (or a free-tier-eligible cluster) — the AZ-mismatch and IAM-permission failure modes covered in file 2 are genuinely hard to internalize without hitting them once for real.
