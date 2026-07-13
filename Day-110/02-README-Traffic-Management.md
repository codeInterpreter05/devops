# Day 110 — Istio Deep Dive: Traffic Management

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Service Mesh | **Flag:** —

## Brief

This is the part of Istio you'll actually write YAML for most often: routing rules, canary splits, retries, timeouts, and circuit breaking — all declared as Kubernetes CRDs and compiled by istiod into Envoy config. Understanding the three core CRDs (`Gateway`, `VirtualService`, `DestinationRule`) and how they compose is the difference between copy-pasting mesh YAML and actually designing a rollout or resilience strategy.

## The three CRDs and how they relate

Think of it as a pipeline: **Gateway** (how traffic enters the mesh) → **VirtualService** (where it's routed, matched by host/path/headers) → **DestinationRule** (how it's treated once routed to a specific service — subsets, load balancing, circuit breaking, TLS).

### Gateway — the mesh's edge listener

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway     # picks the ingress gateway pods that implement this
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "bookinfo.example.com"
```

A `Gateway` only describes **which ports/protocols/hosts are exposed** on an ingress (or egress) Envoy proxy at the mesh edge. It does not describe routing — that's a `VirtualService`'s job. This split lets platform teams own the `Gateway` (network edge, TLS certs) while app teams own `VirtualService`s for their own routing — a common multi-tenancy pattern.

### VirtualService — the router

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v3
      weight: 10
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,connect-failure,refused-stream
    timeout: 10s
```

This shows header-based routing (dark-launch a specific user to `v2`) falling through to a weighted canary split (90/10 between v1/v3), plus per-route `retries` and an overall `timeout`. Note that `retries.attempts × perTryTimeout` can exceed the outer `timeout` — the outer timeout always wins and aborts in-flight retries when it fires. Setting `timeout: 10s` with `perTryTimeout: 5s` and `attempts: 3` means only about two attempts realistically fit before the outer timeout kills the whole request — a very common source of confusion when tuning resilience.

### DestinationRule — policy per destination

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    tls:
      mode: ISTIO_MUTUAL
```

`subsets` define named groups of pods (matched by label selector) that a `VirtualService` can route to by name (`v1`, `v2`, `v3` above). `trafficPolicy` is where **circuit breaking** actually lives in Istio's vocabulary:

- `connectionPool` limits concurrency to a destination — the "breaker" that prevents one slow backend from exhausting client-side connections/requests.
- `outlierDetection` implements **passive health checking**: if a specific endpoint returns `consecutive5xxErrors` within the `interval`, Envoy ejects it from the load-balancing pool for `baseEjectionTime` (with exponential backoff on repeat ejections), up to `maxEjectionPercent` of the pool. This is the classic circuit-breaker pattern, implemented client-side, per-Envoy, with no central coordinator.

## Why circuit breaking is decentralized

Each Envoy sidecar tracks outlier detection independently for the endpoints it talks to — there's no global circuit-breaker state shared across the mesh. This is deliberate: a centralized breaker would be a single point of failure and add a network hop on every decision. The tradeoff is that different clients may eject the same bad backend at slightly different times, which is fine in practice because the goal — stop hammering a failing pod — is achieved close to where the failure is observed.

## Points to Remember

- `Gateway` = edge listener (ports/hosts/TLS), `VirtualService` = routing logic (match + route + resilience), `DestinationRule` = per-destination policy (subsets, connection pools, outlier detection, client-side TLS).
- The outer `timeout` in a `VirtualService` always overrides retry math — check that `attempts × perTryTimeout` doesn't exceed it, or retries get silently truncated.
- Circuit breaking in Istio = `connectionPool` (concurrency limits) + `outlierDetection` (passive ejection on error bursts) inside `DestinationRule.trafficPolicy` — there's no separate "circuit breaker" CRD.
- Outlier detection is per-Envoy/decentralized, not a global mesh-wide breaker — expect slightly different ejection timing across clients calling the same backend.
- Weighted routing (canary %) and header-based routing (dark launch) both live in the same `VirtualService.http[].route`/`match` list — order matters, first match wins.

## Common Mistakes

- Writing a `DestinationRule` with `subsets` but forgetting the underlying pods don't actually carry the `version` label the subset selector expects — the subset silently matches zero pods and routing to it returns 503s.
- Setting `retries` without `perTryTimeout`, which lets a single slow attempt eat the entire budget before a retry even fires — always pair `attempts` with a `perTryTimeout` well below the outer `timeout`.
- Not realizing retries can amplify load during an incident: three retries on a struggling backend is 3x the request volume hitting an already-struggling service — combine retries with circuit breaking (outlier detection), don't use retries as a substitute for it.
- Forgetting `mode: ISTIO_MUTUAL` (or a compatible mode) in `DestinationRule.trafficPolicy.tls` when the mesh enforces STRICT mTLS — a mismatched TLS mode on the client side causes connection failures that look like an app-level bug.
- Applying a `VirtualService`/`DestinationRule` to a `host` that doesn't exactly match the Kubernetes `Service` name (including the namespace suffix for cross-namespace calls, e.g. `reviews.ns2.svc.cluster.local`) — this is a silent no-op with no error, just default routing behavior.
