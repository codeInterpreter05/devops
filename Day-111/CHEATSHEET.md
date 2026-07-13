# Day 111 — Cheatsheet: Cilium & eBPF

## Cilium install / lifecycle

```bash
cilium install                                  # default install
cilium install --set kubeProxyReplacement=true \
  --set k8sServiceHost=<IP> --set k8sServicePort=6443
cilium status --wait                             # wait for healthy install
cilium status                                    # current health snapshot
cilium connectivity test                         # full E2E connectivity/policy test suite
cilium upgrade                                   # in-place upgrade
cilium uninstall                                 # remove Cilium
```

## Cilium diagnostics

```bash
cilium sysdump                       # collect a full debug bundle for support/troubleshooting
kubectl -n kube-system exec -it <cilium-pod> -- cilium status
kubectl -n kube-system exec -it <cilium-pod> -- cilium endpoint list
kubectl -n kube-system exec -it <cilium-pod> -- cilium bpf lb list      # eBPF-programmed load-balancer entries
kubectl -n kube-system exec -it <cilium-pod> -- cilium monitor          # live low-level event stream
```

## Hubble

```bash
cilium hubble enable --ui           # enable Hubble Relay + UI
cilium hubble port-forward &         # local access to Relay
hubble status                        # connection health

hubble observe --follow                          # live flow stream
hubble observe --namespace default --follow      # scope to a namespace
hubble observe --pod default/client --follow     # scope to a pod
hubble observe --verdict DROPPED --follow         # only denied flows
hubble observe --protocol http --follow           # only HTTP (L7) flows
hubble observe --since 5m                          # historical window

cilium hubble ui                     # open the graphical service map
```

## CiliumNetworkPolicy essentials

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: example }
spec:
  endpointSelector: { matchLabels: { app: web } }
  ingress:
  - fromEndpoints: [{ matchLabels: { run: client } }]
    toPorts:
    - ports: [{ port: "80", protocol: TCP }]
      rules:
        http:
        - { method: "GET", path: "/" }
  egress:
  - toEndpoints: [{ matchLabels: { app: db } }]
    toPorts: [{ ports: [{ port: "5432", protocol: TCP }] }]
```

```bash
kubectl apply -f policy.yaml -n <ns>
kubectl get ciliumnetworkpolicies -A
kubectl describe ciliumnetworkpolicy <name>
```

## Bandwidth management

```yaml
metadata:
  annotations:
    kubernetes.io/egress-bandwidth: "10M"
```
```bash
cilium config view | grep bandwidth   # confirm bandwidth manager enabled
```

## Tetragon

```bash
helm install tetragon cilium/tetragon -n kube-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=tetragon

kubectl apply -f tracing-policy.yaml           # apply a TracingPolicy
tetra getevents                                 # stream raw Tetragon events (from the tetragon pod)
tetra getevents -o compact                      # human-readable event stream
kubectl get tracingpolicies                      # list active policies
```

## Quick mental model

```
iptables (kube-proxy):  packet -> walk N sequential rules -> match  (O(n))
eBPF (Cilium):           packet -> hash-map lookup                   (O(1))

Cilium   = CNI layer   = L3/L4 (+ optional L7)  = mandatory
Istio    = app layer   = L7 (assumes a CNI)     = optional add-on
```
