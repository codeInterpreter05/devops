# Day 119 — Zero Trust & mTLS: Principles & the BeyondCorp Model

**Phase:** 4 – Advanced/Specialization | **Week:** W21 | **Domain:** Advanced Security | **Flag:** ⚡ Interview-critical

## Brief

Every security architecture before ~2010 assumed a hard perimeter: a firewall at the edge, a "trusted" internal network behind it, and a VPN to bridge the two. Once you were inside, you were trusted — which is exactly what made lateral movement after a single compromised laptop or leaked credential so devastating (this is literally how Google got breached in Operation Aurora, the event that birthed BeyondCorp). Zero trust inverts the model: **no request is trusted by virtue of where it came from — every request is authenticated and authorized on its own merits, continuously.** For DevOps/platform/security engineers this isn't academic: a Kubernetes cluster's pod network is a flat, high-trust internal network by default, which means "explain Zero Trust and how you'd implement it in a Kubernetes cluster" is one of the most common senior platform-security interview questions, because it forces you to connect network policy, identity, and encryption into one coherent design instead of naming buzzwords.

This day is split into three files:

1. **This file** — zero trust principles, the BeyondCorp model, and network micro-segmentation.
2. **[02-README-SPIFFE-SPIRE-Workload-Identity.md](02-README-SPIFFE-SPIRE-Workload-Identity.md)** — SPIFFE/SPIRE workload identity and cross-domain federation.
3. **[03-README-mTLS-Everywhere-Cert-Manager-Istio.md](03-README-mTLS-Everywhere-Cert-Manager-Istio.md)** — mTLS mechanics, cert-manager, and Istio's mesh-based mTLS.

## Zero trust: never trust, always verify

NIST SP 800-207 (the closest thing to an authoritative spec) distills zero trust into a handful of working principles:

1. **No implicit trust based on network location.** Being "inside the VPC" or "inside the cluster network" grants zero privilege by itself. Every access decision is made per-request.
2. **Every request is authenticated and authorized using identity + context** — workload identity (which service is calling), not IP address or subnet membership. IPs in Kubernetes are ephemeral and get reused across pod restarts within minutes; trusting them is trusting a fact that's true for a different workload an hour later.
3. **Least-privilege, dynamically enforced** — not "allow this subnet to reach that subnet," but "allow this specific workload identity to call this specific endpoint with this specific verb."
4. **Assume breach.** Design as if an attacker is already running code inside your network. The question isn't "how do we keep them out" alone — it's "if they get a shell in one pod, how far can they actually get."
5. **Continuous verification.** Trust isn't granted once at connection time and then forgotten — every request re-verifies identity, and observability/monitoring feeds back into access decisions.

### A concrete before/after

Take a compromised `frontend` pod (an attacker got RCE via a deserialization bug). In a flat-trust cluster with no NetworkPolicy, no mTLS, and RBAC that grants the frontend's ServiceAccount broad API access: the attacker can reach the `payments` database pod directly over the pod network (nothing stops the TCP connection), reach the Kubernetes API server using the mounted ServiceAccount token, and potentially reach cloud metadata endpoints for IAM credentials. In a zero-trust design: a default-deny NetworkPolicy means the frontend pod can't open a connection to `payments` at all unless an explicit allow rule exists; even if a connection is allowed at L3/L4, `payments` requires an mTLS handshake presenting a SPIFFE-verifiable certificate, and the frontend's workload identity was never registered to be issued that identity — so the TLS handshake itself fails; RBAC scoped to exactly what the frontend needs (`list`/`get` on a couple of ConfigMaps, nothing else) means even a fully-functional stolen token is nearly useless. None of these controls alone is bulletproof — the combination is the point.

## The BeyondCorp model

**BeyondCorp** is Google's own production implementation of zero-trust principles, published starting in 2014, built in direct response to Operation Aurora (2009), where attackers who got a foothold inside Google's corporate network were then treated as implicitly trusted. BeyondCorp's core move: **stop trusting the corporate VPN/network as a security boundary at all**, and instead evaluate every request to every internal application based on:

- **User identity** (SSO-verified)
- **Device identity and trust tier** — a managed, patched, encrypted corporate laptop with a valid device certificate scores differently from an unknown device, even for the same user
- **Context** — location, time, historical behavior patterns feeding a continuously-updated trust score

Concretely, this is implemented via:
- A **Device Inventory Service** tracking every known device and its security posture.
- A **Trust Inference** engine continuously scoring device/user trust (not a one-time login check).
- An **Access Control Engine** evaluating policy per-request.
- An **Access Proxy** in front of every internal application — nothing is reachable directly; every request, whether the employee is at a coffee shop or on the corporate LAN, goes through the same identity-aware proxy and gets the same scrutiny.

