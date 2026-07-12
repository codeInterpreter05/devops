# Day 24 — Networking: Kubernetes Services

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Pods are ephemeral and their IPs change every time they're recreated — you can never point a client directly at a pod IP and expect it to keep working. A `Service` solves this by giving a stable virtual address in front of a dynamic, changing set of pods. Every networking question that follows in Kubernetes (Ingress, DNS, NetworkPolicy, service mesh) is built on top of this one abstraction, so getting the four Service types and what each is actually for cold is foundational, not optional.

This day is split into three files:

1. **This file** — Service types: `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`.
2. **[02-README-Ingress-Controllers.md](02-README-Ingress-Controllers.md)** — Ingress vs Ingress Controller, Nginx Ingress, TLS termination with cert-manager.
3. **[03-README-CoreDNS-Internals.md](03-README-CoreDNS-Internals.md)** — CoreDNS internals and how service discovery actually resolves.

## Why Services exist: the pod-IP-churn problem

Every pod restart, reschedule, or scale event gives you brand-new pod IPs. Hardcoding IPs (or even relying on `kubectl get pods -o wide` output) in client config would break constantly. A Service is a stable object with:
- A **stable virtual IP** (for `ClusterIP`/`NodePort`/`LoadBalancer` types) or a stable DNS name that resolves *through* to real backends.
- A **label selector** that determines its backend Pod set dynamically — the Service never "knows" about specific pods by name, it just continuously watches for pods matching its selector.
- An **Endpoints/EndpointSlice** object, maintained by the Endpoint controller, that's the actual living list of `podIP:port` pairs currently backing the Service.

```bash
kubectl get svc myapp -o wide
kubectl get endpoints myapp             # the real, current backend pod IPs
kubectl get endpointslices -l kubernetes.io/service-name=myapp   # scalable modern equivalent
```

## `ClusterIP` — internal-only, the default

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP          # default; can be omitted
  selector:
    app: myapp
  ports:
    - port: 80              # the Service's own port
      targetPort: 8080       # the container port to forward to
```

Gets a virtual IP from the cluster's Service CIDR, reachable **only from inside the cluster**. This is the right default for anything that's only ever called by other pods (an internal API, a database, a cache) — it should never be directly internet-facing.

## `NodePort` — exposes a port on every node's IP

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080        # optional; auto-assigned from 30000-32767 range if omitted
```

Every node in the cluster opens `nodePort` and forwards traffic to the Service (via kube-proxy), regardless of whether a backend pod is actually running on that specific node. This is why `NodePort` is rarely used directly in production for user-facing traffic — you'd need to know/manage individual node IPs, there's no built-in health-aware load balancing across nodes, and it's a building block `LoadBalancer` uses internally, not usually an end-state. It's genuinely useful for quick local testing (`minikube service`) or for exposing something behind your own externally-managed load balancer that targets node IPs directly.

## `LoadBalancer` — provisions a real cloud load balancer

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 8443
```

Triggers the **cloud-controller-manager** to call the cloud provider's API and provision an actual external load balancer (an AWS ELB/NLB, GCP Load Balancer, Azure LB) that targets the Service's `NodePort` on every node behind the scenes. This is why creating a `LoadBalancer` Service costs real money on every cloud — each one is a real, billed cloud resource, not a Kubernetes-internal construct. Annotations control provider-specific behavior:

```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"          # Network LB vs Classic
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"       # internal-only, no public IP
```

**Creating one `LoadBalancer` Service per microservice does not scale cost-wise** — this is precisely the problem Ingress (next file) solves: one shared load balancer routing to many Services by hostname/path, instead of one expensive cloud LB per Service.

## `ExternalName` — a DNS-level alias, not a proxy

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: prod-db.us-east-1.rds.amazonaws.com
```

This type is fundamentally different from the other three — it has **no selector, no virtual IP, no Endpoints object at all**. It's purely a CNAME-style DNS alias: CoreDNS answers `external-db.default.svc.cluster.local` with a `CNAME` pointing at `prod-db.us-east-1.rds.amazonaws.com`, and the client's own DNS resolver/TCP stack does everything else. It's useful for giving an in-cluster-style name to something external (an RDS endpoint, a legacy on-prem service) so app config can uniformly use `<name>.svc.cluster.local` regardless of whether the backend is in-cluster or not — but it does **not** get you load balancing, health checks, TLS termination, or any of the things a real Service provides; it's just a name.

## Points to Remember

- A Service's backend set is dynamic and selector-driven; the real list of ready backend IPs lives in Endpoints/EndpointSlice, maintained continuously by a controller, not the Service object itself.
- `ClusterIP` (internal only) is the default and correct choice for anything not meant to be directly internet-facing.
- `NodePort` opens the same port on every node regardless of where pods actually run, and is mostly a building block for `LoadBalancer`, not a production end-state for user traffic.
- `LoadBalancer` provisions a real, billed cloud resource per Service — fine for a handful of entry points, expensive and unwieldy as your one-LB-per-microservice count grows, which is exactly why Ingress exists.
- `ExternalName` is DNS-only — no proxying, no selector, no load balancing — just a CNAME alias for uniform naming.

## Common Mistakes

- Creating a `LoadBalancer` Service for every internal microservice "to be safe," racking up cloud LB costs and public attack surface for services that should have been `ClusterIP`.
- Assuming `NodePort` load-balances intelligently across nodes with the pod actually running — traffic can land on a node with no local backend pod, and kube-proxy then forwards it to another node's pod, adding a network hop (this "double hop" is avoidable with `externalTrafficPolicy: Local`, which only routes to node-local pods but can create imbalance if pods aren't evenly spread).
- Expecting an `ExternalName` Service to provide load balancing or TLS termination — it's a bare DNS alias; a client connecting through it talks directly to the external endpoint with no Kubernetes-level mediation at all.
- Debugging "Service isn't routing to my pod" without first checking `kubectl get endpoints` — if Endpoints is empty, the Service's label `selector` doesn't match any pod's labels (a typo is the overwhelmingly common cause), and no amount of kube-proxy/iptables debugging will help until the selector is fixed.
- Forgetting that Service `port` and `targetPort` are independent — sending traffic to the wrong `targetPort` (not matching the container's actual listening port) causes connection resets that look like a networking problem but are actually a simple config mismatch.
