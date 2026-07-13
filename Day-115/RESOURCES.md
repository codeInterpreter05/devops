# Day 115 — Resources: Kubernetes Cost Optimisation

## Primary (assigned)

- **Kubecost documentation** (docs.kubecost.com) — the assigned starting point; covers installation, the Allocation API, efficiency metrics, and savings recommendations in depth, and applies almost entirely to OpenCost too since it shares the same core.

## Deepen your understanding

- **OpenCost documentation** (opencost.io/docs) — the CNCF-governed free core; read this alongside Kubecost's docs to understand exactly where the free/paid line sits.
- **Karpenter documentation — NodePools & Disruption** (karpenter.sh/docs/concepts) — the canonical reference for NodePool/EC2NodeClass configuration and consolidation behavior.
- **Kubernetes docs — Vertical Pod Autoscaler** (github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) — the source-of-truth for VPA's update modes and recommendation fields.
- **"Kubernetes Cost Optimization" — Kubecost's own free guide/blog series** (kubecost.com/kubernetes-cost-optimization) — practical, tool-agnostic patterns for right-sizing and bin packing.

## Reference / lookup

- **Karpenter NodePool API reference** (karpenter.sh/docs/concepts/nodepools) — full field-by-field spec for requirements, disruption, and limits.
- **OpenCost Allocation API reference** (opencost.io/docs/api) — every query parameter for `/allocation/compute`.

## Practice

- **kind + kube-prometheus-stack + OpenCost on a laptop** — the entire Lab in this folder can be run locally with `kind create cluster` for the mechanics, though real savings numbers require a cluster with genuinely varied, running workloads (a shared staging cluster is ideal if you have access to one).
