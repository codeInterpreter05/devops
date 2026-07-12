# Day 41 — NetworkPolicies: Fundamentals & Default-Deny

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Kubernetes

## Brief

By default, Kubernetes networking is **fully open**: any pod in a cluster can send traffic to any other pod, in any namespace, on any port — flat, unrestricted, "trust everyone" networking. That's convenient for getting a demo running and catastrophic for production security: a single compromised pod (a vulnerable dependency, a supply-chain-compromised container image, an exploited application) can freely scan and reach every database, internal API, and admin endpoint in the entire cluster. **NetworkPolicies** are Kubernetes' native mechanism for restricting pod-to-pod traffic — the direct, cluster-native answer to "how do you prevent a compromised pod from moving laterally through the network," today's flagged interview question equivalent for this domain.

This day is split into three files:

1. **This file** — the default-open problem, default-deny, and the basic ingress/egress model.
2. **[02-README-Selectors-and-Practical-Patterns.md](02-README-Selectors-and-Practical-Patterns.md)** — namespace/pod selectors and real-world patterns (database isolation, monitoring access).
3. **[03-README-CNI-Plugins-Calico-and-Cilium.md](03-README-CNI-Plugins-Calico-and-Cilium.md)** — why NetworkPolicy enforcement depends entirely on your CNI plugin, and Calico/Cilium specifics.

## Why default-open networking is dangerous — and NetworkPolicy's core model

Every `Pod` gets an IP; by default, every pod can reach every other pod's IP on every port, cluster-wide, with no notion of "this pod shouldn't be allowed to talk to that one." NetworkPolicy objects layer **allow rules** on top of this — critically, NetworkPolicies are **purely additive/allow-based**, never explicit "deny" rules. The mental model:

- **No NetworkPolicy selects a pod** → that pod's traffic (both directions) is fully unrestricted, exactly like the cluster default.
- **At least one NetworkPolicy selects a pod, for a given direction (ingress or egress)** → that direction becomes **default-deny** for that pod, and only traffic matching **at least one** of the applicable policies' rules is allowed. Multiple policies selecting the same pod are combined with **OR** logic — traffic is allowed if it matches any one applicable policy, not only if it matches all of them.

This "whitelist activates a deny-by-default posture, only for the selected direction" behavior is the single most important, most frequently misunderstood mechanic in the entire topic — a NetworkPolicy with an empty `ingress: []` (no rules at all) on a pod selector means **nothing** is allowed in for those pods, while simply *not creating any NetworkPolicy* for a pod leaves it fully open.

## Default-deny-all — the standard starting posture

The recommended baseline for any namespace with real security requirements is an explicit **default-deny-all** policy, then layering specific allow rules on top:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # selects ALL pods in this namespace
  policyTypes:
    - Ingress
    - Egress
```

`podSelector: {}` (empty selector) matches **every pod** in the namespace. With no `ingress`/`egress` rules block present at all (as above), and `policyTypes` listing both directions, this policy makes every pod in `production` deny **all** inbound and **all** outbound traffic — including to/from other pods in the same namespace, and critically, including **DNS** (which is just UDP/TCP traffic to `kube-dns`/`CoreDNS`, subject to the same default-deny once egress is restricted). This is exactly why default-deny-all is applied *first*, then specific allows (including DNS egress) are added on top — deploying default-deny without also allowing DNS egress breaks essentially every application in the namespace immediately, since they can no longer resolve any hostname.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

## Basic ingress and egress rule shape

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web
      ports:
        - protocol: TCP
          port: 8080
```

Reading this precisely: this policy applies to pods labeled `app: api` in the `production` namespace. Because it lists `Ingress` in `policyTypes`, those pods become default-deny for **inbound** traffic — except traffic from pods labeled `app: web` (in the same namespace, since no `namespaceSelector` is given), and only on TCP port 8080. Any other pod, any other port, is now blocked from reaching `app: api` pods — this is the fundamental "3-tier app: web can talk to api, api can talk to db, nothing else can talk to db directly" pattern covered in depth with concrete examples in file 2.

## Points to Remember

- Kubernetes networking is flat and fully open by default — any pod can reach any other pod, any port, cluster-wide, until NetworkPolicies say otherwise.
- NetworkPolicies are purely allow-based (never explicit deny rules) — selecting a pod for a direction (ingress/egress) makes that direction default-deny for that pod, and only explicitly allowed traffic gets through; multiple applicable policies combine with OR logic.
- `podSelector: {}` with no rules and both `policyTypes` listed is the standard default-deny-all baseline for a namespace — apply this first, then layer specific allows.
- Default-deny-all blocks DNS too (it's just traffic to CoreDNS) — you must explicitly allow egress to UDP/TCP port 53 or every application in the namespace breaks immediately.
- A pod not selected by any NetworkPolicy at all remains fully open in both directions — NetworkPolicy is opt-in per pod, not a cluster-wide setting you flip once.

## Common Mistakes

- Deploying a default-deny-all policy without an accompanying DNS-allow policy, then getting flooded with application errors that look like generic connectivity failures but are actually every pod losing the ability to resolve any hostname.
- Believing NetworkPolicies support explicit "deny" rules the way a firewall ACL might — they don't; everything is expressed as "what to allow," and the deny behavior only emerges implicitly once a pod is selected for a given direction.
- Assuming a NetworkPolicy that only sets `policyTypes: [Ingress]` also restricts that pod's outbound traffic — it does not; egress remains fully open unless `Egress` is separately listed and has its own rules.
- Writing a NetworkPolicy and expecting it to take effect, without confirming the cluster's CNI plugin actually enforces NetworkPolicy at all — plain `kubenet`/some basic CNI setups silently ignore NetworkPolicy objects entirely (see file 3) so nothing errors, but nothing is actually enforced either.
- Forgetting that multiple NetworkPolicies selecting the same pod are combined with OR, not AND — assuming you need every applicable policy's conditions satisfied simultaneously, when in reality matching any single applicable policy's rule is sufficient to allow the traffic.
