# Day 119 — Cheatsheet: Zero Trust & mTLS

## Zero trust core principles (NIST SP 800-207)

```
1. No implicit trust from network location (being "inside" grants nothing)
2. Authenticate + authorize every request on identity + context
3. Least privilege, dynamically enforced
4. Assume breach — design as if attacker is already inside
5. Continuous verification, not one-time-at-login
```

## BeyondCorp vs. Zero Trust Architecture

```
BeyondCorp   = Google's specific user-to-app implementation
               (device identity + user identity -> access proxy, no VPN-as-trust)
Zero Trust   = broader NIST framework, also covers workload-to-workload
               (mTLS + workload identity + service mesh authz)
```

## NetworkPolicy (L3/L4 micro-segmentation)

```yaml
# default-deny-all (ingress + egress) in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
# allow only from=app:frontend, tcp/8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  podSelector: {matchLabels: {app: backend}}
  policyTypes: ["Ingress"]
  ingress:
  - from: [{podSelector: {matchLabels: {app: frontend}}}]
    ports: [{protocol: TCP, port: 8080}]
```

```bash
# reminder: NetworkPolicy does NOTHING without an enforcing CNI
# Calico / Cilium / Weave -> enforce
# default kind bridge CNI / some minimal CNIs -> silently no-op
```

## SPIFFE / SPIRE

```
SPIFFE ID:  spiffe://<trust-domain>/<path>
SVID:       X.509-SVID (cert, for mTLS) or JWT-SVID (bearer, for L7/serverless)
Lifetime:   short (SPIRE default ~1h), auto-rotated via local Workload API socket
```

```bash
# registration entry (run against spire-server)
spire-server entry create \
  -spiffeID  spiffe://example.org/ns/production/sa/payment-service \
  -parentID  spiffe://example.org/spire/agent/k8s_psat/<cluster>/<agent-id> \
  -selector  k8s:ns:production \
  -selector  k8s:sa:payment-service

spire-server agent list                 # list attested nodes/agents
spire-server entry show                 # list registration entries
spire-agent api fetch x509 \
  -socketPath /run/spire/sockets/agent.sock   # fetch this workload's SVID
spire-agent api fetch jwt -audience myaudience \
  -socketPath /run/spire/sockets/agent.sock   # fetch a JWT-SVID
```

```
Attestation chain:
  Node attestation    -> Agent proves which node it's on (k8s_psat / cloud IID)
  Workload attestation -> Agent inspects PID->cgroup->container->runtime metadata
  Selector matching   -> attested facts matched against registration entries
  Issuance            -> Server signs SVID, delivered over local socket
```

## cert-manager

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: {name: selfsigned-root-issuer}
spec: {selfSigned: {}}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: internal-ca, namespace: cert-manager}
spec:
  isCA: true
  commonName: internal-root-ca
  secretName: internal-ca-key-pair
  privateKey: {algorithm: ECDSA, size: 256}
  issuerRef: {name: selfsigned-root-issuer, kind: ClusterIssuer}
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: {name: internal-ca-issuer}
spec: {ca: {secretName: internal-ca-key-pair}}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: svc-mtls, namespace: production}
spec:
  secretName: svc-tls
  duration: 24h
  renewBefore: 8h
  usages: ["client auth", "server auth"]
  dnsNames: ["svc.production.svc.cluster.local"]
  issuerRef: {name: internal-ca-issuer, kind: ClusterIssuer}
```

```bash
kubectl get certificate -A                    # check Ready status
kubectl describe certificate <name> -n <ns>   # see renewal/issuance events
kubectl get secret <secretName> -o yaml       # inspect issued cert/key (base64)
```

## Istio mTLS

```yaml
# STRICT mTLS, namespace-scoped (workload > namespace > mesh precedence)
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: {name: default, namespace: production}
spec: {mtls: {mode: STRICT}}   # STRICT | PERMISSIVE | DISABLE
```

```yaml
# identity-based L7 authorization on top of verified mTLS identity
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata: {name: allow-frontend, namespace: production}
spec:
  selector: {matchLabels: {app: payment-service}}
  action: ALLOW
  rules:
  - from: [{source: {principals: ["cluster.local/ns/production/sa/frontend"]}}]
    to: [{operation: {methods: ["POST"], paths: ["/api/charge"]}}]
```

```bash
istioctl proxy-config secret <pod> -n <ns>      # inspect sidecar's live cert
openssl x509 -in cert.pem -noout -text | grep -A1 "Subject Alternative Name"  # find SPIFFE URI
istioctl authz check <pod>.<ns>                  # debug applied AuthorizationPolicies
```

## mTLS handshake (mental model)

```
ClientHello ->
         <- ServerHello + server cert
         <- CertificateRequest        (THIS step = "mutual")
Client cert + CertificateVerify ->
Both verify each other's chain against trusted CA -> session established
```
