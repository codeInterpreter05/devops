# Day 41 — NetworkPolicies: Selectors & Practical Patterns

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Kubernetes

## Brief

Default-deny (file 1) is the posture; **selectors** are the mechanism that lets you precisely describe "who specifically is allowed through." Getting comfortable combining `podSelector` and `namespaceSelector` — including the easy-to-miss detail of how they combine when used together — is what turns NetworkPolicy from a blunt on/off switch into a genuinely useful, fine-grained access-control tool. This file works through the concrete, real patterns you'll actually be asked to implement: isolating a database so only the API tier can reach it, and carving out an exception so a monitoring system can still scrape every pod despite an otherwise locked-down network.

## `podSelector` — matching pods by label, within a namespace

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: web
```

A bare `podSelector` (with no accompanying `namespaceSelector`) matches pods **only within the same namespace** as the NetworkPolicy itself. This is a common point of confusion — people expect `podSelector` alone to match a labeled pod anywhere in the cluster, but it's scoped to the policy's own namespace unless paired with a `namespaceSelector`.

## `namespaceSelector` — matching by namespace label

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
```

A bare `namespaceSelector` matches **all pods in namespaces with that label**, regardless of the pods' own labels. Every namespace automatically gets the `kubernetes.io/metadata.name` label (since Kubernetes 1.21+) set to its own name, which is why matching on that label is the standard, no-extra-setup way to select "the whole `monitoring` namespace."

## Combining both — the AND trap

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
        podSelector:
          matchLabels:
            app: prometheus
```

**Critical detail:** when `namespaceSelector` and `podSelector` appear together **inside the same list item** (same `-` entry under `from:`), they are combined with **AND** — "pods labeled `app: prometheus`, AND ONLY in namespaces labeled `kubernetes.io/metadata.name: monitoring`." This is different from listing them as **separate** entries:

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
      - podSelector:
          matchLabels:
            app: prometheus
```

Here, each `-` is a **separate** rule, combined with **OR** — "anything in the `monitoring` namespace, OR anything labeled `app: prometheus` in the *policy's own namespace*." This YAML indentation difference (same list item vs. separate list items) completely changes the security posture of the policy, and is one of the most consequential, easy-to-miss details in all of NetworkPolicy authoring — a misplaced dash can silently turn an intended narrow AND rule into a much broader OR rule.

## Pattern 1 — Three-tier app isolation (web → api → db)

Goal: `web` can talk to `api`; `api` can talk to `db`; nothing else can reach `db` or `api` directly; `web` is reachable from outside (via Ingress, not shown here).

```yaml
# Deny-all baseline (from file 1) assumed already applied to the namespace.

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: production
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: web } }
      ports:
        - protocol: TCP
          port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels: { app: db }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: api } }
      ports:
        - protocol: TCP
          port: 5432
```

With default-deny-all already in place, these two policies are strictly additive: `db` pods now accept traffic **only** from `api` pods on 5432; `api` pods accept traffic **only** from `web` pods on 8080. A compromised `web` pod cannot reach `db` directly — it would have to first compromise an `api` pod and pivot through it, which is exactly the lateral-movement resistance NetworkPolicy is meant to provide. Note this only restricts **ingress** to `db`/`api` — you'd typically pair this with matching **egress** rules on `web`/`api` too, for genuine default-deny-both-directions coverage (a policy restricting `db`'s ingress doesn't stop `db` itself from making arbitrary outbound connections unless `db` also has an egress-restricting policy).

## Pattern 2 — Monitoring exemption (Prometheus needs to scrape everything)

A fully locked-down namespace breaks metrics scraping unless Prometheus is explicitly allowed in, from every restricted pod's perspective:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: production
spec:
  podSelector: {}          # applies to ALL pods in this namespace
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090        # or whichever port each app exposes metrics on
```

This is layered on top of every other ingress rule already in place (remember: OR combination across policies) — it specifically carves out "Prometheus, from the monitoring namespace, may reach any pod in this namespace on its metrics port," without reopening any other traffic. This is the standard real-world pattern for reconciling "zero-trust internal networking" with "but our observability stack still needs broad read access to scrape metrics everywhere."

## Points to Remember

- A bare `podSelector` in a `from`/`to` rule is scoped to the **policy's own namespace** — it does not match same-labeled pods in other namespaces unless explicitly combined with `namespaceSelector`.
- A bare `namespaceSelector` matches **all pods** in matching namespaces, regardless of pod-level labels — use the auto-applied `kubernetes.io/metadata.name` label to target "the whole X namespace" without extra setup.
- `namespaceSelector` + `podSelector` in the **same** `from`/`to` list item = AND (narrow: this pod label, in this namespace). As **separate** list items = OR (broad: either condition independently) — this YAML structuring detail materially changes what's allowed.
- Three-tier isolation (web → api → db) is built by chaining independent ingress-restricting NetworkPolicies per tier — each tier only accepts traffic from the tier immediately in front of it, forcing an attacker to pivot through each layer rather than reaching the database directly.
- A monitoring/Prometheus exemption is typically layered on top of an otherwise-locked-down namespace as its own additive policy (`podSelector: {}` + a specific `from` rule for the monitoring namespace) rather than baked into every individual service's policy.

## Common Mistakes

- Writing `namespaceSelector` and `podSelector` as separate list items when an AND relationship was intended (or vice versa) — the resulting policy silently allows far more (or less) traffic than intended, with no error or warning from Kubernetes since both forms are valid YAML.
- Forgetting that a bare `podSelector` only matches within the same namespace — writing a "allow from `app: web`" rule expecting it to also match a same-labeled pod in a different namespace, when it actually matches nothing there.
- Only restricting ingress on the database tier and assuming that alone achieves "isolation," while the database pod itself still has unrestricted egress and could, if compromised through some other vector (e.g., a malicious extension), exfiltrate data outward freely.
- Building a monitoring exemption using only a `podSelector` for `app: prometheus` without a `namespaceSelector`, which — combined with the "podSelector is namespace-scoped" rule above — silently fails to match the real Prometheus pods running in a separate `monitoring` namespace.
- Not re-testing existing traffic paths after adding a new default-deny policy to a namespace — a previously-working inter-service call can silently start failing (with generic timeout-style errors, not a clear "blocked by NetworkPolicy" message) if it wasn't accounted for in the allow rules.
