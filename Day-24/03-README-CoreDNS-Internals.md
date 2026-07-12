# Day 24 — Networking: CoreDNS Internals

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Every time a pod calls `http://my-service` instead of a raw IP, CoreDNS is quietly doing the work that makes that possible. It's easy to treat cluster DNS as invisible plumbing until it breaks — and when it does (a common real-world failure mode), you need to understand it as a real, debuggable workload, not a mysterious built-in feature.

## CoreDNS is just a pod

CoreDNS runs as a Deployment (typically 2 replicas for HA) in the `kube-system` namespace, fronted by a `ClusterIP` Service conventionally named `kube-dns` (the name is a historical holdover from CoreDNS's predecessor, `kube-dns`). Every pod in the cluster has its `/etc/resolv.conf` automatically configured by the kubelet to point at that Service's ClusterIP as its nameserver:

```bash
kubectl exec <any-pod> -- cat /etc/resolv.conf
# nameserver 10.96.0.10          <- CoreDNS Service ClusterIP
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
kubectl get configmap -n kube-system coredns -o yaml     # the actual Corefile
```

## The Corefile: CoreDNS's plugin-chain config

CoreDNS is built around **plugins** chained together in a `Corefile`, loaded from a ConfigMap:

```
.:53 {
    errors
    health { lameduck 5s }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
        max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

- **`kubernetes` plugin** — the core piece; watches the apiserver for Services and Endpoints/EndpointSlices, and answers DNS queries for cluster-internal names by looking them up live, not from a static zone file.
- **`forward`** — anything CoreDNS doesn't recognize as an internal cluster name gets forwarded upstream (to `/etc/resolv.conf`, i.e., the node's own configured resolver, ultimately reaching real public DNS).
- **`cache`** — caches answers for the given TTL to reduce load; this is a common source of "I updated DNS and it takes a few seconds/dozens of seconds to reflect" confusion.
- **`loop`** — detects and breaks DNS forwarding loops (CoreDNS accidentally forwarding to itself).

## Name resolution formats

```
<service>.<namespace>.svc.cluster.local          # standard Service DNS name
<service>.<namespace>                              # shorthand, works within cluster
<service>                                            # shorthand, works only from within the same namespace
<pod-ip-with-dashes>.<namespace>.pod.cluster.local  # pod DNS (rare, used mostly for troubleshooting)
<pod>.<service>.<namespace>.svc.cluster.local        # per-pod DNS for StatefulSets behind a headless Service (Day 23)
```

`cluster.local` is the default cluster domain but is configurable at cluster-bootstrap time — some multi-cluster/federated setups use a different domain, which matters when writing DNS-based config that needs to work across clusters.

## `ndots:5` — the surprisingly expensive default

Every pod's `resolv.conf` sets `ndots:5`, meaning: if a queried name has **fewer than 5 dots**, the resolver tries appending each `search` domain first, in order, before trying the name as an absolute/fully-qualified query. So a call to `http://my-service` (0 dots) actually triggers, in the worst case:

```
my-service.default.svc.cluster.local.   -> (usually resolves here, 1st try)
```

But a call to an **external** domain like `api.stripe.com` (2 dots, still under 5) triggers up to 4-5 failed internal lookups before the 5th/6th attempt finally queries it as an absolute name:

```
api.stripe.com.default.svc.cluster.local.   -> NXDOMAIN
api.stripe.com.svc.cluster.local.            -> NXDOMAIN
api.stripe.com.cluster.local.                 -> NXDOMAIN
api.stripe.com.<node search domain, if any>. -> NXDOMAIN
api.stripe.com.                                -> finally resolves
```

This is a well-known, measurable source of extra latency and DNS query volume for apps that make lots of external HTTPS calls — each one pays for several wasted lookups. Fixes: use a **fully-qualified domain name with a trailing dot** (`api.stripe.com.`) in application config to skip the search-domain expansion entirely, or set `dnsConfig.options` on the pod spec to lower `ndots`, or (for very high-QPS external-call-heavy workloads) route external calls through a local caching resolver/sidecar.

```bash
# Diagnose DNS resolution and timing directly from inside a pod
kubectl run dns-debug --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup my-service.default.svc.cluster.local
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100     # CoreDNS's own logs (enable the `log` plugin for query-level detail)
```

## Points to Remember

- CoreDNS is an ordinary Deployment + Service in `kube-system` — it can be scaled, it can crash, it can be resource-starved, and it should be monitored like any other critical workload, not assumed infallible.
- Its behavior is entirely defined by the `Corefile` (a ConfigMap) and its plugin chain — `kubernetes` plugin answers internal names live from the apiserver's watch, `forward` sends everything else upstream.
- `ndots:5` (the default in every pod's `resolv.conf`) means short external hostnames incur multiple failed internal lookups before resolving — a real, measurable latency and DNS-load cost for external-call-heavy services.
- DNS caching (the `cache` plugin, default 30s) is why DNS-based changes (e.g., updating an `ExternalName` target) don't always propagate instantly.
- Per-pod DNS (`pod.service.namespace.svc.cluster.local`) only exists meaningfully behind a **headless** Service — a normal ClusterIP Service just resolves to the one virtual IP.

## Common Mistakes

- Treating a CoreDNS outage/overload as "the cluster network is broken" rather than diagnosing it as a specific, ordinary workload issue — check `kubectl get pods -n kube-system -l k8s-app=kube-dns` and CoreDNS's own resource usage/restarts first.
- Not accounting for `ndots:5` overhead in latency-sensitive, external-API-heavy services, then being confused why DNS query volume/latency is much higher than expected — the fix (trailing-dot FQDNs or tuned `dnsConfig`) is simple once diagnosed.
- Assuming a DNS change takes effect immediately everywhere, ignoring CoreDNS's cache TTL (and clients' own resolver caching) — "I changed it and it's still resolving to the old thing" is very often just cache still being valid.
- Scaling CoreDNS to a single replica (or letting cluster-autoscaler/resource pressure evict it down to one) — a lone CoreDNS pod becomes both a single point of failure and a bottleneck for every DNS-dependent operation cluster-wide.
- Forgetting the `search` domain expansion applies to **any** command run inside a pod, not just application code — a `curl somehost` in a debug shell inside a pod can behave differently (and more slowly) than the exact same command run on your laptop, purely due to `resolv.conf` differences.
