# Day 119 — Zero Trust & mTLS: mTLS Everywhere with cert-manager & Istio

**Phase:** 4 – Advanced/Specialization | **Week:** W21 | **Domain:** Advanced Security | **Flag:** ⚡ Interview-critical

## Brief

**mTLS (mutual TLS)** means both sides of a connection present and verify a certificate — not just the server, as in the one-way TLS that secures your browser to `https://` sites. The sub-topic name for today is deliberately "mTLS *everywhere*, not just service mesh," because a common half-answer in interviews and in real platform designs is "we installed Istio, so we have mTLS." Istio's automatic sidecar mTLS is genuinely excellent, but it only covers traffic between pods that have an Envoy sidecar injected — it says nothing about a pod talking to a managed RDS database, a legacy VM, a workload in a mesh-less cluster, or a cross-cloud call. Real zero trust requires treating **every hop as adversarial**, mesh or not — which means you need both the mesh's automatic mTLS *and* a certificate lifecycle story (cert-manager) for everything the mesh doesn't reach.

## mTLS handshake mechanics — why it's stronger than tokens-over-TLS

A standard one-way TLS handshake (TLS 1.2/1.3): the client sends a `ClientHello`; the server responds with a `ServerHello` and its certificate; the client verifies the server's cert chain against a trusted CA, and the handshake completes. Only the *server's* identity was ever verified.

