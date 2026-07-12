# Day 55 — Cheatsheet: CKA Exam Prep II

## etcd backup / restore

```bash
export ETCDCTL_API=3     # always — v2 is the default and mostly wrong

# find cert paths / port from the static pod manifest
grep -E 'cert-file|key-file|trusted-ca-file|listen-client-urls' /etc/kubernetes/manifests/etcd.yaml

# snapshot
etcdctl snapshot save /opt/backup/snap.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# verify (no connection flags needed — reads local file)
etcdctl --write-out=table snapshot status /opt/backup/snap.db

# restore into a NEW data dir (never overwrite the live one)
etcdctl snapshot restore /opt/backup/snap.db --data-dir=/var/lib/etcd-restored

# then: edit /etc/kubernetes/manifests/etcd.yaml
#   - change hostPath.path to /var/lib/etcd-restored
#   - saving the file auto-triggers kubelet to recreate the static pod

etcdctl member list --endpoints=... --cacert=... --cert=... --key=...
etcdctl endpoint health --endpoints=... --cacert=... --cert=... --key=...
```

## Cluster upgrade (kubeadm)

```bash
apt-cache madison kubeadm                    # see available versions
apt-mark unhold kubeadm && apt install -y kubeadm=<ver> && apt-mark hold kubeadm

kubeadm upgrade plan                          # first control-plane node only
kubeadm upgrade apply v<version>              # first control-plane node
kubeadm upgrade node                          # additional control-plane / worker nodes

kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
apt-mark unhold kubelet kubectl && apt install -y kubelet=<ver> kubectl=<ver> && apt-mark hold kubelet kubectl
systemctl daemon-reexec && systemctl restart kubelet
kubectl uncordon <node>

kubectl get nodes -o wide                     # confirm KUBELET-VERSION everywhere
```
Order: control plane fully upgraded → then workers, one at a time. One minor version at a time only.

## NetworkPolicy

```yaml
# default-deny all ingress for app=backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: deny-all-ingress, namespace: prod}
spec:
  podSelector: {matchLabels: {app: backend}}
  policyTypes: [Ingress]
```
```yaml
# allow only from app=frontend on port 8080
spec:
  podSelector: {matchLabels: {app: backend}}
  policyTypes: [Ingress]
  ingress:
    - from: [{podSelector: {matchLabels: {app: frontend}}}]
      ports: [{protocol: TCP, port: 8080}]
```
- No policy selecting a pod = fully open (default-allow).
- First policy selecting a pod = default-deny for that direction; rules are additive/OR'd across policies.
- Requires an enforcing CNI: Calico / Cilium / Weave = yes. Plain Flannel = no enforcement.

## RBAC

```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=dev:ci-deployer -n dev

kubectl create clusterrole node-reader --verb=get,list --resource=nodes
kubectl create clusterrolebinding read-nodes --clusterrole=node-reader --serviceaccount=dev:ci-deployer

# bind a ClusterRole but scoped to one namespace (very common pattern)
kubectl create rolebinding dev-viewers --clusterrole=view --serviceaccount=dev:ci-deployer -n dev

kubectl auth can-i delete pods --as=system:serviceaccount:dev:ci-deployer -n dev
kubectl auth can-i --list --as=jane -n dev
kubectl create token ci-deployer -n dev --duration=1h
```
| Object | Scope |
|---|---|
| Role | one namespace |
| RoleBinding | one namespace (can bind a ClusterRole, scoped to that namespace) |
| ClusterRole | cluster-wide definition (usable namespaced or cluster-wide) |
| ClusterRoleBinding | cluster-wide |

## Cluster debugging

```bash
kubectl get nodes                              # node health
kubectl get pods -n kube-system                # control plane + CNI health
kubectl describe pod <pod> -n <ns>              # check Events section first

# when the API server itself might be down:
crictl ps -a                                    # talks to container runtime directly (CRI)
crictl logs <container-id>
journalctl -u kubelet -f                        # kubelet is a systemd unit, not a static pod

cat /etc/kubernetes/manifests/*.yaml            # inspect static pod manifests for bad edits
```
