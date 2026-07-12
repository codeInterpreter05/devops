# Day 22 — K8s Architecture Deep Dive: The Control Plane

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Every Kubernetes cluster is really two clusters glued together: a small, tightly-coupled **control plane** that makes decisions, and a fleet of **worker nodes** that actually run your containers. If you can't draw the control plane from memory and explain what each piece is responsible for, you can't reason about failure modes ("the API server is down, can I still run `kubectl get pods`?" — no, but running pods keep running), and you'll fumble the single most common Kubernetes interview question: "what happens when you `kubectl apply`?"

This day is split into three files:

1. **This file** — the control plane: `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager` (+ `cloud-controller-manager`).
2. **[02-README-Worker-Node.md](02-README-Worker-Node.md)** — the worker node: `kubelet`, `kube-proxy`, the container runtime, and the CRI/CNI/CSI plugin model.
3. **[03-README-Pod-Scheduling-Lifecycle.md](03-README-Pod-Scheduling-Lifecycle.md)** — tying it all together: a step-by-step trace of `kubectl apply -f pod.yaml`, and an etcd deep dive.

## The control plane, component by component

The control plane is the "brain" — it holds the cluster's desired state, watches the actual state, and reconciles the two. In a managed service (EKS, GKE, AKS) you don't see these components as pods you manage — the cloud provider runs them for you — but conceptually they're still there and behave identically. In `minikube`/`kubeadm` clusters you can literally see them as static pods in the `kube-system` namespace.

### `kube-apiserver` — the front door

The API server is the **only** component that talks to `etcd` directly, and it's the only entry point for every other component and every `kubectl` command. It is a stateless, horizontally-scalable REST server that:

- Validates and admits requests (authentication → authorization (RBAC) → admission control, in that order).
- Persists the validated object to `etcd`.
- Serves the **watch** API — every other component (scheduler, controllers, kubelets) doesn't poll; it opens a long-lived HTTP watch connection and receives a stream of change events.

```bash
kubectl get --raw /apis                     # see every API group the apiserver exposes
kubectl get --raw /healthz                   # raw health check, bypasses normal object handling
kubectl proxy --port=8080                    # local proxy that authenticates for you, useful for curl-ing the API directly
curl http://localhost:8080/api/v1/pods
```

Because the apiserver is stateless, you can run 3+ replicas behind a load balancer for HA — none of them hold state in memory that the others don't have (they all read/write the same `etcd`). This is *why* Kubernetes scales its control plane horizontally so easily compared to, say, a traditional database-backed monolith.

### `etcd` — the source of truth

`etcd` is a distributed, consistent key-value store (using the **Raft** consensus algorithm) and it is the **only place cluster state actually lives**. Every Kubernetes object — every Pod, Deployment, Secret, ConfigMap — is a key under `/registry/...` in `etcd`. Nothing else in the cluster is a database; the scheduler and controllers are stateless processes that derive everything from watching the apiserver.

Why this matters operationally:
- **Raft requires a quorum** — for a cluster of `N` etcd members, you can tolerate `(N-1)/2` failures. This is why production etcd clusters are almost always **3 or 5 members**, never an even number (an even number adds nodes without adding fault tolerance and increases the chance of a split-brain-adjacent stall).
- etcd is **latency-sensitive** — it's typically run on fast local SSDs, and running it on slow/network-attached disks is a classic cause of API server timeouts and a "sluggish" cluster.
- Losing etcd = losing the cluster's brain. Running pods keep running (kubelets don't need etcd for pods already scheduled to them), but nothing new can be scheduled, no config changes can be applied, and `kubectl get` calls will hang/fail. **Back up etcd** (`etcdctl snapshot save`) — it's the single most important backup in a self-managed cluster.

```bash
# Inspecting etcd directly (only possible if you manage etcd yourself, not on EKS/GKE)
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods/default/my-pod

# Snapshot backup — the single most important etcd operational command
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db
```

