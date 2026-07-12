# Day 22 — Cheatsheet: K8s Architecture Deep Dive

## Control plane components

```
kube-apiserver           only component that talks to etcd; authn -> authz -> admission -> persist
etcd                     distributed KV store (Raft); source of truth; needs quorum (3 or 5 members)
kube-scheduler           watches unscheduled pods; Filter -> Score -> Bind (writes nodeName only)
kube-controller-manager  bundles reconciliation loops: Deployment, ReplicaSet, Node, Endpoint, etc.
cloud-controller-manager cloud-specific glue: provisions LBs, attaches volumes, labels nodes w/ AZ
```

## Worker node components

```
kubelet       node agent; watches apiserver for pods bound to itself; talks to runtime via CRI
kube-proxy    programs iptables/IPVS so ClusterIP Services route to real pod IPs
container     containerd / CRI-O implement CRI; runc/gVisor/kata implement OCI runtime spec below that
runtime
CNI plugin    separate concern: gives every pod a routable IP (Calico, Cilium, AWS VPC CNI)
```

## Cluster inspection

```bash
kubectl cluster-info                          # control plane endpoint + core service addresses
kubectl get componentstatuses                 # deprecated but still shows scheduler/etcd/controller-manager health on some clusters
kubectl get nodes -o wide                     # node list w/ internal/external IP, OS, runtime version
kubectl describe node <node>                  # capacity, allocatable, conditions, running pods
kubectl get pods -n kube-system -o wide       # see control plane pods (self-managed / minikube / kubeadm)
kubectl get --raw /healthz                    # raw apiserver health
kubectl get --raw /apis                       # list API groups
kubectl proxy --port=8080                     # authenticated local proxy to the apiserver
```

## Scheduling / events

```bash
kubectl get events --watch
kubectl get events -o wide                    # includes reporting component column
kubectl describe pod <pod>                    # bottom Events section: Scheduled/Pulling/Pulled/Started or FailedScheduling
kubectl get pod <pod> -o jsonpath='{.spec.nodeName}'
kubectl get pod <pod> -o jsonpath='{.metadata.resourceVersion}'
```

## Static pods (kubeadm / minikube self-hosted control plane)

```bash
ls /etc/kubernetes/manifests/                 # kubelet starts these directly, no apiserver round-trip
cat /etc/kubernetes/manifests/etcd.yaml
journalctl -u kubelet -f                      # kubelet's own log stream
```

## Container runtime (CRI-native, post-dockershim)

```bash
crictl ps                                     # like `docker ps`, but via CRI
crictl images
crictl logs <container-id>
crictl inspect <container-id>
```

## kube-proxy internals

```bash
kubectl get endpoints <svc>                   # backend pod IPs a Service currently routes to
kubectl get endpointslices                    # scalable successor to Endpoints
iptables -t nat -L KUBE-SERVICES -n           # DNAT rules (iptables mode)
ipvsadm -Ln                                    # virtual servers (IPVS mode)
```

## etcd (self-managed clusters only)

```bash
ETCDCTL_API=3 etcdctl member list --write-out=table
ETCDCTL_API=3 etcdctl endpoint health
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db
ETCDCTL_API=3 etcdctl get /registry/pods/default/<name> --print-value-only
```

## minikube / k9s

```bash
minikube start --driver=docker --nodes=1
minikube ssh
minikube stop
minikube delete
k9s                                            # TUI; :pods, :ns, :node, l = logs, d = describe
```
