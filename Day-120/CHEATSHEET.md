# Day 120 — Cheatsheet: CKS Exam Prep

## CKS curriculum domains (approx. weights — verify current numbers)

```
Cluster Setup                              ~10%
Cluster Hardening                          ~15%
System Hardening                           ~15%
Minimize Microservice Vulnerabilities      ~20%
Supply Chain Security                      ~20%
Monitoring, Logging and Runtime Security   ~20%
```

## kube-bench

```bash
kube-bench run --targets master,node,etcd,policies
kube-bench run --targets master              # control-plane only
kube-bench run --targets etcd                 # etcd checks only
kube-bench --version 1.24                     # pin CIS benchmark version
kube-bench run --benchmark eks-1.0            # managed-cluster-specific variant (EKS example)
kube-bench run --json > results.json          # machine-readable output
```

## Audit logging

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources: [{group: "", resources: ["secrets"]}]
    verbs: ["get", "list"]
  - level: Metadata
    omitStages: ["RequestReceived"]
  - level: None
    users: ["system:kube-proxy"]
```

```bash
# kube-apiserver static pod manifest flags (+ matching volumeMounts/volumes!)
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
--audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml   # ship to SIEM instead

# querying
cat audit.log | jq 'select(.verb=="delete" and .objectRef.resource=="secrets")'
cat audit.log | jq 'select(.user.username=="system:serviceaccount:default:ci-bot")'
```

## Falco

```bash
helm install falco falcosecurity/falco -n falco --create-namespace \
  --set driver.kind=modern_ebpf
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
falco --list                                  # list all loaded rules
falco -r /path/rules.yaml --dry-run           # validate rule syntax
```

```yaml
- rule: Terminal shell in container
  desc: A shell was spawned in a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
  output: >
    Shell in container (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
  priority: WARNING
```

## Seccomp

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault      # | Localhost | Unconfined
    # localhostProfile: profiles/audit.json   # only with type: Localhost
```

```bash
# profile file location (kubelet-config dependent)
/var/lib/kubelet/seccomp/profiles/<name>.json
```

## AppArmor

```bash
apparmor_parser -q -r /path/to/profile    # load a profile on THIS node
aa-status                                  # list loaded profiles + enforcement mode
```

```yaml
securityContext:
  appArmorProfile:
    type: Localhost                      # | RuntimeDefault | Unconfined
    localhostProfile: k8s-nginx-restricted
# legacy (pre-1.30): annotation on the Pod
# container.apparmor.security.beta.kubernetes.io/<container>: localhost/k8s-nginx-restricted
```

## Trivy / Trivy Operator

```bash
trivy image myregistry/myapp:1.2.3            # standalone CLI, CI-time scan
trivy image --severity CRITICAL,HIGH myapp     # filter by severity
trivy config ./k8s-manifests/                  # misconfig scan on manifests/IaC
trivy fs --scanners secret .                   # scan for leaked secrets in a repo

helm install trivy-operator aqua/trivy-operator -n trivy-system --create-namespace

kubectl get vulnerabilityreports -A
kubectl get configauditreports -A
kubectl get exposedsecretreports -A
kubectl get vulnerabilityreports -A -o json | jq '.items[] | select(.report.summary.criticalCount > 0)'
```

## Image signing (cosign / sigstore)

```bash
cosign generate-key-pair
cosign sign --key cosign.key myregistry/myapp@sha256:<digest>   # sign by DIGEST, not tag
cosign verify --key cosign.pub myregistry/myapp@sha256:<digest>
```

## Kyverno admission policy skeleton

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-runtime-default-seccomp
spec:
  validationFailureAction: Audit    # flip to Enforce after burn-in
  rules:
  - name: check-seccomp
    match:
      any: [{resources: {kinds: ["Pod"]}}]
    validate:
      message: "seccompProfile must be RuntimeDefault"
      pattern:
        spec:
          securityContext:
            seccompProfile: {type: RuntimeDefault}
```

## NetworkPolicy hardening baseline

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny-all, namespace: production}
spec: {podSelector: {}, policyTypes: ["Ingress", "Egress"]}
---
# don't forget DNS egress or every pod loses name resolution
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: allow-dns, namespace: production}
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
  - to: []
    ports: [{protocol: UDP, port: 53}, {protocol: TCP, port: 53}]
```

## Exam-day static-pod reminder

```bash
# control-plane hardening = editing these directly, NOT kubectl apply
/etc/kubernetes/manifests/kube-apiserver.yaml
/etc/kubernetes/manifests/kube-controller-manager.yaml
/etc/kubernetes/manifests/kube-scheduler.yaml
/etc/kubernetes/manifests/etcd.yaml
# kubelet watches this dir and restarts the static pod on save —
# a bad edit (missing volumeMount) crash-loops it with NO kubectl feedback
```
