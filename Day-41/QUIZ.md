# Day 41 — Quiz: NetworkPolicies

Try to answer without looking at your notes. Answers are at the bottom.

1. What is Kubernetes' default pod-to-pod networking posture before any NetworkPolicy exists?
2. Does a NetworkPolicy support explicit "deny" rules? Explain how default-deny actually emerges.
3. What happens if two different NetworkPolicies both select the same pod for ingress — are their rules combined with AND or OR?
4. Why does applying a default-deny-all policy to a namespace typically break every application in it immediately, unless one more policy is added? What is that policy?
5. What's the difference in scope between a bare `podSelector` and a bare `namespaceSelector` used in a `from`/`to` rule?
6. If `namespaceSelector` and `podSelector` appear in the *same* list item under `from:`, how are they combined? What if they appear as *separate* list items?
7. In a 3-tier web → api → db isolation setup, why does restricting only `db`'s ingress not fully "isolate" the database?
8. What Kubernetes-provided label lets you target "all pods in namespace X" without adding any custom labels to that namespace?
9. Is it possible to create and apply a `NetworkPolicy` object on a cluster where it has zero actual effect on traffic? Why?
10. Name one CNI plugin that historically does not enforce NetworkPolicy on its own, and the common pairing used to add enforcement to it.
11. What capability does `CiliumNetworkPolicy` offer that the standard Kubernetes NetworkPolicy API cannot express at all?
12. **Interview question:** How do you prevent a compromised pod from scanning the rest of the cluster network?

---

## Answers

1. Fully open — any pod can reach any other pod, in any namespace, on any port, cluster-wide, with no restriction at all.
2. No, the base Kubernetes NetworkPolicy API is purely allow-based; there's no explicit deny rule type. Default-deny emerges implicitly: once at least one NetworkPolicy selects a pod for a given direction (ingress or egress), that direction becomes deny-by-default for that pod, and only traffic matching an allow rule from an applicable policy gets through.
3. OR — if a pod is selected by multiple applicable NetworkPolicies, traffic is allowed if it matches at least one of them; it does not need to satisfy every applicable policy's rules simultaneously.
4. Because default-deny-all also blocks DNS traffic (which is just UDP/TCP port 53 traffic to CoreDNS/kube-dns) — without an explicit allow rule for DNS egress, every pod loses the ability to resolve any hostname, breaking essentially all service-to-service communication that relies on DNS names rather than raw IPs. The fix is adding an explicit "allow egress to port 53" policy alongside the default-deny-all.
5. A bare `podSelector` only matches pods within the **same namespace** as the NetworkPolicy itself. A bare `namespaceSelector` matches **all pods** in any namespace whose labels match, regardless of those pods' own labels.
6. Same list item = AND (the traffic source must satisfy both conditions simultaneously — e.g., a specific pod label, and only within a specific namespace). Separate list items (each its own `-` entry) = OR (either condition independently is sufficient to allow the traffic) — this is one of the most consequential, easy-to-misconfigure details in NetworkPolicy YAML.
7. Restricting `db`'s ingress only controls what can connect **into** `db` — it does nothing to restrict what `db` itself can connect **out to** (egress). If `db` is compromised through some other vector, it could still make arbitrary outbound connections unless a separate egress-restricting policy is also applied to it; true isolation typically requires restricting both directions across all tiers, not just inbound to the most sensitive one.
8. `kubernetes.io/metadata.name` — automatically applied to every namespace (since Kubernetes 1.21+), set to the namespace's own name, letting you write `namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: <ns> } }` without needing to label the namespace yourself first.
9. Yes — `NetworkPolicy` is a Kubernetes API resource describing intent; actual enforcement (dropping non-matching packets) is delegated entirely to the cluster's CNI plugin. If the CNI plugin doesn't implement NetworkPolicy support, the API server will still accept and store the object with no errors, but nothing in the data plane actually enforces it.
10. Flannel, in its default/basic configuration, historically does not enforce NetworkPolicy on its own. It's commonly paired with Calico specifically for policy enforcement while Flannel continues to handle basic pod networking (the "Canal" combination).
11. Layer 7 (application-layer) policy — e.g., restricting traffic based on specific HTTP methods and paths, gRPC service names, or DNS names, even within an already-permitted TCP connection/port. Standard Kubernetes NetworkPolicy is limited to Layer 3/4 (IP/pod/namespace selectors and ports/protocols) and has no concept of application-layer content at all.
12. Strong answer: "First, confirm the cluster's CNI plugin actually enforces NetworkPolicy — Calico, Cilium, or Weave Net, not a bare default CNI like plain Flannel. Then apply a default-deny-all NetworkPolicy to relevant namespaces (`podSelector: {}` with both `Ingress` and `Egress` in `policyTypes`), immediately paired with an explicit DNS-egress allow rule so name resolution keeps working. On top of that baseline, add narrowly-scoped allow rules per legitimate traffic path — for example, a 3-tier app where `web` can reach `api`, `api` can reach `db`, but nothing else can reach `db` or `api` directly, forcing a compromised pod to pivot through each tier rather than reaching sensitive services directly. I'd also add an explicit exemption for the monitoring/observability stack to keep metrics scraping working. For anything requiring finer control — like restricting specific HTTP methods/paths, or blocking egress to the cloud metadata endpoint (169.254.169.254) to prevent SSRF-based credential theft — I'd reach for CNI-specific policy CRDs like Cilium's Layer 7 `CiliumNetworkPolicy` or Calico's `GlobalNetworkPolicy`, since the base Kubernetes NetworkPolicy API can't express those. The overall goal is that even if one pod is fully compromised, its ability to reach anything beyond its immediate legitimate dependencies is structurally blocked at the network layer, not just relying on application-level trust."
