# Day 23 — Cheatsheet: Workloads — Deployments & StatefulSets

## Deployments

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl set image deployment/web nginx=nginx:1.26         # trigger a rolling update
kubectl rollout status deployment/web                      # block until rollout completes/fails
kubectl rollout history deployment/web                     # list revisions
kubectl rollout history deployment/web --revision=2        # diff of a specific revision
kubectl rollout undo deployment/web                        # roll back one revision
kubectl rollout undo deployment/web --to-revision=1         # roll back to specific revision
kubectl rollout pause deployment/web
kubectl rollout resume deployment/web
kubectl scale deployment/web --replicas=6
kubectl get rs -l app=web                                    # see owned ReplicaSets + pod-template-hash
```

## Deployment rollout strategy (YAML)

```yaml
spec:
  strategy:
    type: RollingUpdate       # or: Recreate
    rollingUpdate:
      maxUnavailable: 1        # or "25%"
      maxSurge: 1               # or "25%"
  minReadySeconds: 10
  revisionHistoryLimit: 10
```

## StatefulSets

```bash
kubectl get statefulset
kubectl get pods -l app=<name> -o wide     # ordinals: <name>-0, <name>-1, ...
kubectl get pvc -l app=<name>               # one PVC per ordinal, survives pod deletion
kubectl scale statefulset <name> --replicas=5
kubectl delete pod <name>-1                  # comes back with SAME name/identity/storage
kubectl rollout status statefulset/<name>
```

## StatefulSet YAML skeleton

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: redis }
spec:
  serviceName: redis            # must match a headless Service
  podManagementPolicy: OrderedReady   # or: Parallel
  replicas: 3
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers:
        - name: redis
          image: redis:7
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 10Gi } }
---
apiVersion: v1
kind: Service
metadata: { name: redis }
spec:
  clusterIP: None                # headless -> per-pod DNS records
  selector: { app: redis }
  ports: [{ port: 6379 }]
```

Per-pod DNS: `<pod>.<service>.<namespace>.svc.cluster.local` e.g. `redis-1.redis.default.svc.cluster.local`

## DaemonSets

```bash
kubectl get daemonset -A
kubectl get pods -o wide -l app=<name>       # should equal node count (matching selector/tolerations)
kubectl rollout status daemonset/<name>
```

```yaml
spec:
  template:
    spec:
      tolerations:
        - operator: Exists       # run on tainted control-plane nodes too
```

## Pod lifecycle hooks

```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      lifecycle:
        postStart:
          exec: { command: ["sh", "-c", "echo started"] }   # async, no ordering guarantee vs main process
        preStop:
          exec: { command: ["sh", "-c", "sleep 10"] }        # kubelet WAITS for this before SIGTERM
```

Shutdown sequence: endpoint removal (parallel) -> `preStop` runs & completes -> `SIGTERM` -> grace period -> `SIGKILL` if still alive.
