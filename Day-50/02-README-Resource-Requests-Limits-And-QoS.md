# Day 50 — Probes, Resources & QoS: Requests, Limits & QoS Classes

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Requests and limits are how Kubernetes turns a shared pile of CPU/memory across a cluster's nodes into something schedulable and (mostly) fair. Get this wrong and you get one of two classic failure modes: pods that can't be scheduled anywhere ("Insufficient cpu" events) because requests are set unrealistically high, or a "noisy neighbor" problem where one pod with no limits starves everything else on the node because requests were set unrealistically low (or omitted). The QoS class Kubernetes derives from your requests/limits is what decides *who gets evicted first* when a node runs low on resources — understanding this mechanism is what separates "I set some numbers in YAML" from actually understanding scheduling and eviction behavior.

## Requests vs. limits — two completely different jobs

- **`requests`** — what the **scheduler** uses to decide *which node* a pod can fit on. The scheduler sums up all requests already placed on a node and only schedules a new pod if the node has enough *unrequested* capacity left. Requests are a **reservation**, not a hard ceiling — a container can use more than its request if the node has spare capacity at that moment.
- **`limits`** — the **hard ceiling** the kubelet/container runtime enforces at runtime, independent of scheduling. For CPU, exceeding the limit means the container is **throttled** (CFS quota mechanism — it doesn't get killed, its CPU time is just capped, causing latency/slowdowns). For memory, exceeding the limit means the container is **OOMKilled** (memory can't be "throttled" the way CPU can — there's no way to partially deny a memory allocation once the app has already been granted it, so the kernel's OOM killer intervenes).

```yaml
resources:
  requests:
    cpu: "250m"        # 0.25 vCPU reserved for scheduling purposes
    memory: "256Mi"
  limits:
    cpu: "500m"        # hard cap — throttled above this
    memory: "512Mi"    # hard cap — OOMKilled above this
```

**Why CPU and memory are handled so differently at the limit boundary** comes down to what the resource actually *is*: CPU is a compressible, time-sliceable resource (you can just give a process less time on the CPU without losing anything — it's slower, not broken). Memory is incompressible — once a byte is allocated, taking it away means losing data, so the kernel's only real option when limits are breached is to kill the offending process outright.

## How the scheduler actually uses requests

The scheduler's job during the **Filter** phase is roughly: `node.allocatable - sum(requests of all already-scheduled pods) >= new pod's requests?` for every resource type. This is why a cluster can show low *actual* CPU utilization (say 20%) via `kubectl top nodes` while still refusing to schedule a new pod with "0/5 nodes are available: insufficient cpu" — the scheduler cares about **requested**, not **used**, capacity. This is the single most common "why won't my pod schedule" confusion in real clusters: people look at `top`/Grafana showing spare capacity and don't realize every pod's *request* — regardless of actual usage — is what's being counted.

```bash
kubectl describe node <node-name>       # shows "Allocated resources" section — requests vs. capacity, per resource
kubectl top pod                          # shows actual live usage — a completely different number from requests
```

## QoS classes — derived automatically, not set directly

Kubernetes assigns each pod one of three **Quality of Service** classes based purely on how you set requests/limits — there's no `qosClass:` field you write yourself:

| Class | How it's assigned | Eviction priority under node pressure |
|---|---|---|
| **Guaranteed** | Every container in the pod has `limits == requests` for **both** CPU and memory (and both must be explicitly set) | **Last** to be evicted — Kubernetes treats these as the most protected workloads |
| **Burstable** | At least one container has a request or limit set for CPU or memory, but doesn't qualify for Guaranteed | Evicted **after BestEffort, before Guaranteed** — ranked further by how far actual usage exceeds requests |
| **BestEffort** | No requests or limits set at all, on any container in the pod | **First** to be evicted under node memory pressure |

Check it directly:
```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

**Why this matters for real production behavior**: when a node hits memory pressure, the kubelet doesn't wait for the kernel OOM killer to pick a victim somewhat arbitrarily — it proactively evicts pods in QoS order (BestEffort first, then Burstable pods furthest over their requests, and only touches Guaranteed pods as an absolute last resort). This means a critical, latency-sensitive service should almost always be **Guaranteed**, while a batch job or a dev/staging workload can reasonably be **Burstable** or even **BestEffort**.

## Choosing requests/limits values in practice

- **Right-sizing requests**: base them on observed steady-state usage (via `kubectl top`, Prometheus/VPA history), not a guess. Setting requests far above real usage wastes cluster capacity (nodes look "full" to the scheduler while actually idle); setting them far below causes overcommitment and noisy-neighbor CPU throttling / memory pressure once real usage climbs.
- **CPU limits — a genuinely debated practice**: many teams deliberately **don't set a CPU limit at all** (only a request), because CPU throttling under a hard limit can cause latency spikes even when the node has spare CPU sitting idle (the CFS quota mechanism throttles per-period regardless of overall node headroom in older kernels/cgroup v1 setups). Setting only requests for CPU (making the pod Burstable, not Guaranteed) is a common, defensible pattern — just know it trades away the "Guaranteed" QoS protection.
- **Memory limits — almost always set these**, because unlike CPU, an unbounded memory leak or spike in one pod can crash the whole node (taking down every other pod on it, including unrelated ones) rather than "just" throttling that one workload.

## Points to Remember

- Requests drive **scheduling** (which node); limits drive **runtime enforcement** (throttle for CPU, OOMKill for memory). They answer different questions and are enforced by different mechanisms.
- CPU limit breaches throttle (compressible resource, no data loss, just slower); memory limit breaches kill the container (incompressible, no safe way to reclaim already-allocated memory).
- QoS class (Guaranteed/Burstable/BestEffort) is derived automatically from requests/limits, not set directly — Guaranteed requires `requests == limits` for both CPU and memory on every container.
- Eviction order under node pressure is BestEffort first, then Burstable (worst-overage first), Guaranteed last — put your most critical workloads in Guaranteed.
- The scheduler reasons about **requested** capacity, not **actual/used** capacity — a node can look idle in monitoring while still being "full" from the scheduler's point of view.

## Common Mistakes

- Confusing "my node's CPU usage is only 20%" with "there's room to schedule more pods" — scheduling is based on requests, not live usage; low utilization with full requested capacity means no more pods will schedule there.
- Omitting resource requests/limits entirely on a workload "to keep things simple," unintentionally making it BestEffort — the first thing evicted under any node memory pressure, often with no warning until it happens in production.
- Setting `requests == limits` for memory (common, sensible) but not doing the same for CPU, then being surprised the pod isn't Guaranteed — Guaranteed requires equality on **both** resources, on **every** container in the pod (including sidecars/init containers, depending on version-specific overhead rules).
- Setting CPU limits far too tight based on average usage rather than peak/burst usage, causing constant throttling during traffic spikes even though the node has plenty of idle CPU to spare.
- Not distinguishing `kubectl top` (live actual usage) from `kubectl describe node`'s "Allocated resources" (sum of requests) when debugging scheduling issues — using the wrong number leads to the wrong diagnosis.