### `kube-scheduler` — decides *where* a Pod runs

The scheduler watches the apiserver for Pods with an empty `.spec.nodeName` (meaning "unscheduled") and assigns them to a node. It never talks to `etcd` directly and it never actually *starts* anything — its entire job is to write a node name back onto the Pod object via the apiserver. Two-phase decision:

1. **Filtering (predicates)** — eliminate nodes that can't run the pod at all: insufficient CPU/memory requests available, node selector / affinity mismatch, taints the pod doesn't tolerate, port conflicts, volume zone mismatch.
2. **Scoring (priorities)** — rank the remaining nodes: least-requested resources, pod (anti-)affinity spread, image already present locally, topology spread constraints. Highest score wins; ties are broken randomly to avoid piling everything on one "best" node.

```bash
kubectl get events --field-selector involvedObject.kind=Pod,reason=Scheduled
kubectl describe pod <pod>    # bottom "Events" section shows the scheduler's decision (or why it's still Pending)
```

### `kube-controller-manager` — reconciliation loops

This is a single binary that bundles dozens of independent **controllers**, each running the same pattern forever: *observe current state → compare to desired state → take action to close the gap*. Examples: the **Deployment controller** (creates/updates ReplicaSets), the **ReplicaSet controller** (creates/deletes Pods to match `replicas:`), the **Node controller** (marks nodes `NotReady` and evicts pods after a timeout), the **Endpoints controller** (keeps Service endpoint lists in sync with matching Pods).

Every controller follows the same **level-triggered, not edge-triggered** reconciliation model — it doesn't care *what changed*, it just keeps re-checking "is reality equal to spec?" This is why Kubernetes is self-healing and also why it's *eventually* consistent, not instantly consistent: there's always a reconcile loop tick between "you changed the spec" and "the world matches it."

### `cloud-controller-manager` — the cloud-specific glue

Splits out the cloud-provider-specific logic (creating an AWS ELB when you create a `LoadBalancer` Service, attaching EBS volumes, labeling nodes with their AWS instance type/AZ) from the core `kube-controller-manager`, so core Kubernetes doesn't need to know anything about AWS/GCP/Azure APIs. On EKS this runs as the `aws-cloud-controller-manager` (or is baked into the managed control plane, invisible to you).

## Points to Remember

- Only `kube-apiserver` talks to `etcd`. Every other component watches the apiserver — nothing polls a database directly.
- `etcd` uses Raft and needs a quorum: run 3 or 5 members, never an even number, and put it on fast local disk.
- The scheduler only **decides**, it never **executes** — it writes `nodeName` onto the Pod; the kubelet on that node does the actual work.
- `kube-controller-manager` bundles many independent, level-triggered reconciliation loops — this is the source of Kubernetes' self-healing behavior.
- In managed Kubernetes (EKS/GKE/AKS) you don't operate these components, but you're still responsible for reasoning about their behavior (e.g., API server rate limits, etcd-backed etcd-latency-driven slowness) during incidents.

## Common Mistakes

- Saying "the scheduler starts the pod" — it doesn't; it only assigns a node. The kubelet on that node pulls the image and starts the container.
- Assuming a control-plane outage kills running workloads — it doesn't. Pods already scheduled keep running via the kubelet; you just lose the ability to schedule new work or make changes until the control plane recovers.
- Running etcd with 2 or 4 members "for a bit more redundancy" — this doesn't improve fault tolerance (quorum math is `(N-1)/2` failures tolerated) and just adds another vote that can be lost.
- Confusing `kube-controller-manager` (in-cluster, cloud-agnostic controllers like Deployment/ReplicaSet/Node) with `cloud-controller-manager` (cloud-specific glue like provisioning ELBs) — they're separate binaries with separate responsibilities.
- Forgetting that on EKS/GKE you cannot `kubectl exec` into or directly inspect control-plane pods (they aren't exposed) — you rely on CloudTrail/audit logs and provider-published metrics instead of `etcdctl`/pod logs.
