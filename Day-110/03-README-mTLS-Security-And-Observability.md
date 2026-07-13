# Day 110 — Istio Deep Dive: mTLS Security & Observability

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Service Mesh | **Flag:** —

## Brief

"How does Istio implement mTLS? What certs, how rotated?" is the flagged interview question for this day, and it's asked because mTLS-by-default is Istio's single biggest security selling point over running services bare. Pairing that with the observability stack (Kiali/Jaeger/Prometheus) closes the loop: a mesh isn't just "encrypt everything," it's "encrypt everything *and* let you see what's actually happening between services," which is what makes it operable at scale.

## mTLS in Istio: the mechanism

Every workload in the mesh gets a cryptographic identity, not just a network address. Istio uses the **SPIFFE** (Secure Production Identity Framework For Everyone) identity format, encoded into the SAN (Subject Alternative Name) field of an X.509 certificate:

```
spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>
```

This ties identity to the pod's **Kubernetes ServiceAccount**, not its IP (IPs are ephemeral and reused; identity shouldn't be). That's why swapping a pod's IP doesn't affect mTLS — the cert is bound to the ServiceAccount, and any pod scheduled with that ServiceAccount gets the matching identity.

**Issuance and rotation flow:**

1. The `istio-agent` process (running alongside Envoy in the sidecar) generates a private key and a CSR for its pod's identity.
2. It sends the CSR to istiod over the same secured gRPC channel used for xDS, using the Istio CA's gRPC API.
3. istiod's built-in CA (or an external CA/plugin like `cert-manager` with `istio-csr`, or Vault) validates the request against the Kubernetes ServiceAccount token bound to that pod, and signs a certificate — by default valid for **24 hours**. SDS (Secret Discovery Service) delivers it to Envoy, never touching disk.
4. Before expiry (roughly at 80% of the lifetime), `istio-agent` automatically requests a new cert — rotation is continuous and silent, no restart required, and no per-workload Kubernetes Secret object involved by default.

The root CA cert itself is longer-lived and can be self-signed by istiod or provided by an external CA/intermediate — production setups typically plug in an external root (via a `cacerts` Secret, or `istio-csr` + cert-manager) so the root of trust isn't tied to Istio's own ephemeral state.

## PERMISSIVE vs STRICT — mTLS modes

Controlled by `PeerAuthentication`:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system     # mesh-wide when in the root/config namespace
spec:
  mtls:
    mode: STRICT
```

- **PERMISSIVE** (the safe default when first enabling the mesh): a sidecar accepts *both* plaintext and mTLS connections on the same port. This lets you onboard services incrementally — pods still outside the mesh (no sidecar yet) keep working with plaintext, while meshed pods automatically start using mTLS to each other. Without this, flipping on Istio mesh-wide would break every caller that hasn't been injected yet.
- **STRICT**: only mTLS connections are accepted; plaintext is rejected. This is the end state you migrate to once every workload in scope has a sidecar.

Mode can be set at mesh level, namespace level, or per-workload (`selector`) — the most specific scope wins. A namespace-level policy overrides the mesh-wide default, and a workload-level policy overrides the namespace one. This layered override model recurs across most of Istio's security/config CRDs (e.g. `AuthorizationPolicy`), so learning it once here pays off elsewhere.

**Practical migration path:** mesh-wide `PERMISSIVE` → inject sidecars everywhere → verify with Kiali's traffic graph or `istioctl` that everything is actually using mTLS → flip to mesh-wide `STRICT` → keep narrow `PERMISSIVE` exceptions only for legacy non-meshed callers you can't yet migrate (e.g., a legacy VM hitting the cluster through an East-West gateway).

## Observability stack

- **Kiali** — visualizes the *service graph* built from Envoy telemetry: which services call which, with live traffic percentages, error rates, and — crucially — whether an edge is using mTLS (shown as a padlock icon). This is usually the fastest way to *prove* STRICT mTLS is actually active end-to-end, rather than trusting the YAML.
- **Jaeger** (or any OpenTelemetry-compatible backend) — distributed tracing. Envoy propagates trace headers (`x-request-id`, B3/W3C `traceparent`) automatically between hops, but each *service* must forward those headers on any outbound call it makes for the trace to stay connected — Envoy doesn't inject app-level context automatically. A service that drops incoming trace headers when making its own downstream call will show up as a separate, disconnected trace fragment.
- **Prometheus** — every Envoy sidecar exposes rich request/connection metrics (`istio_requests_total`, `istio_request_duration_milliseconds`, labeled by source/destination/response code) scraped by Prometheus. Grafana's Istio dashboards are built directly on these metrics.

## Points to Remember

- Identity is SPIFFE-based, tied to the Kubernetes ServiceAccount, not the pod IP — this is *why* mTLS survives pod restarts/rescheduling seamlessly.
- Certs are short-lived (default 24h) and auto-rotated by `istio-agent` well before expiry, delivered via SDS — no per-workload Kubernetes Secret to manage or leak.
- `PeerAuthentication` controls mTLS enforcement (`PERMISSIVE`/`STRICT`/`DISABLE`), scoped mesh-wide → namespace → workload, most specific wins.
- `PERMISSIVE` exists specifically to allow incremental mesh rollout without breaking not-yet-injected callers — it's a migration tool, not a target end state.
- Kiali proves mTLS visually (padlock on graph edges); Jaeger requires services to propagate trace headers on their own outbound calls, or traces fragment; Prometheus is the metrics backbone behind almost every Istio dashboard.

## Common Mistakes

- Flipping straight to `STRICT` mesh-wide on day one, breaking any pod that isn't injected yet (batch jobs, admission webhooks, ingress from outside the mesh) — always land on `PERMISSIVE`, verify, then tighten.
- Assuming mTLS certificates need manual rotation/renewal like a typical cert-manager-issued TLS cert — Istio's default workload cert lifecycle is fully automatic and much shorter-lived by design, which reduces the blast radius of a leaked cert.
- Treating Jaeger trace gaps as a tracing-system bug when the real cause is application code not forwarding `x-request-id`/`traceparent`/`b3` headers on its own outbound requests.
- Confusing `PeerAuthentication` (mTLS transport security — *is this connection encrypted/authenticated*) with `AuthorizationPolicy` (*is this authenticated caller allowed to call this endpoint*) — they solve different problems and both are usually needed for real zero-trust posture.
- Not checking `istioctl proxy-config`/Kiali after a mode change — assuming the change took effect mesh-wide instantly when a stale sidecar (one that hasn't received the latest push) is still negotiating the old mode.
