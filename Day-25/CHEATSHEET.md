# Day 25 — Cheatsheet: Storage in K8s

## PV / PVC

```bash
kubectl get pv                      # cluster-scoped: Available / Bound / Released / Failed
kubectl get pvc                     # namespaced: Pending / Bound
kubectl describe pv <name>
kubectl describe pvc <name>         # events show WHY it's Pending if stuck
kubectl get pv <name> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
kubectl patch pv <name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data-pvc }
spec:
  accessModes: ["ReadWriteOnce"]     # ROX, RWX, ReadWriteOncePod also exist
  storageClassName: gp3
  resources: { requests: { storage: 10Gi } }
```

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations: { storageclass.kubernetes.io/is-default-class: "true" }
provisioner: ebs.csi.aws.com          # efs.csi.aws.com for EFS/RWX
parameters: { type: gp3, encrypted: "true" }
volumeBindingMode: WaitForFirstConsumer   # avoids AZ mismatch for zonal storage
reclaimPolicy: Delete                       # or Retain
allowVolumeExpansion: true
```

```bash
kubectl get storageclass
kubectl describe storageclass <name>
```

## Access modes

```
RWO   ReadWriteOnce      one node, read-write (multiple pods on that node OK since 1.22+)
ROX   ReadOnlyMany        many nodes, read-only
RWX   ReadWriteMany        many nodes, read-write — needs EFS/NFS, NOT EBS
RWOP  ReadWriteOncePod     single POD cluster-wide, read-write (strictest)
```

## CSI

```bash
kubectl get csidrivers
kubectl get volumeattachments
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl get pods -n kube-system -l app=ebs-csi-node       # DaemonSet, one per node
eksctl create addon --cluster <name> --name aws-ebs-csi-driver \
  --service-account-role-arn <irsa-role-arn>
```

## emptyDir / hostPath

```yaml
volumes:
  - name: scratch
    emptyDir: { sizeLimit: 1Gi }        # medium: Memory -> tmpfs/RAM-backed
  - name: host-logs
    hostPath: { path: /var/log, type: Directory }
```

## Volume Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata: { name: ebs-snapshot-class }
driver: ebs.csi.aws.com
deletionPolicy: Delete
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: data-snapshot }
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source: { persistentVolumeClaimName: data-pvc }
```

```yaml
# Restore into a NEW pvc
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: restored-pvc }
spec:
  storageClassName: gp3
  dataSource: { name: data-snapshot, kind: VolumeSnapshot, apiGroup: snapshot.storage.k8s.io }
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 10Gi } }
```

```bash
kubectl get volumesnapshot
kubectl get volumesnapshotcontent
kubectl describe volumesnapshot <name>   # readyToUse: true/false
```
