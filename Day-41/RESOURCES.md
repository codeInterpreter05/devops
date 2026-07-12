# Day 41 — Resources: NetworkPolicies

## Primary (assigned)

- **Kubernetes Network Policy Recipes** (github.com/ahmetb/kubernetes-network-policy-recipes) — the assigned starting point. A curated set of ready-to-apply NetworkPolicy YAML examples covering nearly every common pattern, including the default-deny, cross-namespace, and monitoring-exemption patterns used in today's notes.

## Deepen your understanding

- **Kubernetes official docs — Network Policies** (kubernetes.io/docs/concepts/services-networking/network-policies) — the authoritative spec for `podSelector`/`namespaceSelector` combination semantics and `policyTypes` behavior.
- **Calico documentation — Network Policy** (docs.tigera.io) — full reference for `GlobalNetworkPolicy`, policy ordering/priority, and explicit deny actions beyond the base Kubernetes API.
- **Cilium documentation — Network Policy** (docs.cilium.io) — full reference for `CiliumNetworkPolicy`, including Layer 7 HTTP/gRPC/DNS-aware rules and the eBPF enforcement model.
- **"Cilium vs Calico" comparison write-ups** (from either project's blog, or independent practitioner comparisons) — useful for understanding the practical trade-offs (performance model, feature depth, operational maturity) when choosing a CNI for a new cluster.

## Reference / lookup

- **`kubectl get pods -n kube-system`** — the fastest on-cluster way to identify which CNI plugin is actually running, before trusting any NetworkPolicy.
- **Kubernetes docs — "Declare Network Policy" walkthrough** — a guided, runnable tutorial using `kind`/Calico, good as a sanity check against today's lab steps.

## Practice

- **Complete today's lab end to end** on a real `kind` + Calico cluster — the assigned hands-on activity (zero-trust 3-tier app with a monitoring exemption) is exactly what most interview "design network segmentation" questions expect you to have hands-on experience with.
- Try swapping Calico for Cilium in the lab and reproduce the same policies as `CiliumNetworkPolicy`, then add one Layer 7 rule (e.g., restrict `web -> api` to `GET` only) to directly experience the capability gap between the two policy models.
