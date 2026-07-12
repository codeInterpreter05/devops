# Day 50 — Quiz: Probes, Resources & QoS

Try to answer without looking at your notes. Answers are at the bottom.

1. What is the difference between what a liveness probe failure does versus what a readiness probe failure does?
2. Why should a liveness probe generally avoid checking downstream dependencies like a database connection?
3. What problem do startup probes solve that `initialDelaySeconds` on a liveness probe alone cannot solve well?
4. What does a `requests` value control, and what does a `limits` value control? Why are they used by different subsystems?
5. Why does exceeding a CPU limit result in throttling, while exceeding a memory limit results in the container being killed?
6. What are the three QoS classes, how is each derived, and what does the class determine during node memory pressure?
7. A node shows 20% CPU utilization in Grafana, but a new pod still fails to schedule with "Insufficient cpu." What's the likely explanation?
8. What does `OOMKilled` with exit code 137 actually mean at the kernel level?
9. Name two distinct root causes that can both produce an `OOMKilled` status on the same container, and how you'd tell them apart.
10. What's the difference between what a LimitRange controls and what a ResourceQuota controls?
11. If a namespace has a ResourceQuota specifying `requests.memory`, what happens if you create a pod with no resources block at all? How does pairing it with a LimitRange change that outcome?
12. **Interview question:** What is the difference between a liveness and readiness probe? What happens if readiness fails vs liveness fails?

---

## Answers

1. Liveness probe failure causes the kubelet to **kill and restart** the container. Readiness probe failure **removes the pod from the Service's Endpoints** (stops routing traffic to it) without restarting anything — the container keeps running.
2. Because a downstream dependency being briefly unavailable doesn't mean *this process* is broken or needs a restart — restarting does nothing to fix an external outage, destroys in-memory state/connections, and can cause a restart storm across every replica simultaneously. That situation should make the pod fail its **readiness** probe (temporarily stop receiving traffic) while staying alive and ready to resume once the dependency recovers.
3. Startup probes give a slow-booting app a one-time, generous startup budget during which liveness/readiness checks are suppressed entirely. Using only `initialDelaySeconds` forces a single fixed delay that either wastes time on the common fast-boot case or still causes restart loops whenever startup occasionally exceeds that fixed value.
4. `requests` is used by the **scheduler** to decide which node has enough reserved-but-unused capacity to place a pod. `limits` is enforced at **runtime** by the kubelet/container runtime as a hard ceiling. They're handled by different subsystems because scheduling is a placement decision made once, while limit enforcement is a continuous runtime guarantee.
5. CPU is a compressible, time-sliceable resource — a process can simply be given less CPU time without losing any data, so the kernel throttles it via CFS quotas. Memory is incompressible — once allocated, there's no safe way to reclaim it without destroying data, so the only enforcement option when a memory limit is breached is for the kernel's OOM killer to terminate the process.
6. Guaranteed (requests == limits for both CPU and memory, on every container), Burstable (some requests/limits set but doesn't qualify for Guaranteed), BestEffort (no requests/limits set at all). Under node memory pressure, BestEffort pods are evicted first, then Burstable (worst overage first), and Guaranteed pods are evicted last.
7. The scheduler reasons about **requested** capacity, not actual/used capacity. Even if usage is low, if the sum of all pods' *requests* already on that node nearly equals the node's allocatable capacity, the scheduler will refuse to place a new pod whose requests don't fit — regardless of how idle the node actually is.
8. It means the container's memory usage exceeded its cgroup memory limit, and the Linux kernel's OOM killer sent `SIGKILL` (signal 9) to the process. Exit code 137 = 128 + 9, the standard convention for "process terminated by signal N."
9. (a) The container's own memory limit is genuinely too low for what the app needs (leak or undersized limit) — confirmed by the container itself exceeding its cgroup limit independent of the node. (b) The whole node was under memory pressure from the combined usage of multiple pods, and the kubelet's eviction manager killed lower-QoS pods node-wide to relieve it — check `kubectl describe node` for a `MemoryPressure` condition and `kubectl get events` to distinguish a node-level eviction from a container specifically breaching its own limit.
10. LimitRange sets **per-object** defaults and min/max/ratio bounds within a namespace (applies to each container/pod individually). ResourceQuota caps the **aggregate total** across the whole namespace (sum of all requests/limits, total pod/object counts). LimitRange constrains individual objects; ResourceQuota constrains the namespace as a whole.
11. The pod creation is **rejected** at admission time with a quota error, because a ResourceQuota on `requests.memory` requires every pod in that namespace to explicitly specify that value. Pairing it with a LimitRange that sets a `defaultRequest` (and `default` for limits) auto-fills those values on any container that omits them, so the pod is admitted successfully instead of being rejected.
12. Strong answer: "A liveness probe tells Kubernetes whether the container process itself is healthy enough to keep running — if it fails repeatedly, the kubelet kills and restarts the container. A readiness probe tells Kubernetes whether the pod is currently able to serve traffic — if it fails, the pod is removed from the Service's Endpoints so no traffic is routed to it, but the container keeps running untouched and can flip back to ready without any restart. The key operational distinction is that liveness failures should be reserved for genuine, unrecoverable process-level problems (deadlocks, hangs) since a restart is a heavy-handed fix, while readiness failures are meant for transient conditions (warming up, a downstream dependency briefly down) that don't require killing the process at all — using liveness for a downstream dependency issue is a very common production mistake that causes unnecessary restart storms."
