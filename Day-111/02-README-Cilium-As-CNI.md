# Day 111 — Cilium & eBPF: Cilium as CNI, Network Policy & Service Mesh

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Service Mesh | **Flag:** ⚡

## Brief

Cilium is the reference implementation that turns eBPF's raw kernel programmability into an actual CNI plugin, network policy engine, load balancer (kube-proxy replacement), and — in "ambient"/sidecar-free mode — a service mesh data plane. Understanding what Cilium replaces (kube-proxy, and optionally the sidecar model itself) and how identity-based network policy works is the practical, hands-on core of today's activity.

## Cilium as a CNI: identity, not just IPs

Traditional CNI plugins (Calico's default mode, Flannel) largely think in terms of **IP addresses and IP-based `NetworkPolicy` rules**. Cilium introduces **identity-based security**: every pod gets a **security identity** derived from its Kubernetes labels (not its IP), and that identity — not the IP — is what policy is written and enforced against. This matters because pod IPs are ephemeral (a pod restarts, gets a new IP) but its labels, and therefore its identity and policy exposure, are stable.

Concretely, Cilium maintains an identity allocation mechanism (backed by its own key-value store, or CRDs like `CiliumIdentity` in modern deployments) mapping a set of security-relevant labels to a numeric identity — which is what actually gets checked against eBPF policy maps on the data path, a fast lookup rather than a string/IP comparison.

## Replacing kube-proxy

```bash
cilium install --set kubeProxyReplacement=true \
  --set k8sServiceHost=<API_SERVER_IP> \
  --set k8sServicePort=6443
```

When `kubeProxyReplacement` is enabled, Cilium's eBPF programs handle Service VIP translation, load balancing across endpoints, session affinity, and NodePort/LoadBalancer/ExternalIP handling directly in the kernel data path — `kube-proxy` can be removed from the cluster entirely (or left as a no-op). This is the literal hands-on activity for today: replace kube-proxy with Cilium and verify pod-to-pod flows still work (via Hubble, covered in the next file).

## CiliumNetworkPolicy — beyond Kubernetes NetworkPolicy

Kubernetes' native `NetworkPolicy` is L3/L4 only (IPs/ports) plus namespace/label selectors. Cilium's own CRD, `CiliumNetworkPolicy` (and cluster-wide `CiliumClusterwideNetworkPolicy`), adds **L7-aware policy** — because Cilium's eBPF/Envoy-based proxy can parse HTTP, gRPC, Kafka, and DNS:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-get-reviews
spec:
  endpointSelector:
    matchLabels:
      app: productpage
  egress:
  - toEndpoints:
    - matchLabels:
        app: reviews
    toPorts:
    - ports:
      - port: "9080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/reviews/.*"
```

This says: `productpage` may only make `GET /reviews/*` calls to `reviews` on port 9080 — anything else (a `POST`, a different path) is dropped at the network layer, without either service's application code knowing or caring. This is meaningfully more powerful than IP/port-only policy: it's enforcement at the *application protocol* level, done in a sidecar-less data path rather than requiring an app-level authorization library.

## Cilium as a service mesh: sidecar vs sidecar-less ("ambient")

Cilium can operate in two relevant modes for mesh-like functionality:

- **Cilium Service Mesh (sidecar-less)** — L7 visibility/policy and even Ingress/Gateway API support are implemented via **per-node** Envoy proxies (or eBPF alone for L3/L4) rather than a sidecar in every pod. This avoids the sidecar tax: no extra container per pod, no per-pod resource overhead, no sidecar injection webhook to manage, lower latency (no extra proxy hop inside the pod's own network namespace for L3/L4 traffic since eBPF intercepts inline).
- Pure eBPF datapath handles L3/L4 (connection routing, load balancing, basic policy) with **no proxy involved at all** — genuinely just kernel-level packet steering. Only L7-aware policy or mTLS-with-application-protocol-awareness needs an Envoy hop, and even then it's shared per-node, not per-pod.

This is the core of the "Cilium vs Istio: different layers" comparison (detailed in the next file): Cilium's baseline is a CNI plus an L3/L4/L7 policy engine that *can* offer mesh-like features without sidecars; Istio's baseline is a sidecar-per-pod proxy mesh that assumes an existing CNI underneath it.

## Bandwidth management with eBPF

Cilium implements **bandwidth management / traffic shaping** using eBPF's `tc` hooks combined with the kernel's **EDT (Earliest Departure Time)** rate-limiting model, configured per pod via an annotation:

```yaml
metadata:
  annotations:
    kubernetes.io/egress-bandwidth: "10M"
```

Rather than the older token-bucket-filter (`tbf`) qdisc approach (which adds queuing latency and is harder to tune per pod at scale), Cilium's eBPF-based approach calculates, per packet, the earliest time it's allowed to leave, directly in the kernel egress path — giving accurate per-pod bandwidth caps without a separate userspace shaping daemon or one qdisc per pod.

## Points to Remember

- Cilium's core security primitive is **identity** (derived from pod labels), not IP — this is what makes its network policy resilient to pod churn/restarts/IP reuse.
- `kubeProxyReplacement=true` moves Service load balancing (ClusterIP/NodePort/LoadBalancer) into eBPF entirely, letting you remove `kube-proxy` and its iptables rule maintenance overhead.
- `CiliumNetworkPolicy` extends plain Kubernetes `NetworkPolicy` with L7 awareness (HTTP methods/paths, gRPC, Kafka, DNS) — enforced without app code changes.
- Cilium's sidecar-less service mesh mode uses per-node Envoy only when L7 features are needed; pure L3/L4 traffic is handled by eBPF alone with no proxy hop.
- Bandwidth limiting uses eBPF plus EDT at the `tc` egress hook — more precise and lower-overhead than traditional `tc` qdisc shaping.

## Common Mistakes

- Enabling `kubeProxyReplacement` without removing/disabling the existing `kube-proxy` DaemonSet, leading to two systems fighting over Service routing (subtle, intermittent connectivity bugs).
- Writing `CiliumNetworkPolicy` L7 HTTP rules and forgetting they implicitly require default-deny semantics for that endpoint once *any* policy selects it — an endpoint with an L7-only allow rule and no matching L3/L4 rule for other traffic can unexpectedly drop previously-allowed connections.
- Assuming Cilium's sidecar-less mesh gives you Istio-identical features (e.g., full `VirtualService`-style weighted traffic splitting, fault injection) — feature parity is not 1:1; verify what your specific Cilium version supports (Gateway API support varies by version) before assuming an Istio migration playbook applies unchanged.
- Applying a bandwidth-limit annotation and expecting ingress shaping too — `egress-bandwidth` only shapes egress; ingress shaping has separate/less mature support and is often better handled upstream (e.g., at the LoadBalancer).
- Not checking `cilium status`/`cilium connectivity test` after install — silently running with a partially-healthy agent (BPF map pressure, a node stuck NotReady for Cilium-specific reasons) that manifests later as random connectivity flakiness.
