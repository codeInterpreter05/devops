# Day 22 — K8s Architecture Deep Dive: The Worker Node

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

The control plane (previous file) decides *what should happen*. The worker node is where it actually *does* happen. Every node runs the same three components — `kubelet`, `kube-proxy`, and a container runtime — and understanding the boundary between them is what lets you debug "my pod is stuck in `ContainerCreating`" versus "my pod runs but I can't reach it" versus "my pod keeps getting OOMKilled" as three completely different classes of problem, each owned by a different component.

## `kubelet` — the node's agent

The kubelet is the only control-plane-facing component on a worker node. It:

- Watches the apiserver for Pods assigned to **its own node** (`spec.nodeName == this node`).
- Talks to the container runtime (via the **CRI** — Container Runtime Interface) to actually pull images and start/stop containers.
- Reports node and pod status back to the apiserver (this is what powers `kubectl get nodes` and `kubectl get pods` status columns).
- Runs **liveness, readiness, and startup probes** and acts on the results (restarting a container on failed liveness, pulling it out of Service endpoints on failed readiness).
- Enforces resource limits by configuring cgroups, and evicts pods under node memory/disk pressure.

```bash
kubectl describe node <node>          # Kubelet-reported capacity, allocatable, conditions
kubectl get node <node> -o yaml | grep -A5 conditions:   # Ready/MemoryPressure/DiskPressure/PIDPressure
journalctl -u kubelet -f              # kubelet's own logs, on the node itself (SSH/session-manager access needed)
```

**Static pods** are a special kubelet feature worth knowing: the kubelet can start pods directly from manifest files in a local directory (`/etc/kubernetes/manifests` by default), *without* going through the apiserver at all. This is exactly how `kubeadm` bootstraps the control plane itself — `kube-apiserver`, `etcd`, `kube-scheduler`, and `kube-controller-manager` are frequently just static pods that the kubelet on the control-plane node starts on its own, chicken-and-egg problem solved.

## Container runtime & the CRI

Since Kubernetes 1.24, Docker Engine itself is no longer supported as a runtime (`dockershim` was removed) — the kubelet only speaks **CRI (Container Runtime Interface)**, a gRPC API. Runtimes that implement CRI directly: **containerd** (most common on EKS/GKE/most managed clusters today) and **CRI-O**. Under the hood, containerd itself uses **runc** (or `gVisor`/`kata-containers` for stronger isolation) to actually create the Linux namespaces/cgroups that form a container — runc implements the lower-level **OCI runtime spec**.

```
kubelet --(CRI, gRPC)--> containerd --(OCI)--> runc --> Linux namespaces + cgroups
```

Why this layering exists: it decouples Kubernetes from any single container implementation. You can swap `containerd` for `CRI-O` without Kubernetes itself changing — as long as the CRI contract is honored.

```bash
# On a node with containerd, the CRI-native replacement for `docker ps`
crictl ps
crictl logs <container-id>
crictl inspect <container-id>
```

## `kube-proxy` — implements the Service abstraction on each node

A Kubernetes **Service** is a virtual IP (`ClusterIP`) that doesn't correspond to any real network interface — it's a routing rule. `kube-proxy` runs on every node and is responsible for making that virtual IP actually route traffic to one of the Service's backing Pods. It watches Endpoints/EndpointSlices from the apiserver and programs one of two backends:

- **iptables mode** (long-time default): writes `DNAT` rules so that traffic to a Service's `ClusterIP:port` gets rewritten to a randomly-chosen backend Pod IP:port. Simple, but rule evaluation is roughly O(n) in the number of Services — with thousands of Services, `iptables`-mode kube-proxy's rule sync becomes a measurable latency/CPU cost.
- **IPVS mode**: uses the kernel's IP Virtual Server, a real load balancer built for exactly this, with O(1) lookups and more load-balancing algorithm choices (round robin, least connection). Preferred at scale.

```bash
kubectl get endpoints <service-name>      # which pod IPs a Service currently routes to
kubectl get endpointslices                # the newer, scalable replacement for Endpoints
sudo iptables -t nat -L KUBE-SERVICES -n | head    # see the DNAT rules kube-proxy wrote (iptables mode)
ipvsadm -Ln                                          # see virtual servers (IPVS mode)
```

**Important nuance:** `kube-proxy` handles Service *routing*, but pod-to-pod networking (every pod getting a routable IP, cross-node connectivity) is a completely separate concern handled by the **CNI plugin** (Calico, Cilium, AWS VPC CNI, etc.) — `kube-proxy` assumes the underlying pod network already works and just adds Service-level load balancing on top of it.

## Points to Remember

- Three components per node: `kubelet` (talks to control plane + runtime), container runtime via CRI (`containerd`/`CRI-O`), `kube-proxy` (implements Services).
- The kubelet is the *only* worker-node component that talks to the apiserver — the runtime and kube-proxy don't communicate with the control plane directly.
- Static pods are started by the kubelet from local manifest files, bypassing the apiserver entirely — this is how self-hosted control planes bootstrap themselves.
- CRI decouples Kubernetes from any specific container engine; since 1.24, Docker Engine (dockershim) is no longer a supported runtime path — use `containerd` or `CRI-O`.
- `kube-proxy` (Service routing) and the CNI plugin (pod IP networking) are separate systems solving separate problems — don't conflate "Service isn't reachable" with "pod network is broken."

## Common Mistakes

- Debugging "pod can't reach a Service" by looking only at kube-proxy/iptables, when the actual problem is the CNI plugin (pod networking) or a NetworkPolicy blocking the traffic — check pod-to-pod connectivity directly first (`kubectl exec ... -- curl <podIP>`) to isolate which layer is broken.
- Confusing liveness and readiness probes: a failing **liveness** probe restarts the container; a failing **readiness** probe just removes it from Service endpoints (no restart). Assuming a readiness failure will "fix itself" via restart is wrong — it won't restart anything.
- Thinking Docker is still involved at runtime on modern clusters — since dockershim's removal (Kubernetes 1.24+), `docker ps` on a node won't show what the kubelet is managing; use `crictl` instead.
- Not realizing static pods (in `/etc/kubernetes/manifests`) won't show up in `kubectl get pods -A` reflecting apiserver-managed state changes the same way — editing them means editing the file on the node, not `kubectl edit`.
- Assuming `iptables`-mode kube-proxy scales indefinitely — with very large clusters/Service counts it becomes a real bottleneck, which is why IPVS mode (or eBPF-based dataplanes like Cilium replacing kube-proxy entirely) exist.
