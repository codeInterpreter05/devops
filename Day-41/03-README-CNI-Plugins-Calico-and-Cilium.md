# Day 41 — NetworkPolicies: CNI Plugins — Calico & Cilium

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Kubernetes

## Brief

Here's the fact that catches people off guard: **the Kubernetes API server will happily accept and store a `NetworkPolicy` object even if absolutely nothing in the cluster enforces it.** `NetworkPolicy` is just a Kubernetes API resource — a piece of intent. Actually *enforcing* that intent (dropping packets that violate it) is the job of the **CNI (Container Network Interface) plugin**, and not every CNI plugin implements NetworkPolicy enforcement. This is one of the most consequential, least-obvious facts in this whole topic, and exactly why "what CNI plugin do you run, and does it enforce NetworkPolicy" is a question worth asking before trusting any NetworkPolicy you've written actually does anything.

## The CNI's role — NetworkPolicy is a spec, not an implementation

Kubernetes defines the `NetworkPolicy` API and its semantics (file 1/2), but delegates actual packet-level enforcement entirely to whichever CNI plugin the cluster uses. Some CNI plugins provide pod networking but **do not implement NetworkPolicy at all** — in that case, creating NetworkPolicy objects has zero effect on actual traffic; the API accepts them, `kubectl get networkpolicy` shows them, and none of it is enforced. This has caused real production incidents where teams believed they had network segmentation in place (because the YAML existed and applied cleanly) when the underlying CNI was silently ignoring it entirely.

Common CNI plugins and their NetworkPolicy support:
- **Calico** — full NetworkPolicy support, plus its own richer `GlobalNetworkPolicy`/`NetworkPolicy` CRDs with additional features (e.g., policy ordering/priority, deny rules, DNS-aware policies) beyond the base Kubernetes API.
- **Cilium** — full NetworkPolicy support, built on eBPF instead of iptables, plus its own `CiliumNetworkPolicy` CRD supporting **Layer 7** (HTTP method/path, gRPC service, DNS name) rules — well beyond Kubernetes' native L3/L4-only NetworkPolicy spec.
- **Weave Net** — supports NetworkPolicy enforcement.
- **Flannel** (in its default/basic configuration) — provides pod networking (overlay) but historically **does not enforce NetworkPolicy** on its own; it's commonly paired with Calico specifically for policy enforcement (the "Canal" combination: Flannel for networking, Calico's policy engine layered on top).
- Managed cloud offerings (EKS, GKE, AKS) each have their own defaults and options — e.g., EKS defaults to the AWS VPC CNI, which historically required enabling Calico (or Cilium) as an add-on specifically to get NetworkPolicy enforcement; GKE's Dataplane V2 is Cilium-based and enforces NetworkPolicy natively.

**The practical takeaway: always verify which CNI plugin a cluster runs and confirm it actually enforces `NetworkPolicy` before relying on it as a real security boundary** — `kubectl get pods -n kube-system` (looking for `calico-node`, `cilium`, etc.) or checking the cluster provisioning config (Terraform/eksctl/cluster.yaml) is how you confirm this in practice.

## Calico

Calico can operate purely as a NetworkPolicy-enforcing layer (paired with another CNI for basic networking) or as a full CNI+policy solution. Beyond the base Kubernetes NetworkPolicy API, Calico's own CRDs add capabilities the standard API lacks:

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny-egress-except-dns
spec:
  order: 100
  selector: all()
  types:
    - Egress
  egress:
    - action: Allow
      protocol: UDP
      destination:
        ports: [53]
```

Notable Calico-specific features: explicit `order`/priority across multiple policies (Kubernetes-native NetworkPolicy has no priority concept — all applicable policies are just OR'd together), explicit `Deny` actions (not just implicit deny-by-omission), and cluster-wide (`GlobalNetworkPolicy`, not namespace-scoped) policies — useful for enforcing an organization-wide baseline (like "nothing may reach the metadata service IP `169.254.169.254`except pods explicitly labeled to need it," a common defense against SSRF-based cloud credential theft).

## Cilium — eBPF-based, with Layer 7 awareness

Cilium implements networking and policy enforcement using **eBPF** (running verified, sandboxed programs directly in the Linux kernel) rather than traditional iptables rule chains — meaningfully more performant at scale, since eBPF hooks operate at a lower level than iptables' sequential rule-matching, which can become a real bottleneck with thousands of rules across a large cluster.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-allow-get-only
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: web
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/api/v1/.*"
```

This is a fundamentally different capability class from standard Kubernetes NetworkPolicy: standard NetworkPolicy can only say "allow TCP port 8080 from `app: web`" — it has **no concept of HTTP methods, paths, or application-layer content at all**. `CiliumNetworkPolicy` can additionally restrict *which HTTP methods/paths* are allowed even within an already-permitted TCP connection — e.g., allowing `GET /api/v1/*` while blocking `POST`/`DELETE` on the same port from the same source, something plain L3/L4 NetworkPolicy structurally cannot express.

## Points to Remember

- `NetworkPolicy` is a Kubernetes API object describing intent; the CNI plugin is what actually enforces it at the packet level — a cluster can accept and store NetworkPolicy YAML while enforcing none of it, if the CNI doesn't implement policy support.
- Always confirm the cluster's CNI plugin actually supports NetworkPolicy enforcement (Calico, Cilium, Weave Net do; plain/default Flannel historically does not) before treating written policies as a real security boundary.
- Calico adds cluster-wide `GlobalNetworkPolicy`, explicit priority/ordering, and explicit deny actions beyond what the base Kubernetes NetworkPolicy API supports.
- Cilium is eBPF-based (kernel-level enforcement, generally more performant at scale than iptables) and its `CiliumNetworkPolicy` CRD supports Layer 7 rules (HTTP method/path, gRPC, DNS name) — capabilities entirely outside what standard L3/L4 NetworkPolicy can express.
- Managed Kubernetes offerings vary in default CNI and NetworkPolicy support out of the box (e.g., EKS's default VPC CNI historically needed Calico/Cilium added specifically for enforcement; GKE Dataplane V2 is Cilium-based and enforces it natively) — check, don't assume.

## Common Mistakes

- Writing and applying NetworkPolicy YAML, seeing it apply cleanly with no errors, and concluding the cluster is now segmented — without ever confirming the CNI plugin actually enforces NetworkPolicy at all, leading to a false sense of security.
- Assuming standard Kubernetes NetworkPolicy can express Layer 7 rules (specific HTTP paths/methods, gRPC service names) — it fundamentally cannot; that requires a CNI-specific CRD like `CiliumNetworkPolicy`.
- Not accounting for CNI-specific priority/ordering (Calico) when multiple overlapping policies exist — assuming Kubernetes-native OR-only combination logic applies uniformly, when a Calico `GlobalNetworkPolicy` with an explicit `Deny` and lower `order` value can override what a namespace-scoped allow policy would otherwise permit.
- Choosing a CNI plugin purely for its networking/performance characteristics without checking its NetworkPolicy (and Layer 7 policy, if needed) support, then discovering the gap only after security requirements surface later in the project.
- Forgetting that switching CNI plugins on a live cluster (e.g., migrating from Flannel to Calico for policy enforcement) is a nontrivial, often disruptive operation — not something to plan for "later" once workloads and existing NetworkPolicy assumptions are already deeply baked into the cluster.
