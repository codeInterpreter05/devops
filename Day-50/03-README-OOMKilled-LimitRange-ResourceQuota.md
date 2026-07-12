# Day 50 — Probes, Resources & QoS: OOMKilled, LimitRange & ResourceQuota

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

`OOMKilled` is one of the most common status codes you'll debug in any real Kubernetes cluster, and understanding it requires connecting the dots between cgroups, the kernel's OOM killer, and your pod's memory limit. LimitRange and ResourceQuota are the two admission-time guardrails cluster/namespace admins use to stop teams from either forgetting to set resources at all, or requesting far more than a namespace should ever consume. Together, these three topics are what turns "requests and limits" from a YAML syntax exercise into an actual operational skill.

## What `OOMKilled` actually means, mechanically

When you set a memory `limit`, Kubernetes doesn't enforce it in userspace — it configures a **cgroup** (control group) memory limit for the container's process tree. If the container's total memory usage tries to exceed that cgroup limit, the **Linux kernel's OOM killer** intervenes and sends `SIGKILL` to a process inside that cgroup. Kubelet then observes the container exited with that signal and reports the container status as `OOMKilled` with exit code `137` (128 + 9, where 9 is `SIGKILL`'s signal number — `128 + N` is the standard convention for "killed by signal N").

```bash
kubectl describe pod <name>
# Look for:
#   Last State:     Terminated
#   Reason:         OOMKilled
#   Exit Code:      137
```

**Two distinct causes produce the exact same `OOMKilled` symptom**, and telling them apart is the actual debugging skill:
1. **The container's own memory limit is too low** for what the app legitimately needs (a memory leak, a workload that grew, or a limit that was copy-pasted from a different service without adjustment).
2. **The whole node is under memory pressure** (multiple pods collectively exceeding node capacity) — the kubelet's node-level eviction manager may kill lower-QoS pods node-wide *before* any single container even hits its own limit, as a way to protect overall node stability. This is a **pod eviction**, distinct from — but frequently confused with — a container-level OOMKill; check `kubectl get events` and `kubectl describe node` for `MemoryPressure` conditions to distinguish "this container specifically exceeded its own limit" from "the node evicted pods to relieve pressure."

**Practical diagnosis flow:**
```bash
kubectl top pod <name> --containers          # is usage climbing steadily (leak) or spiking (burst)?
kubectl describe pod <name>                   # confirm OOMKilled + exit code 137
kubectl get events --field-selector involvedObject.name=<name>
kubectl describe node <node> | grep -A5 Conditions   # check for MemoryPressure=True
```

## Preventing OOMKilled in practice

- **Right-size the memory limit** using real observed peak usage (not a guess), ideally with headroom above the highest legitimate spike you've measured — not the average.
- **Set memory requests equal to (or close to) limits** for memory-sensitive workloads — this pushes the pod toward Guaranteed/higher-protected Burstable QoS, reducing the odds it's picked first for node-level eviction even before it hits its own container limit.
- **Fix leaks, don't just raise the limit** — a limit bump masks a leak temporarily; the container will eventually hit the new, higher ceiling too. Use `kubectl top pod --containers` over time, or a language-specific profiler (heap dump for JVM, `pprof` for Go), to confirm whether usage is genuinely growing unbounded versus just needing a higher steady-state ceiling.
- **Language runtime memory settings must respect the container limit**, not just assume they own the whole node's RAM. Classic bug: a JVM with `-Xmx` unset (or set based on host RAM detection assumptions) allocates a heap sized for the *node's* total memory, not the *container's* cgroup limit, immediately blowing past a much smaller container memory limit. Modern JDKs (10+) are cgroup-aware by default, but older JDKs/misconfigured flags still cause this constantly in the wild.

## LimitRange — per-object defaults and bounds within a namespace

A `LimitRange` is a namespaced admission-time policy that:
- Sets **default requests/limits** applied automatically to any container in that namespace that doesn't specify its own (so `BestEffort`-by-omission doesn't silently happen).
- Enforces **min/max bounds** on what any single container/pod/PVC in the namespace may request (rejecting pod creation outright if it's outside range).
- Can constrain the **ratio of limit to request** (`maxLimitRequestRatio`), preventing wildly overcommitted containers (e.g., requesting 100m CPU but setting a 10-core limit).

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "250m"
        memory: "256Mi"
      min:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "2"
        memory: "2Gi"
      maxLimitRequestRatio:
        cpu: "4"
```

This is namespace-scoped and applies **per individual object** (per-container defaults/bounds) — it doesn't cap the namespace's total consumption. That's ResourceQuota's job.

## ResourceQuota — aggregate namespace-wide ceilings

A `ResourceQuota` caps the **total sum** across all objects in a namespace — total CPU/memory requested+limited, total pod count, total PVC count/storage, total count of a given object type (Services, Secrets, ConfigMaps, LoadBalancers, etc.).

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services.loadbalancers: "2"
```

**Critical interaction to know**: if a `ResourceQuota` in a namespace specifies `requests.cpu`/`requests.memory` (or their `limits.*` equivalents), **every pod created in that namespace must explicitly specify requests/limits for that resource**, or the API server will reject the pod outright at admission time (`failed quota: ... must specify cpu/memory`) — this is precisely why pairing a `ResourceQuota` with a `LimitRange` (which supplies sane defaults) is the standard combination: the quota enforces the ceiling, the LimitRange guarantees every pod actually has values to count against it, so nobody hits a confusing rejection just for omitting a `resources:` block.

## Points to Remember

- `OOMKilled` / exit code 137 = kernel OOM killer sent SIGKILL because the container's cgroup memory limit was exceeded — CPU limits throttle instead, they never OOMKill.
- The same `OOMKilled` symptom can mean either "this container alone exceeded its own limit" or "the node was under memory pressure and evicted lower-QoS pods" — check node conditions (`MemoryPressure`) to tell them apart.
- Raising a memory limit is a mitigation, not a fix, for a genuine memory leak — confirm via usage trend before just bumping the number.
- LimitRange = per-object defaults/min/max/ratio within a namespace. ResourceQuota = aggregate namespace-wide totals (sum of requests/limits, object counts). They solve different problems and are commonly used together.
- If a namespace has a ResourceQuota on `requests.cpu`/`requests.memory`, every pod in that namespace **must** specify those values explicitly or be rejected at creation — pairing it with a LimitRange avoids that friction by auto-filling sane defaults.

## Common Mistakes

- Bumping a memory limit repeatedly every time a pod OOMKills without ever checking whether usage is trending upward (a leak) versus just needing a higher steady ceiling — this delays fixing an actual bug and eventually just hits a bigger wall.
- Assuming an `OOMKilled` pod means that specific container's code is broken, when the real cause was node-wide memory pressure evicting a lower-QoS pod to protect the node — always check node conditions before assuming the app itself is at fault.
- Setting a JVM/other managed-runtime container's memory limit without configuring the runtime to respect the cgroup limit (old JDK versions, or explicit `-Xmx` flags computed from host RAM), causing the runtime to allocate more heap than the container is actually permitted, guaranteeing an eventual OOMKill.
- Deploying a `ResourceQuota` with `requests.cpu`/`requests.memory` set, without a companion `LimitRange`, and then being confused by every subsequent pod-without-resources getting rejected at creation with a cryptic quota error.
- Confusing LimitRange's per-container `max` with a namespace-wide cap — a LimitRange `max: cpu: 2` stops any *single* container from requesting more than 2 cores, but says nothing about how many *total* cores the whole namespace can consume across many pods; only ResourceQuota controls that aggregate.
