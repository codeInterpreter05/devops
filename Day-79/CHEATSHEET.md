# Day 79 — Cheatsheet: Runtime Security

## Falco rule anatomy

```yaml
- macro: <name>              # reusable condition fragment
  condition: <expr>

- list: <name>                # reusable value list
  items: [a, b, c]

- rule: <name>
  desc: <human description>
  condition: >
    <boolean expression over fields like proc.name, container.id, evt.type>
  output: >
    <templated alert string with %field placeholders>
  priority: EMERGENCY|ALERT|CRITICAL|ERROR|WARNING|NOTICE|INFORMATIONAL|DEBUG
  tags: [container, shell, ...]
```

Common fields: `proc.name`, `proc.pname`, `proc.cmdline`, `proc.tty`, `container.id`, `container.image.repository`, `k8s.ns.name`, `k8s.pod.name`, `user.name`, `fd.name`, `evt.type`.

```bash
# Falco install / operate
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco -n falco --create-namespace --set driver.kind=ebpf
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

## Seccomp

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault      # or: Localhost (needs profile file on node) | Unconfined
```

```json
// custom profile at /var/lib/kubelet/seccomp/profiles/<name>.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    { "names": ["read", "write", "open", "close"], "action": "SCMP_ACT_ALLOW" }
  ]
}
```

## AppArmor

```yaml
# K8s 1.30+ (GA field)
securityContext:
  appArmorProfile:
    type: RuntimeDefault      # or: Localhost + localhostProfile: <name> | Unconfined

# Pre-1.30 (annotation form)
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/<container>: runtime/default
```

## Hardened pod template

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile: { type: RuntimeDefault }
containers:
  - securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities: { drop: ["ALL"] }
      appArmorProfile: { type: RuntimeDefault }
```

## Kubernetes audit policy

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources: [{ group: "", resources: ["pods/exec", "pods/attach", "secrets"] }]
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
  - level: Metadata
    omitStages: ["RequestReceived"]
```

```
apiserver flags:
  --audit-policy-file=<path>
  --audit-log-path=<path>
  --audit-log-maxage / --audit-log-maxbackup / --audit-log-maxsize
  --audit-webhook-config-file=<path>   # ship to external SIEM
```

## Detecting an exec into a container (the interview answer)

```
1. Falco:  rule "Terminal shell in container" fires -> proc.tty != 0 inside a container
2. Audit log: verb=connect, objectRef.subresource=pods/exec, user.username, sourceIPs
3. Correlate timestamps -> full picture: WHAT happened (Falco) + WHO did it (audit log)
```

## Incident response quick sequence

```
1. Contain    - NetworkPolicy deny-all on pod, or cordon node — do NOT delete the pod yet
2. Collect    - kubectl logs / describe, filesystem snapshot, matching audit log entries
3. Correlate  - line up Falco alert timestamp with audit log event
4. Eradicate  - rotate all credentials the pod could access, redeploy from clean image
5. Recover    - restore traffic gradually, watch for recurrence
6. Postmortem - blameless writeup, feed fix back into rules/RBAC/CI gates
```
