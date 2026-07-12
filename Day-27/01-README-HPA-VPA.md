# Day 27 — Autoscaling: HPA & VPA

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Autoscaling in Kubernetes operates on two independent axes that are frequently conflated in interviews: **how many pods** should exist (Horizontal Pod Autoscaler) and **how much CPU/memory each pod** should be given (Vertical Pod Autoscaler). Understanding both, and why you almost never want them fighting over the same metric on the same workload, is what separates "I set replicas: 3 and called it autoscaling" from actually designing a system that scales safely under real, bursty traffic.

This day is split into three files:

1. **This file** — HPA (CPU/memory/custom metrics) and VPA.
2. **[02-README-KEDA.md](02-README-KEDA.md)** — KEDA: event-driven autoscaling from external metrics (queue depth, etc.).
3. **[03-README-Cluster-Autoscaler-Karpenter.md](03-README-Cluster-Autoscaler-Karpenter.md)** — node-level autoscaling (Cluster Autoscaler vs Karpenter) and cooldown/flapping prevention.

## HPA: scaling the number of Pods

The **Horizontal Pod Autoscaler** watches a metric, compares it against a target, and adjusts a Deployment/StatefulSet's `replicas` count to try to bring the observed metric back to target. It requires the **metrics-server** add-on (for CPU/memory) to be running — HPA has no built-in metric collection of its own, it queries the `metrics.k8s.io` API that metrics-server (or a custom/external metrics API) provides.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: web-hpa }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70          # target: average pod CPU at 70% of its REQUEST (not limit)
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0        # react immediately to scale-up signals
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60                 # add at most 4 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300       # wait 5 min of sustained low usage before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
```

**Critical detail interviewers probe for:** CPU utilization percentage is calculated against the pod's **`resources.requests.cpu`**, not its `limits.cpu`. A pod with no CPU request set has no baseline for HPA to compute a percentage against, and HPA simply cannot scale on CPU for that workload at all — this is the single most common reason "my HPA shows `<unknown>` for current CPU" in real clusters.

```bash
kubectl get hpa web-hpa
kubectl describe hpa web-hpa               # current vs target metric, and recent scaling events
kubectl top pods                            # requires metrics-server; raw CPU/memory usage per pod
```

### Custom and external metrics

Beyond CPU/memory, HPA (`autoscaling/v2`) supports:
- **`Pods` metrics** — a metric averaged across all pods of the target (e.g., requests-per-second per pod), sourced via the **custom metrics API**, typically backed by Prometheus Adapter.
- **`Object` metrics** — a metric describing a *different* Kubernetes object (e.g., an Ingress's request rate).
- **`External` metrics** — a metric from **outside** the cluster entirely (an SQS queue depth, a Kafka consumer lag figure), sourced via the **external metrics API**. This is exactly the gap **KEDA** (next file) fills far more conveniently than hand-rolling a Prometheus Adapter config.

```yaml
  metrics:
    - type: External
      external:
        metric:
          name: sqs_queue_depth
          selector:
            matchLabels: { queue: orders }
        target:
          type: AverageValue
          averageValue: "30"
```

## VPA: scaling the size of each Pod

The **Vertical Pod Autoscaler** does the orthogonal thing — instead of changing replica count, it observes actual CPU/memory usage over time and recommends (or automatically applies) better `resources.requests`/`limits` values per container. It runs three components: a **Recommender** (watches usage history, computes suggestions), an **Updater** (evicts pods that deviate too far from the recommendation, if in `Auto`/`Recreate` mode), and an **Admission Controller** (a mutating webhook that rewrites requests/limits on new pods as they're created).

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata: { name: web-vpa }
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Auto"     # or: "Off" (recommend-only), "Initial" (apply only at pod creation), "Recreate"
```

```bash
kubectl describe vpa web-vpa       # shows Recommender's target/lowerBound/upperBound for cpu & memory
```

**Why VPA `Auto` mode is disruptive:** VPA can only change a running Pod's resource requests by **evicting and recreating it** (Kubernetes does not support in-place resource resizing for most resource types on most versions — in-place vertical scaling is a newer, still-maturing feature in recent Kubernetes versions). This means VPA in `Auto` mode causes real pod restarts, which is why it's commonly run in `Off` mode (recommendations only, surfaced via `kubectl describe vpa`, applied manually or via CI) for anything latency-sensitive.

## Why HPA and VPA rarely combine cleanly on the same metric

Running HPA and VPA **both targeting CPU** on the same workload creates a feedback loop: VPA raises the CPU request to reduce per-pod utilization percentage, which changes HPA's utilization calculation (since it's a percentage of request), which can suppress HPA's scale-up signal even under genuinely higher load — the two controllers can work against each other's assumptions. The generally recommended pattern is: **HPA on CPU/memory or custom metrics for horizontal scaling, VPA in `Off`/recommendation-only mode used periodically to right-size requests**, rather than both fighting live over the same signal in `Auto` mode simultaneously.

## Points to Remember

- HPA changes **replica count**; VPA changes **per-pod resource requests/limits** — orthogonal axes, not competing implementations of "autoscaling."
- HPA's CPU percentage is computed against `resources.requests.cpu` — a container with no CPU request means HPA has no baseline and shows `<unknown>`.
- `metrics-server` is a hard prerequisite for CPU/memory-based HPA; custom/external metrics need their own adapter (Prometheus Adapter, or KEDA, covered next file).
- VPA `Auto`/`Recreate` mode evicts and recreates pods to apply new resource values — it is inherently disruptive, which is why `Off` (recommend-only) is a common safer default.
- Combining HPA and VPA on the *same* metric (typically CPU) for the *same* workload risks a feedback loop; separate the concerns (HPA for scale-out, VPA for periodic right-sizing) instead.

## Common Mistakes

- Deploying an HPA on a Deployment with no `resources.requests.cpu` set anywhere, then being confused why `kubectl describe hpa` shows `<unknown>` for the current metric and no scaling ever happens.
- Setting `minReplicas` equal to `maxReplicas` "just to be safe," which defeats the entire purpose of HPA while still paying the small operational overhead of running one.
- Running VPA in `Auto` mode on a latency-sensitive, low-replica-count service and being surprised by periodic, seemingly random pod restarts — that's VPA's Updater evicting pods to apply new recommendations, working exactly as configured, just not as expected.
- Assuming HPA reacts instantly and precisely to load spikes — the default 15-second metrics-server scrape interval, plus `stabilizationWindowSeconds`, plus new pod startup/readiness time, all add real latency between "load increases" and "more capacity is actually serving traffic." Design for that lag (e.g., via `minReplicas` headroom or over-provisioning) rather than assuming HPA is instantaneous.
- Forgetting HPA needs `scaleTargetRef` to point at a workload controller (Deployment/StatefulSet/ReplicaSet), not a bare Pod — HPA cannot scale an object that has no replica-count field to adjust.
