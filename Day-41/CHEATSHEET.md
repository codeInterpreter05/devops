# Day 41 — Cheatsheet: NetworkPolicies

## Default-deny-all (namespace baseline)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-all, namespace: production }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

## Allow DNS egress (required alongside any default-deny)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-dns, namespace: production }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to: [{ namespaceSelector: {} }]
      ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
```

## Basic ingress rule shape

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-web-to-api, namespace: production }
spec:
  podSelector: { matchLabels: { app: api } }
  policyTypes: [Ingress]
  ingress:
    - from: [{ podSelector: { matchLabels: { app: web } } }]
      ports: [{ protocol: TCP, port: 8080 }]
```

## Selector combination rules

```
podSelector alone        -> matches pods, SAME namespace as the policy only
namespaceSelector alone   -> matches ALL pods in matching namespace(s)

- from:
  - namespaceSelector: {...}
    podSelector: {...}          # SAME list item = AND
- from:
  - namespaceSelector: {...}
  - podSelector: {...}           # SEPARATE list items = OR
```

## Cross-namespace selector (auto-labeled since k8s 1.21+)

```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: monitoring
```

## Monitoring exemption pattern

```yaml
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: monitoring } }
          podSelector: { matchLabels: { app: prometheus } }
      ports: [{ protocol: TCP, port: 9090 }]
```

## Key semantics

```
- No policy selects a pod       -> fully open (both directions)
- Policy selects pod + Ingress   -> that pod's INGRESS becomes default-deny except listed rules
- Policy selects pod + Egress    -> that pod's EGRESS becomes default-deny except listed rules
- Multiple applicable policies   -> combined with OR (any match = allowed)
- NetworkPolicy = allow-only     -> there is no explicit "deny" rule in the base K8s API
```

## Verifying enforcement (CNI check)

```bash
kubectl get pods -n kube-system | grep -Ei 'calico|cilium|weave'
kubectl get nodes -o wide
```

## Calico GlobalNetworkPolicy (beyond base K8s API)

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata: { name: deny-metadata-service }
spec:
  order: 100
  selector: all()
  types: [Egress]
  egress:
    - action: Deny
      destination: { nets: ["169.254.169.254/32"] }
```

## CiliumNetworkPolicy (Layer 7)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: api-allow-get-only }
spec:
  endpointSelector: { matchLabels: { app: api } }
  ingress:
    - fromEndpoints: [{ matchLabels: { app: web } }]
      toPorts:
        - ports: [{ port: "8080", protocol: TCP }]
          rules:
            http:
              - method: "GET"
                path: "/api/v1/.*"
```

## Testing connectivity quickly

```bash
kubectl run test --rm -it --image=busybox --labels="app=web" -n shop -- sh -c "nc -zv -w3 <target-svc> <port>"
```
