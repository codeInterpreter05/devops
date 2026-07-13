# Day 115 — Kubernetes Cost Optimisation: Right-Sizing & Bin Packing

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** FinOps | **Flag:** —

## Brief

Once Kubecost/OpenCost tell you *which* namespaces are wasteful (low efficiency), the fix is almost always one of two mechanical problems: pods requesting far more than they use (right-sizing), or the scheduler being unable to pack pods tightly enough onto nodes because of how those requests are set (bin packing). These two problems compound each other — over-requested pods make bin packing worse, and bad bin packing makes right-sizing individual pods look less impactful than it actually is at the cluster level.

## Right-sizing: requests vs. actual usage

Every pod's `resources.requests` is a promise to the scheduler: "reserve this much CPU/memory for me, guaranteed." The **Vertical Pod Autoscaler (VPA)** and Kubecost's own recommendations both work by comparing that promise against **actual observed usage** over a lookback window, then recommending a request value closer to reality.

**Why over-requesting happens constantly in practice:**
- Engineers copy a `resources` block from another service "to be safe" without profiling their own workload.
- Requests are set once at initial deployment and never revisited as the workload's real traffic/behavior stabilizes.
- Fear of OOMKills or CPU throttling leads to generous padding that's never walked back down.

**VPA in recommendation mode (the safe way to start):**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: checkout-api-vpa
  namespace: checkout
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: checkout-api
  updatePolicy:
    updateMode: "Off"   # "Off" = recommendations only, does not touch running pods
```

```bash
kubectl describe vpa checkout-api-vpa -n checkout
# Look at status.recommendation.containerRecommendations:
#   target: what VPA thinks the request SHOULD be
#   lowerBound / upperBound: the safe range given usage variance
```

`updateMode: "Off"` is the correct starting point in any real environment — it never evicts or resizes a running pod, it just tells you what the number *should* be so a human (or a controlled rollout) applies it deliberately. `"Auto"` mode will actually evict and recreate pods to apply new requests, which is disruptive for anything without multiple replicas and a PodDisruptionBudget, and is generally avoided for stateful or singleton workloads.

**Kubecost's request-sizing recommendations** work similarly but present the answer in dollar terms directly — "reducing this deployment's memory request from 4Gi to 1.5Gi based on 30-day p99 usage saves $340/month" — which is usually a more persuasive artifact for getting a team to actually act than a raw VPA YAML recommendation.

**A practical right-sizing workflow:**
1. Deploy VPA in `Off` mode across a namespace, let it observe for at least 1-2 full traffic cycles (7-14 days minimum, longer if there's weekly seasonality).
2. Cross-reference VPA's recommendation with Kubecost's per-workload efficiency number — they should agree; if they don't, investigate the metrics pipeline before trusting either.
3. Apply new requests during a low-risk deployment window, watch for OOMKills/throttling in the following days, and set the new request at the recommended value **plus a deliberate buffer** (VPA's `upperBound`, not `target`, is often the safer number to actually ship) rather than the bare minimum observed.

## Cluster bin packing

Even with perfectly right-sized pods, a cluster wastes money if the scheduler can't pack pods tightly enough onto nodes — e.g., 20 nodes each running at 40% utilization because pod sizes don't divide evenly into node capacity, when the same workload could fit on 10 nodes at 80% utilization.

**What drives poor bin packing:**
- **Node group instance size mismatch** — large pods on small-instance node groups waste the "remainder" capacity on every node; e.g., a 3-vCPU-request pod on 4-vCPU nodes leaves 1 vCPU stranded per node, unusable by anything bigger than 1 vCPU.
- **Topology spread constraints and pod anti-affinity** — necessary for availability, but every spread/anti-affinity rule reduces packing density by design (that's the point — don't "fix" this by removing HA guarantees).
- **Daemonsets** — every node pays their overhead (log shippers, CNI, service mesh sidecar injectors) regardless of how many workload pods are on it, so fewer, bigger nodes amortize daemonset overhead across more workload capacity than many small nodes.
- **The default/least-utilized scheduler behavior without a packing-aware autoscaler** — the default Cluster Autoscaler scales node groups reactively per pending pod, not with an eye toward global packing efficiency the way Karpenter's consolidation logic does (see the next file).

**Concrete levers:**
- Prefer **fewer, larger nodes** over many small nodes where HA/spread requirements allow — this amortizes both daemonset overhead and the per-node "remainder" waste.
- Use `kubectl top nodes` / Kubecost's cluster efficiency view to spot chronically underutilized nodes, and check whether Cluster Autoscaler / Karpenter consolidation is actually running (it can be disabled or set too conservatively).
- Watch for **pod disruption budgets set too strictly** (e.g., `maxUnavailable: 0` on every deployment) — this can block the autoscaler from safely consolidating/draining nodes even when there's genuine idle capacity to reclaim, because it can never find a "safe" node to empty.

## Points to Remember

- VPA's `updateMode: "Off"` gives you a safe, non-disruptive recommendation — use it as the default starting mode; `"Auto"` actively evicts pods and needs PodDisruptionBudgets and multiple replicas to be safe.
- Right-sizing based on `target` recommendation alone can be too aggressive — `upperBound` (or `target` + explicit buffer) is usually the safer number to actually ship to avoid trading a cost win for a reliability regression.
- Bin-packing waste is a *node-shape* problem, not just a *pod-size* problem — fewer, larger nodes generally pack better and amortize daemonset overhead better than many small nodes, subject to HA/spread constraints.
- Overly strict PodDisruptionBudgets can silently prevent the autoscaler from consolidating idle nodes, so a cluster can look "unable to scale down" for reasons that have nothing to do with the autoscaler's own logic.
- Efficiency (usage/requests) and bin-packing density (used-node-capacity/total-node-capacity) are two separate levers — fixing one without the other leaves real savings on the table.

## Common Mistakes

- Switching VPA straight to `Auto` mode on a stateful or single-replica workload, causing avoidable disruption the first time it decides to resize.
- Right-sizing to the exact p50 or average observed usage instead of a higher percentile (p95/p99) or VPA's `upperBound`, then getting hit with OOMKills the next time traffic spikes above the (now too-tight) request.
- Running many small worker nodes "for blast-radius reasons" without weighing the bin-packing and daemonset-overhead cost of that choice — blast radius is a legitimate concern, but it's a tradeoff to make deliberately, not a default.
- Setting `maxUnavailable: 0` on every PodDisruptionBudget by habit, then being confused why the cluster autoscaler never scales down even during genuinely idle periods.
- Comparing raw node CPU/memory utilization (`kubectl top nodes`) to conclude "the cluster is fine" while ignoring per-pod request-vs-usage efficiency — a cluster can look highly utilized in aggregate while still being full of individually over-requested pods that happen to sum up close to capacity.
