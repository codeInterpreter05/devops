# Day 57 — Cheatsheet: Service Mesh Intro

## Istio install & basics

```bash
istioctl install --set profile=demo -y        # install control plane
kubectl get pods -n istio-system              # istiod, ingress gateway health
kubectl label namespace default istio-injection=enabled   # auto-inject sidecars

istioctl analyze                              # lint mesh config for common mistakes
istioctl proxy-status                         # sync status of every sidecar with istiod
istioctl proxy-config routes <pod>.<ns>       # dump actual Envoy route config
istioctl proxy-config cluster <pod>.<ns>      # dump actual Envoy cluster (upstream) config
istioctl uninstall --purge -y                 # full uninstall
```

## `DestinationRule` — subsets + policy

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata: {name: reviews-destination}
spec:
  host: reviews.default.svc.cluster.local
  subsets:
    - {name: v1, labels: {version: v1}}
    - {name: v2, labels: {version: v2}}
  trafficPolicy:
    connectionPool:
      tcp: {maxConnections: 100}
    outlierDetection:                 # = circuit breaker
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

## `VirtualService` — routing

```yaml
# weighted canary split (per-request, not per-connection)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: {name: reviews-route}
spec:
  hosts: [reviews.default.svc.cluster.local]
  http:
    - route:
        - {destination: {host: reviews.default.svc.cluster.local, subset: v1}, weight: 90}
        - {destination: {host: reviews.default.svc.cluster.local, subset: v2}, weight: 10}
```

```yaml
# header-based routing — MUST come before the catch-all (first-match-wins)
http:
  - match: [{headers: {end-user: {exact: "jason"}}}]
    route: [{destination: {host: reviews, subset: v2}}]
  - route: [{destination: {host: reviews, subset: v1}}]   # catch-all, must be last
```

```yaml
# fault injection — for deliberate resilience testing only, remove after
http:
  - fault:
      delay: {percentage: {value: 10}, fixedDelay: 5s}
      abort: {percentage: {value: 5}, httpStatus: 500}
    route: [{destination: {host: ratings}}]
```

## Linkerd

```bash
linkerd check --pre                          # pre-install validation
linkerd install | kubectl apply -f -         # install control plane
linkerd check                                # post-install validation
linkerd viz install | kubectl apply -f -     # observability extension

kubectl annotate namespace myapp linkerd.io/inject=enabled
linkerd viz stat deploy -n myapp             # golden metrics: success rate, RPS, latency
linkerd viz top deploy/myapp -n myapp        # live request-level view
linkerd uninstall | kubectl delete -f -
```

## Istio vs Linkerd, at a glance

| | Istio | Linkerd |
|---|---|---|
| Data plane | Envoy (general-purpose) | linkerd2-proxy (Rust, purpose-built) |
| mTLS | on, configurable via `PeerAuthentication` | on by default, minimal config |
| Traffic split API | `VirtualService`/`DestinationRule` (custom CRDs) | historically SMI `TrafficSplit`, now own extensions |
| Install complexity | higher, more moving parts | deliberately minimal |
| Feature breadth | widest (fault injection, rich L7 rules) | narrower, focused on core needs |

## When to skip a mesh entirely

```
< ~5-10 interdependent services         -> premature
no mTLS/compliance need, no canary need -> unused value
no platform team to operate it          -> becomes the outage, not the fix
narrower tool solves the actual problem -> API gateway / cert-manager / client lib instead
```
