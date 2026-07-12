# Day 57 — Service Mesh Intro: Istio Traffic Management

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Kubernetes | **Flag:** 📌

## Brief

Traffic management is the most immediately visible, demo-friendly capability of a service mesh, and it's exactly what today's hands-on activity (deploying Istio's Bookinfo sample app and configuring a 90/10 canary split) exercises. The two CRDs that do almost all of the work are `VirtualService` (routing rules — *where* traffic goes) and `DestinationRule` (policies applied to traffic *after* routing — subsets, load-balancing algorithm, connection pool/circuit-breaking settings). Understanding the division of labor between these two objects is the single most common point of confusion for people new to Istio.

## `DestinationRule` — defining subsets and per-subset policy

A Kubernetes `Service` load-balances across *all* pods matching its selector — it has no concept of "version." Istio's `DestinationRule` layers **subsets** on top of a Service, typically keyed off a `version` label already present on your pods, and can attach connection-pool, outlier-detection (circuit breaking), and load-balancing policy per subset:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews.default.svc.cluster.local
  subsets:
    - name: v1
      labels: {version: v1}
    - name: v2
      labels: {version: v2}
  trafficPolicy:
    connectionPool:
      tcp: {maxConnections: 100}
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

`outlierDetection` here is Istio's circuit breaker: if a specific pod backing the `reviews` service returns 5 consecutive 5xx errors, Envoy stops sending it traffic for `baseEjectionTime` (ejects it from the load-balancing pool), then gradually re-includes it — protecting the rest of the system from a single misbehaving instance without anyone manually intervening.

## `VirtualService` — routing rules

`VirtualService` decides *which subset* a request goes to, based on match conditions (headers, URI path, weights):

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
    - reviews.default.svc.cluster.local
  http:
    - route:
        - destination:
            host: reviews.default.svc.cluster.local
            subset: v1
          weight: 90
        - destination:
            host: reviews.default.svc.cluster.local
            subset: v2
          weight: 10
```

This is the exact shape of today's lab task: **90% of traffic to `v1`, 10% to `v2`** — a canary release where you validate a new version against a small slice of real production traffic before shifting more. Because this is enforced at the Envoy sidecar (the *caller's* sidecar decides the split, based on config pushed by `istiod`), there's no application code involved and no client-side logic deciding which version to call — the split is transparent to both the calling and called services.

**Weights must sum to 100** (or Istio rejects/ignores the config, depending on version) and are evaluated **per request**, not per connection — so a single client making many requests will see roughly the declared ratio across those requests, not get pinned to one subset for the life of a TCP connection (important to know, since some people mistakenly assume this behaves like session affinity by default).

## Header-based and path-based routing (beyond weighted splits)

`VirtualService` can route by content instead of (or in addition to) weight — the mechanism behind "internal employees see the canary, everyone else sees stable" patterns:

```yaml
spec:
  hosts: [reviews.default.svc.cluster.local]
  http:
    - match:
        - headers:
            end-user: {exact: "jason"}
      route:
        - destination: {host: reviews.default.svc.cluster.local, subset: v2}
    - route:
        - destination: {host: reviews.default.svc.cluster.local, subset: v1}
```

Match rules are evaluated **top to bottom, first match wins** — the opposite additive model from Kubernetes NetworkPolicy. This ordering matters: a catch-all rule (no `match:` block) must always go last, or it will shadow every more-specific rule below it.

## Fault injection — testing resilience deliberately

Because `VirtualService` sits in the actual request path, it can also inject synthetic failures for chaos-engineering-style tests without touching application code:

```yaml
spec:
  hosts: [ratings.default.svc.cluster.local]
  http:
    - fault:
        delay: {percentage: {value: 10}, fixedDelay: 5s}
        abort: {percentage: {value: 5}, httpStatus: 500}
      route:
        - destination: {host: ratings.default.svc.cluster.local}
```

This delays 10% of requests by 5 seconds and forces 5% to fail with a 500 — letting you validate that upstream services (timeouts, retries, fallback logic) actually behave correctly under partial failure, deliberately, in a controlled way, rather than discovering it during a real incident.

## Points to Remember

- `DestinationRule` defines subsets (usually by a `version` label) and per-subset traffic policy (connection pools, circuit breaking via `outlierDetection`); `VirtualService` defines routing rules that pick *which* subset a request goes to.
- Weighted routing splits are evaluated **per request**, not pinned per connection — don't assume sticky session behavior without adding it explicitly (e.g., via consistent-hash load balancing).
- `VirtualService` match rules are evaluated top-to-bottom, first-match-wins — always put the catch-all/default route last.
- `outlierDetection` is Istio's circuit breaker — it ejects a misbehaving pod from the load-balancing pool automatically based on consecutive error thresholds.
- Fault injection (`fault.delay`/`fault.abort`) lets you test resilience against latency/errors deliberately, entirely at the mesh layer, with zero application code changes.

## Common Mistakes

- Writing a `VirtualService` weighted split without a corresponding `DestinationRule` defining the subsets it references — Istio has nothing to route to and the config is effectively broken/ignored.
- Assuming a 90/10 weighted split gives one user a consistent experience across multiple requests (it doesn't, by default — it's evaluated per request, so a single user could bounce between v1 and v2 responses).
- Putting a catch-all (no-match) HTTP route *before* more specific header/path-matched routes in a `VirtualService` — since it's first-match-wins, the specific rules become unreachable dead code.
- Forgetting weights must sum to 100 across a route's destinations, leading to unexpected traffic distribution or config rejection.
- Treating fault injection as something you'd leave in a production `VirtualService` permanently — it's a testing tool; leaving `fault.abort` configured in prod after a chaos test is a classic "wait, why is 5% of our traffic failing" incident.
