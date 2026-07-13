# Day 110 — Istio Deep Dive: Control Plane & Sidecar Injection

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Service Mesh | **Flag:** —

## Brief

A service mesh exists to pull cross-cutting concerns — retries, mTLS, traffic shaping, observability — out of application code and into an infrastructure layer of proxies. Istio is the most widely adopted implementation of that idea, and understanding *how* it actually works (not just what it markets itself as) is what separates a candidate who can explain the mechanism from one who can only repeat buzzwords. This file covers the control plane (istiod, the merger of Pilot/Citadel/Galley) and how the Envoy sidecar actually gets into your pods and onto your traffic path.

This day is split into three files:
1. **This file** — control plane architecture and sidecar injection mechanics.
2. **[02-README-Traffic-Management.md](02-README-Traffic-Management.md)** — `VirtualService`, `DestinationRule`, `Gateway`, retries/timeouts/circuit breaking.
3. **[03-README-mTLS-Security-And-Observability.md](03-README-mTLS-Security-And-Observability.md)** — mTLS modes and the Kiali/Jaeger/Prometheus stack.

## The historical split: Pilot, Citadel, Galley → istiod

Before Istio 1.5, the control plane was a set of separate microservices:

- **Pilot** — converts high-level traffic rules (`VirtualService`/`DestinationRule`) into Envoy-native xDS configuration and pushes it to sidecars.
- **Citadel** — the certificate authority: issues, rotates, and distributes SPIFFE-format X.509 certs to workloads for mTLS.
- **Galley** — configuration ingestion and validation, the gatekeeper for CRDs before they reached Pilot.
- **Mixer** (removed even earlier, in 1.5) — out-of-process policy/telemetry, deprecated because per-request out-of-process calls to Mixer added significant latency.

Since Istio 1.5, all of this is merged into a single binary/deployment called **istiod**. The merge bought operational simplicity: one deployment to scale, upgrade, and secure; fewer network hops between components (Pilot calling Galley over gRPC was itself a latency and failure source); and a smaller attack surface. The subsystems still exist logically (you'll see their names in logs and metrics) but no longer run as separate pods.

## What istiod actually does at runtime

1. Watches the Kubernetes API server (or other config sources) for Istio CRDs (`VirtualService`, `DestinationRule`, `Gateway`, `PeerAuthentication`, etc.) and native objects (`Service`, `EndpointSlice`).
2. Validates and translates that config into **xDS** (Envoy's discovery protocol family: LDS/RDS/CDS/EDS — Listener/Route/Cluster/Endpoint Discovery Service).
3. Pushes xDS updates to every Envoy sidecar over a persistent gRPC stream (**ADS** — Aggregated Discovery Service). This is a push model, not polling, so propagation is near-real-time.
4. Acts as the CA: the `istio-agent` sidecar process requests a certificate via the **SDS** (Secret Discovery Service) API over the same channel; istiod signs a short-lived, workload-bound X.509 cert.

## Envoy sidecar injection mechanics

Istio never modifies your application binary — it runs an Envoy proxy **container in the same pod** and transparently reroutes traffic through it via `iptables` rules (or eBPF/CNI in ambient mode — see Day 111 for eBPF fundamentals). There are two ways the sidecar container gets added:

**Automatic injection** — label the namespace and a Kubernetes **MutatingWebhookConfiguration** does the rest:

```bash
kubectl label namespace default istio-injection=enabled
```

Every new pod's spec is intercepted before scheduling, and istiod's injection webhook adds an `istio-proxy` container (the Envoy sidecar), an `istio-init` container (or CNI plugin) that programs iptables redirect rules, and shared volumes for certs and the Envoy bootstrap config.

**Manual injection** — for CI pipelines that render final YAML ahead of time:

```bash
istioctl kube-inject -f deployment.yaml | kubectl apply -f -
```

This runs the same templating logic client-side and emits plain YAML with the sidecar baked in — useful in GitOps flows where a live mutating webhook mid-apply is undesirable, or when you want to audit exactly what gets injected before it reaches the cluster.

Why `iptables`? The `istio-init` container (runs once, before the app and sidecar start) programs `PREROUTING`/`OUTPUT` chains so that inbound traffic to the pod is redirected to Envoy's inbound listener (port 15006) first, which forwards to the real app port on `localhost`; outbound traffic from the app is redirected to Envoy's outbound listener (15001) before it leaves the pod, so Envoy can apply routing, mTLS, and retries — even though the app just thinks it made a normal socket call. The redirection happens below the socket layer, which is exactly why the app is unaware it's in a mesh.

## Points to Remember

- istiod = Pilot (xDS translation) + Citadel (CA/cert issuance) + Galley (config validation) merged into one deployment since 1.5; Mixer was removed earlier for latency reasons.
- Config propagation is push-based over a persistent gRPC stream (ADS), not polling — changes reach sidecars in roughly sub-second-to-a-few-seconds time in a healthy mesh.
- Sidecar injection can be automatic (namespace label + mutating webhook) or manual (`istioctl kube-inject`) — both produce the same end result, just at different points in the pipeline.
- Traffic redirection into the sidecar happens via `iptables` rules set up by an init container (or a CNI plugin/eBPF in ambient mode) — the application is unmodified and unaware.
- Certificates are short-lived and auto-rotated by istiod acting as CA — there's no long-lived static cert to manage per workload.

## Common Mistakes

- Enabling `istio-injection=enabled` on a namespace, then wondering why *already-running* pods aren't in the mesh — the webhook only fires on pod creation; existing pods need `kubectl rollout restart deployment/<name>`.
- Assuming istiod pushes config instantaneously and debugging a "stale route" as an app bug, when it's actually an istiod push delay — check `istioctl proxy-status` (`SYNCED` vs `STALE`) first.
- Forgetting the `istio-init` container needs `NET_ADMIN`/`NET_RAW` capabilities to write iptables rules — this breaks under restrictive Pod Security Standards/admission unless explicitly allowed, a common cause of a sidecar `CrashLoopBackOff`.
- Confusing "istiod is down" with "mesh traffic stops" — existing sidecars keep their last-received xDS config and keep working; only *new* config changes and *newly starting* pods are affected until istiod recovers. This nuance matters a lot during an incident postmortem.
