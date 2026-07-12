# Day 50 — Cheatsheet: Probes, Resources & QoS

## Probes — YAML skeletons

```yaml
startupProbe:
  httpGet: { path: /healthz, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10          # total boot budget = failureThreshold * periodSeconds

livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3         # fails -> container restarted

readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 3         # fails -> removed from Service Endpoints, no restart

# other probe mechanisms:
livenessProbe:
  exec: { command: ["cat", "/tmp/healthy"] }
livenessProbe:
  tcpSocket: { port: 8080 }
livenessProbe:
  grpc: { port: 9090 }
```

| Probe | Fail action | Restart? |
|---|---|---|
| startup | keep waiting; kill container if budget exhausted | yes (if exhausted) |
| liveness | restart container | yes |
| readiness | remove from Service Endpoints | no |

## Requests / limits — YAML

```yaml
resources:
  requests: { cpu: "250m", memory: "256Mi" }   # scheduling reservation
  limits:   { cpu: "500m", memory: "512Mi" }   # hard ceiling: CPU throttles, memory OOMKills
```

## QoS classes

| Class | Rule | Eviction order |
|---|---|---|
| Guaranteed | requests == limits, both CPU & memory, every container | last |
| Burstable  | some requests/limits set, doesn't qualify for Guaranteed | middle |
| BestEffort | no requests/limits at all | first |

```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

## Inspecting resources vs. usage

```bash
kubectl describe node <node>            # "Allocated resources" = sum of REQUESTS (scheduling view)
kubectl top node                         # actual live usage
kubectl top pod --containers             # actual live usage per container
kubectl describe pod <name>              # Last State / Reason / Exit Code
```

## OOMKilled diagnosis

```bash
kubectl describe pod <name> | grep -A5 "Last State"   # Reason: OOMKilled / Exit Code: 137
kubectl get events --field-selector involvedObject.name=<name>
kubectl describe node <node> | grep -A5 Conditions     # check MemoryPressure
```
Exit code 137 = 128 + 9 (SIGKILL). CPU limit breach -> throttled (no kill). Memory limit breach -> OOMKilled.

## LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata: { name: default-limits, namespace: team-a }
spec:
  limits:
    - type: Container
      default: { cpu: "500m", memory: "512Mi" }        # applied if limit omitted
      defaultRequest: { cpu: "250m", memory: "256Mi" }  # applied if request omitted
      min: { cpu: "50m", memory: "64Mi" }
      max: { cpu: "2", memory: "2Gi" }
      maxLimitRequestRatio: { cpu: "4" }
```

## ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: team-a-quota, namespace: team-a }
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services.loadbalancers: "2"
```

```bash
kubectl describe resourcequota team-a-quota -n team-a   # used vs hard, per resource
kubectl describe limitrange default-limits -n team-a
```

Rule: if a ResourceQuota sets `requests.cpu`/`requests.memory`, every pod in that namespace MUST specify them (or be paired with a LimitRange that auto-fills defaults) or creation is rejected.