mTLS adds one critical step: after presenting its own certificate, the **server sends a `CertificateRequest`** back to the client, demanding the client present its own certificate too. The client responds with its certificate plus a `CertificateVerify` message — a signature proving it actually holds the private key matching that certificate (not just that it copy-pasted someone else's public cert). Both sides now verify each other's chain against a trusted CA before the connection is considered established.

**Why this beats "bearer token/API key over one-way TLS," the far more common pattern in the wild:** a bearer token is a value — if it leaks (in a log line, an error message, a misconfigured proxy header, a compromised downstream service that legitimately received it), anyone who has the string can replay it from anywhere. An mTLS client certificate's *private key* is what proves identity, and that private key, in a well-designed system, **never leaves the workload it belongs to** — SPIRE delivers key material over a local Unix socket the workload alone can read; well-configured cert-manager setups avoid ever writing the private key to a place a different workload could read it (the `cert-manager-csi-driver`/`csi-driver-spiffe` mounts keys via an ephemeral CSI volume rather than a long-lived Secret sitting in etcd). A stolen bearer token is directly reusable by whoever has it; a stolen certificate without its private key is worthless.

## cert-manager: certificate lifecycle for anything, mesh or not

**cert-manager** is a Kubernetes controller that watches `Certificate` custom resources, requests signed certs from a configured `Issuer`/`ClusterIssuer` (a self-signed root, an internal CA, HashiCorp Vault, or a public ACME provider like Let's Encrypt), and keeps them renewed automatically as Kubernetes Secrets — this is exactly the piece that lets you run mTLS between plain, mesh-less workloads (an app talking to a database sidecar, a VM, a cross-cluster batch job) with the same "no static secrets to babysit" property SPIRE gives you inside a cluster.

A self-signed root → intermediate CA → leaf-cert chain for internal mTLS:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-root-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: internal-root-ca
  secretName: internal-ca-key-pair
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-root-issuer
    kind: ClusterIssuer
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-ca-key-pair
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: payment-service-mtls
  namespace: production
spec:
  secretName: payment-service-tls
  duration: 24h
  renewBefore: 8h
  usages: ["client auth", "server auth"]
  dnsNames:
    - payment-service.production.svc.cluster.local
  issuerRef:
    name: internal-ca-issuer
    kind: ClusterIssuer
```

`duration: 24h` / `renewBefore: 8h` is deliberate: short-lived certs shrink the usable window of a stolen key from the traditional "1-year enterprise cert" down to hours, and cert-manager auto-renews well before expiry, rewriting the target Secret in place. **The operational trap:** many application frameworks read the TLS certificate from disk once at process startup and cache it in memory — cert-manager rotating the underlying Secret changes the file on disk, but the running process keeps serving the *old* (soon-to-expire, then expired) cert until it's restarted or explicitly reloads. Fix this with a sidecar/init pattern that watches the mounted Secret volume and signals the app to reload (or use `csi-driver-spiffe`/`cert-manager-csi-driver`, which mounts fresh key material via an ephemeral CSI volume and is designed around live rotation rather than a static Secret snapshot).

## Istio's automatic mesh-based mTLS

Istio issues its own workload certificates through `istiod` (or, in more security-conscious setups, delegates actual signing to an external CA via `istio-csr` fronting cert-manager) and injects an Envoy sidecar into every pod. Sidecar-to-sidecar traffic is automatically upgraded to mTLS, with each workload's identity encoded directly as a **SPIFFE ID** — Istio was one of SPIFFE's original adopters, and its identity format *is* `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`, so everything in the previous file about SPIFFE IDs applies directly.

`PeerAuthentication` controls the mTLS mode:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

- `STRICT` — reject any plaintext connection; only mTLS accepted.
- `PERMISSIVE` — accept both plaintext and mTLS on the same port; the standard **migration mode** for onboarding existing workloads without an outage while sidecars roll out gradually.
- `DISABLE` — plaintext only.

Precedence is workload-level > namespace-level > mesh-level (a `PeerAuthentication` named `default` in the `istio-system` namespace sets the mesh-wide default) — a common exam/production gotcha is setting mesh-wide `STRICT` and being surprised when one namespace still accepts plaintext, because a namespace- or workload-scoped policy there overrides it.

On top of mTLS-verified identity, `AuthorizationPolicy` makes the actual zero-trust access decision at L7:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-payment
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/charge"]
```

The `principals` field is literally the peer's SPIFFE ID as extracted from its verified mTLS certificate — this is zero trust actually landing in policy: the decision is "does the cryptographically verified caller identity match," never "does the caller's IP fall in an allowed range."

## The gap: mTLS beyond the mesh

Sidecar mTLS only covers workloads with a sidecar. It says nothing about:

- A managed database (RDS, Cloud SQL) — most teams connect with TLS-to-the-DB-only plus a password, not mutual cert auth, leaving a real gap inside an otherwise "zero trust" cluster.
- A legacy VM not running in the mesh — Istio addresses this partially via `WorkloadEntry`/VM onboarding (extending the mesh identity model to non-K8s workloads), but it requires deliberate setup, not something you get for free.
- A workload in a different, mesh-less cluster, or a third-party SaaS callback.

The fix in each case is the same principle applied manually where the mesh doesn't reach: cert-manager (or SPIRE, which also supports non-container workload attestation for bare VMs/processes) issuing and rotating certificates directly to those endpoints, egress gateways terminating/re-establishing mTLS toward external services, and — critically — refusing to consider a system "zero trust" just because the Kubernetes-internal hops are covered while the database connection or the VM-to-cluster hop is still plaintext-plus-password.

## Points to Remember

- mTLS = both sides present and verify certificates (the extra `CertificateRequest`/`CertificateVerify` step is what makes it "mutual"); one-way TLS only verifies the server.
- A stolen bearer token is directly reusable by an attacker; a stolen certificate without its private key is not — this is the core reason mTLS is a stronger identity primitive than tokens riding over one-way TLS.
- cert-manager gives any workload — mesh or not — automatic certificate issuance and renewal; short `duration`/`renewBefore` values shrink stolen-key blast radius but require the app (or a sidecar/CSI driver) to actually reload rotated certs, not just cache them at startup.
- Istio identities are SPIFFE IDs; `PeerAuthentication` sets the mTLS *mode* (STRICT/PERMISSIVE/DISABLE) with workload > namespace > mesh precedence, while `AuthorizationPolicy` makes the actual allow/deny decision based on the verified peer identity (`principals`).
- "mTLS everywhere" means every hop, not just intra-mesh hops — databases, VMs, and cross-cluster/cross-cloud calls need an explicit answer too, or the zero-trust story has an unstated gap.

## Common Mistakes

- Declaring "we have zero trust" because Istio is installed, without checking whether `PeerAuthentication` is actually `STRICT` anywhere — a mesh installed in `PERMISSIVE` mode (the sane migration default) still accepts plaintext everywhere until someone flips it.
- Forgetting `PeerAuthentication` precedence and being confused when a namespace- or workload-level policy silently overrides an intended mesh-wide `STRICT` setting.
- Treating cert-manager-issued certs as "set and forget" — rotation only helps if the application actually picks up the new cert; a cached-at-startup TLS context means rotation is invisible to the running process until a restart.
- Leaving database and VM connections on TLS-plus-password while the in-mesh traffic is fully mTLS, then presenting the architecture as uniformly zero-trust in an interview or design review — a reviewer who asks "what about the database hop" will immediately find the gap.
- Writing an `AuthorizationPolicy` `principals` match against the wrong SPIFFE ID format (forgetting the `cluster.local/ns/<ns>/sa/<sa>` shape, or copying a raw `spiffe://` URI where Istio expects it without the scheme) and having the policy silently match nothing, denying everything.
