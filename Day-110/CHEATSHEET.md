# Day 110 — Cheatsheet: Istio Deep Dive

## Install / lifecycle

```bash
istioctl install --set profile=demo -y      # install with demo profile
istioctl install --set profile=minimal -y   # minimal control plane only
istioctl verify-install                     # confirm install is healthy
istioctl uninstall --purge -y               # remove everything
istioctl version                            # client + control plane versions
istioctl upgrade                            # in-place upgrade
```

## Sidecar injection

```bash
kubectl label namespace <ns> istio-injection=enabled     # enable auto-injection
kubectl label namespace <ns> istio-injection-             # disable
kubectl rollout restart deployment/<name>                 # re-inject already-running pods
istioctl kube-inject -f deploy.yaml | kubectl apply -f -   # manual injection
```

## Debugging config / sync state

```bash
istioctl proxy-status                       # SYNCED vs STALE per sidecar
istioctl proxy-config cluster <pod>          # Envoy CDS view for a pod
istioctl proxy-config listener <pod>         # Envoy LDS view
istioctl proxy-config route <pod>            # Envoy RDS view
istioctl proxy-config endpoint <pod>          # Envoy EDS view
istioctl x describe pod <pod>                # human-readable summary: routes, policies, mTLS
istioctl analyze                             # static config validation / lint
```

## Gateway

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata: { name: my-gateway }
spec:
  selector: { istio: ingressgateway }
  servers:
  - port: { number: 80, name: http, protocol: HTTP }
    hosts: ["example.com"]
```

## VirtualService — routing, retries, timeout, fault injection

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: { name: my-vs }
spec:
  hosts: ["my-svc"]
  http:
  - match: [{ headers: { end-user: { exact: jason } } }]
    route: [{ destination: { host: my-svc, subset: v2 } }]
  - route:
    - { destination: { host: my-svc, subset: v1 }, weight: 90 }
    - { destination: { host: my-svc, subset: v3 }, weight: 10 }
    retries: { attempts: 3, perTryTimeout: 2s, retryOn: 5xx,connect-failure }
    timeout: 5s
    fault:
      abort: { percentage: { value: 50 }, httpStatus: 500 }
      delay: { percentage: { value: 10 }, fixedDelay: 3s }
```

## DestinationRule — subsets, circuit breaking, TLS

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: { name: my-dr }
spec:
  host: my-svc
  subsets:
  - { name: v1, labels: { version: v1 } }
  - { name: v2, labels: { version: v2 } }
  trafficPolicy:
    connectionPool:
      tcp: { maxConnections: 100 }
      http: { http1MaxPendingRequests: 50, maxRequestsPerConnection: 10 }
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    tls: { mode: ISTIO_MUTUAL }   # DISABLE | SIMPLE | ISTIO_MUTUAL | MUTUAL
```

## Security — PeerAuthentication & AuthorizationPolicy

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: default, namespace: istio-system }   # mesh-wide
spec:
  mtls: { mode: STRICT }   # PERMISSIVE | STRICT | DISABLE
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata: { name: allow-productpage, namespace: default }
spec:
  selector: { matchLabels: { app: reviews } }
  action: ALLOW
  rules:
  - from: [{ source: { principals: ["cluster.local/ns/default/sa/productpage"] } }]
    to: [{ operation: { methods: ["GET"] } }]
```

## Observability addons

```bash
kubectl apply -f samples/addons          # Kiali, Jaeger, Prometheus, Grafana
istioctl dashboard kiali
istioctl dashboard jaeger
istioctl dashboard grafana
istioctl dashboard prometheus
```

## Useful Envoy stats via istioctl

```bash
kubectl exec <pod> -c istio-proxy -- pilot-agent request GET stats | grep <upstream-cluster>
kubectl exec <pod> -c istio-proxy -- pilot-agent request GET stats | grep outlier_detection
kubectl exec <pod> -c istio-proxy -- pilot-agent request GET stats | grep upstream_rq_pending_overflow
```
