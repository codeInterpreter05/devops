# Day 45 — K8s Advanced Scheduling: Pod Affinity/Anti-Affinity & Topology Spread Constraints

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** Kubernetes | **Flag:** –

## Brief

Node affinity (previous file) controls placement relative to *node properties*. Pod affinity/anti-affinity and topology spread constraints control placement relative to *other pods* — which is what actually determines whether your replicas survive an AZ outage, whether latency-sensitive pairs land close together, and whether one node ends up hosting 8 replicas of the same deployment while another sits empty. This is the mechanism behind today's hands-on activity: spreading replicas across AZs for resilience.

## Pod affinity and anti-affinity

Same two-strength model as node affinity (`required`/`preferred`, both `...DuringSchedulingIgnoredDuringExecution`), but the match target is **other pods' labels**, scoped by a `topologyKey`.

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: ["web-frontend"]
      topologyKey: "topology.kubernetes.io/zone"
```

**`topologyKey` is the crucial concept** — it defines what "together" or "apart" means. It's just a node label; the scheduler groups nodes by the value of that label and applies the affinity/anti-affinity rule across those groups, not literally node-by-node.
- `topologyKey: kubernetes.io/hostname` → anti-affinity means "not on the same physical node."
- `topologyKey: topology.kubernetes.io/zone` → anti-affinity means "not in the same AZ" — this is the one you want for the classic "spread replicas across AZs for resilience" requirement.

**Pod anti-affinity (`required`) is how you guarantee replica spread across failure domains**: "don't schedule this pod on a node that's in the same zone as another pod matching `app=web-frontend`" forces each replica into a distinct zone. With `required`, if you have more replicas than zones, extra replicas will stay `Pending` — this is a real tradeoff (guaranteed spread vs. potential unschedulable pods) that's exactly why topology spread constraints (below) were introduced as a more nuanced tool.

**Pod affinity** (attraction) is used less often for resilience and more for **colocation** — e.g., placing a cache sidecar-adjacent service near the pods that call it most, or ensuring a latency-sensitive pair lands in the same zone to minimize cross-AZ network hops (and cross-AZ data transfer cost, which is non-trivial at scale).

**Cost of pod (anti-)affinity**: it's computationally expensive relative to node affinity — the scheduler must examine other pods across the cluster, not just static node labels — and this scales poorly on very large clusters with many affinity rules. Kubernetes docs explicitly warn against overusing it in clusters with hundreds of nodes.

## Topology Spread Constraints — the modern, more flexible tool

Topology Spread Constraints (`topologySpreadConstraints` on the pod spec) were built specifically to solve "spread pods evenly across a topology domain" more gracefully than pod anti-affinity's binary required/preferred model.

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: web-frontend
```

- **`maxSkew`** — the maximum allowed difference between the topology domain with the most matching pods and the one with the fewest. `maxSkew: 1` across 3 zones with 6 replicas means each zone gets exactly 2 (perfectly even); it wouldn't allow one zone to have 4 while another has 2.
- **`whenUnsatisfiable`** — `DoNotSchedule` (hard — like `required`, pod stays Pending if the constraint can't be met) or `ScheduleAnyway` (soft — like `preferred`, the scheduler still tries to minimize skew but won't block scheduling).
- **`topologyKey`** — same concept as pod affinity's, but here it's about spreading matching pods evenly across the label's distinct values, not just a binary "same/different."
- You can (and often should) define **multiple constraints simultaneously** — e.g., spread across zones AND across hostnames — so replicas are both AZ-resilient and not overly concentrated on individual nodes within a zone.

**Topology spread vs. pod anti-affinity — which to use:**
- Anti-affinity answers a *binary* question well: "never 2 of these on the same node/zone." Good for strict exclusivity (e.g., never colocate two StatefulSet leader pods).
- Topology spread answers a *proportional/even-distribution* question: "spread N replicas roughly evenly across zones," which is what you actually want for HA replica placement, and it degrades more gracefully — a `maxSkew` violation due to one zone being temporarily full doesn't necessarily block scheduling elsewhere the way a strict anti-affinity rule might.
- Modern guidance (and this day's hands-on activity, "configure anti-affinity to spread replicas across AZs") often really means topology spread constraints in practice — many engineers use "anti-affinity" colloquially to describe the *goal*, but implement it with the more flexible topology spread mechanism, especially for replica counts that don't divide evenly across the number of zones.

## Points to Remember

- `topologyKey` defines what "together"/"apart" means for pod affinity/anti-affinity — it's a node label, and the scheduler groups nodes by its value, not literal per-node comparisons.
- Pod anti-affinity with `required` gives a hard guarantee (e.g., never 2 replicas in the same zone) but can leave excess replicas `Pending` if replica count exceeds the number of topology domains.
- Topology Spread Constraints (`maxSkew` + `whenUnsatisfiable`) give proportional, more forgiving control over distribution and usually better fit "spread replicas across AZs" than binary anti-affinity.
- Pod (anti-)affinity is more expensive for the scheduler to evaluate than node affinity because it requires inspecting other pods cluster-wide — avoid overusing it on very large clusters.
- You can combine multiple topology spread constraints (e.g., zone AND hostname) on the same pod spec for layered resilience guarantees.

## Common Mistakes

- Using `podAntiAffinity` with `required` for AZ spread when replica count doesn't evenly divide across available zones, causing pods to get stuck `Pending` instead of gracefully accepting a slightly uneven distribution — topology spread constraints with `ScheduleAnyway` handle this far more gracefully.
- Forgetting `topologyKey` and assuming anti-affinity means "different physical node" by default — it depends entirely on which label you choose; using `kubernetes.io/hostname` vs. `topology.kubernetes.io/zone` produces very different placement guarantees.
- Setting `maxSkew: 1` with `whenUnsatisfiable: DoNotSchedule` without understanding that this can block scheduling entirely once a zone is capacity-constrained — appropriate for strict SLAs, surprising if you expected it to behave like a soft preference.
- Heavy use of `required` pod affinity/anti-affinity rules across a large cluster, causing scheduler latency to degrade — Kubernetes explicitly documents this as a scaling concern.
- Assuming pod anti-affinity re-balances already-running pods when node/pod topology changes later — like node affinity, it's only evaluated at scheduling time (`IgnoredDuringExecution`), so already-placed pods don't move themselves in response to drift.
