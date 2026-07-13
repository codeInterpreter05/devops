# Day 115 — Kubernetes Cost Optimisation: Kubecost & OpenCost

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** FinOps | **Flag:** —

## Brief

A Kubernetes cluster's AWS bill shows up as "EC2 instances" — it has no idea that those instances are shared by 40 namespaces belonging to 12 teams. Cloud-provider billing tools stop at the node boundary; **Kubecost** and its open-source core **OpenCost** exist specifically to allocate cost *inside* the cluster, down to namespace, deployment, pod, or even label/annotation level. This is the tool that makes "how much does the `checkout` team's workload actually cost us" an answerable question instead of a guess, and it's the direct follow-up to Day 114's cloud-level cost visibility, one layer deeper.

This day is split into three focused files:

1. **This file** — how Kubecost/OpenCost allocate shared cluster cost, and namespace-level chargeback.
2. **[02-README-Right-Sizing-And-Bin-Packing.md](02-README-Right-Sizing-And-Bin-Packing.md)** — right-sizing pod requests and cluster bin packing.
3. **[03-README-Spot-Nodes-With-Karpenter.md](03-README-Spot-Nodes-With-Karpenter.md)** — Karpenter and Spot nodes for K8s compute.

## The core problem: a cluster is one bill, many tenants

A single EKS/GKE node running at $0.50/hour might host 15 pods from 6 different namespaces, each with different CPU/memory *requests* (what's reserved) vs. *usage* (what's actually consumed). AWS's bill has no concept of any of that — it just sees "EC2 instance running." Kubecost/OpenCost sit inside the cluster, scrape metrics (via Prometheus), and reconstruct a per-workload cost breakdown by combining:

- **Node pricing** — pulled from the cloud provider's pricing API (or a static pricing sheet for on-prem), giving a $/hour cost per node based on its instance type, region, and purchasing option (On-Demand/Spot/RI-covered).
- **Resource requests and actual usage** — scraped from `kube-state-metrics` and cAdvisor/Prometheus, per pod, per container.
- **Allocation methodology** — cost is generally allocated proportionally by **resource requests** (not limits, not raw usage) when workloads are actively scheduled, because requests are what actually reserves capacity on the node regardless of whether the pod uses all of it. Idle/unused *requested* capacity still gets billed to whoever requested it — which is precisely the signal you want, because it's what should drive right-sizing conversations.

## OpenCost vs. Kubecost

**OpenCost** is the CNCF-sandbox open-source cost allocation *engine* — it's the core Kubecost donated to the community, providing the allocation API and a basic UI, with no paid tier gating. **Kubecost** is the commercial product built around that same core, adding: long-term data retention/storage (OpenCost's free tier keeps limited history), multi-cluster aggregation into a single view, savings recommendations (rightsizing, RI/Savings Plan suggestions specific to K8s), governance/budgeting alerts, and SSO/RBAC for a larger organization.

**Practical rule of thumb**: start with OpenCost (it's free, in-cluster, and CNCF-governed) to get real per-namespace numbers immediately. Reach for Kubecost's paid tier when you need multi-cluster rollups, long-term trend data beyond what your own Prometheus retention window can hold, or governance workflows (budget alerts, showback reports) that a platform team needs to hand to finance on a recurring basis.

```bash
# Install OpenCost via Helm (needs an existing Prometheus in-cluster or reachable)
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm install opencost opencost/opencost -n opencost --create-namespace \
  --set opencost.exporter.defaultClusterId=my-cluster \
  --set opencost.prometheus.external.enabled=true \
  --set opencost.prometheus.external.url=http://prometheus-server.monitoring.svc:9090

# Query allocation cost via the OpenCost API directly
curl "http://localhost:9003/allocation/compute?window=7d&aggregate=namespace" | jq
```

## Namespace-level cost allocation and chargeback

Namespace is the natural chargeback boundary in most orgs because it usually maps 1:1 to a team or product line already (a good namespace-per-team convention pays for itself here). Kubecost/OpenCost's `Allocation` API can group by namespace, label, annotation, controller, or any combination:

```bash
curl "http://localhost:9003/allocation/compute?window=30d&aggregate=namespace,label:team" | jq
```

This produces a report like:

| Namespace | Team label | CPU cost | Memory cost | Total | Efficiency |
|---|---|---|---|---|---|
| checkout | payments | $412 | $180 | $592 | 34% |
| search | search-infra | $890 | $310 | $1,200 | 71% |
| batch-etl | data-platform | $205 | $95 | $300 | 22% |

**Efficiency** here means `actual usage / requested resources` — the single most actionable number on this report. A namespace at 22% efficiency is requesting roughly 4.5x more CPU/memory than it uses, meaning ~78% of what's charged to that team is pure waste from over-requested pods, not real consumption.

**Shared/idle cost allocation**: cluster-level overhead that isn't attributable to any one namespace (kube-system, monitoring stack, unallocated idle node capacity) gets bucketed separately and is typically split either evenly across namespaces or proportionally by each namespace's share of allocated cost — Kubecost calls this "shared cost" and lets you configure the split strategy. This matters for chargeback fairness: a team running a tiny workload shouldn't be charged the same flat share of cluster overhead as a team running half the cluster.

## Points to Remember

- Cost is allocated primarily by **resource requests**, not limits or raw usage — because requests are what actually reserves node capacity, and over-requesting (not over-using) is what wastes money in a bin-packed cluster.
- OpenCost is the free, CNCF, in-cluster core; Kubecost is the commercial layer on top adding multi-cluster views, long retention, and governance features — start with OpenCost, upgrade only when you hit a specific gap.
- Namespace is the default, natural chargeback unit because it typically already maps to team/product ownership — but label/annotation-based aggregation lets you slice further (e.g., by environment or cost center) without changing your namespace layout.
- "Efficiency" (usage ÷ requests) is the single most useful number on a cost allocation report — low efficiency means the team is over-requesting, which is the #1 fixable cost driver in most clusters.
- Shared/idle cluster cost (kube-system, monitoring, unallocated capacity) must be explicitly split across tenants somehow — ignoring it understates the true cluster cost and makes any single namespace's report look artificially cheap.

## Common Mistakes

- Allocating cost by resource **limits** instead of **requests** — limits aren't reserved capacity, so this overstates cost for workloads with generous limits but modest requests and misrepresents what's actually driving node sizing.
- Treating Kubecost/OpenCost numbers as billing-grade precision without reconciling against the actual cloud invoice — allocation is an estimate based on scraped metrics and pricing-API data, useful for relative comparison and trend-spotting, but should be reconciled periodically against Cost Explorer/CUR for absolute accuracy.
- Ignoring the "shared cost" bucket entirely when reporting namespace costs to a team, making their number look artificially low and causing confusion when it doesn't reconcile with the total cluster bill.
- Standing up Kubecost without first ensuring Prometheus retention/scrape interval is adequate — cost data is only as good as the underlying metrics; a Prometheus with 15-day retention can't answer "what did this cost last quarter" without Kubecost's own longer-term storage backing it.
- Using namespace-per-environment (e.g., one giant `prod` namespace for everything) instead of namespace-per-team/service — this collapses the natural chargeback boundary and forces reliance on labels for every single query, which is more fragile than getting the namespace layout right from the start.