**Translating this to cloud-native infrastructure:** the direct analogues are identity-aware proxies (Google Identity-Aware Proxy, Pomerium, `oauth2-proxy` in front of internal dashboards) for user-to-service access, and **workload identity + mTLS** (SPIFFE/SPIRE, service mesh) for service-to-service access. The unifying idea in both cases: nothing is trusted because of *where it's coming from* (corporate network, cluster-internal IP) — only because of a *cryptographically verified identity* evaluated against policy, every single time.

**BeyondCorp vs. "Zero Trust" generally:** BeyondCorp is a specific, named, mostly user-to-application architecture. "Zero Trust" (per NIST 800-207) is the broader framework that also explicitly covers workload-to-workload traffic — which is where SPIFFE, mTLS, and service mesh authorization policies (covered in the next two files) come in. In an interview, naming BeyondCorp as "Google's applied zero trust architecture, primarily for user access to internal apps" and then pivoting to "the workload-to-workload equivalent inside a cluster is mTLS + workload identity + NetworkPolicy" demonstrates you understand both halves of the picture.

## Network micro-segmentation

Micro-segmentation means dividing what used to be one flat trusted network into many small, independently-controlled segments — ideally down to per-workload granularity — with explicit, enforced rules governing what can talk to what.

**The Kubernetes default you must know cold:** with **no NetworkPolicy objects applied**, every pod in a cluster can reach every other pod in every namespace on any port. This is the single most common CKS-exam and real-world misconfiguration: teams assume Kubernetes is "secure by default" on networking, and it explicitly is not — NetworkPolicy is opt-in, and it requires a CNI plugin that actually enforces it (**Calico, Cilium, Weave** — enforce; the basic `kubenet`/bridge-only setups on some minimal clusters **do not enforce policies at all**, silently accepting policy objects that do nothing).

A minimal default-deny baseline, applied per-namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # selects ALL pods in the namespace
  policyTypes: ["Ingress", "Egress"]
```

Then layer explicit allows on top, e.g. permitting `frontend` to reach `backend` on port 8080:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**L3/L4 segmentation vs. L7/identity segmentation — know the difference cold.** NetworkPolicy operates on IP addresses, ports, and label selectors resolved to IPs at a point in time. It's necessary but not sufficient for real zero trust, for one core reason: it trusts that "traffic from an IP that currently belongs to a pod labeled `app=frontend`" really is the frontend workload — but pod IPs are recycled constantly, and if an attacker can get a pod scheduled with the same label (e.g., via a compromised CI pipeline or an over-permissive RBAC grant on Pod creation), NetworkPolicy's identity check is trivially satisfied. **mTLS with SPIFFE-based identity (next two files) verifies cryptographic identity, not label/IP membership** — a compromised or spoofed pod cannot present a certificate it was never issued. This is why the strongest designs use both layers: NetworkPolicy reduces blast radius and attack surface cheaply at L3/L4, and mTLS + L7 authorization (Istio `AuthorizationPolicy`) provides the actual cryptographic guarantee — defense in depth, not redundancy.

## Points to Remember

- Zero trust = no implicit trust from network location; every request authenticated/authorized on identity + context, continuously, assuming breach has already happened somewhere.
- BeyondCorp is Google's specific user-to-application implementation of zero trust (device + user identity through an access proxy, no VPN-as-trust-boundary); it is not identical to the full NIST Zero Trust Architecture, which also covers workload-to-workload traffic.
- Kubernetes has **no network segmentation by default** — every pod can reach every pod cluster-wide until you apply NetworkPolicy, and NetworkPolicy only does anything if your CNI plugin enforces it (Calico/Cilium/Weave, not all CNIs).
- NetworkPolicy (L3/L4, IP/label-based) and mTLS/identity-based authorization (L7, cryptographic) are complementary layers, not substitutes — IPs and labels are spoofable/reassignable, cryptographic identity is not.
- "Assume breach" reframes the entire design question from "keep attackers out" to "limit what an attacker can do once they're already in one pod."

## Common Mistakes

- Assuming Kubernetes networking is "secure by default" and skipping NetworkPolicy entirely, or applying policies on a CNI plugin (e.g., default kind/bridge networking) that doesn't actually enforce them — the objects exist, nothing changes.
- Treating NetworkPolicy alone as "we've implemented zero trust" — it verifies IP/label membership, not cryptographic identity, and says nothing about encryption in transit.
- Confusing BeyondCorp (a specific Google architecture for user-to-app access) with the general Zero Trust Architecture concept in interviews — naming BeyondCorp correctly but describing NIST 800-207's workload-to-workload principles under that name signals confusion, not depth.
- Writing a default-deny-all NetworkPolicy and forgetting DNS: pods need explicit egress to CoreDNS/kube-dns on UDP/TCP 53, or every service in the namespace starts failing to resolve names, which often looks like "the network is broken" rather than an obvious DNS error.
- Believing "assume breach" means you shouldn't bother preventing breaches — it's an addition to prevention (patching, scanning, hardening), not a replacement for it.
