# Day 23 — Workloads: StatefulSets

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Deployments assume every replica is interchangeable and disposable — kill one, start a new one with a random name and a random IP, nobody cares. That assumption breaks completely for databases, message brokers, and anything with per-instance identity or ordering requirements (Postgres primary/replica, a Kafka broker, an Elasticsearch node, Redis Cluster/Sentinel). `StatefulSet` exists specifically to give up some of a Deployment's flexibility in exchange for the guarantees stateful systems need: **stable, unique network identity** and **ordered, predictable lifecycle**. This is a very commonly asked interview topic precisely because "when would you use a StatefulSet over a Deployment" tests whether you understand *why* the distinction exists, not just that it does.

## Stable network identity

Every Pod in a StatefulSet gets a **predictable, stable name**: `<statefulset-name>-0`, `<statefulset-name>-1`, `<statefulset-name>-2`, ... — never a random suffix like a Deployment's pods. Critically, if `redis-1` is deleted and recreated, **it comes back as `redis-1`, not a new name** — same ordinal, same identity.

This only becomes *useful* when paired with a **headless Service** (`clusterIP: None`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None          # headless — no virtual IP, no load-balancing
  selector:
    app: redis
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis        # must match the headless Service above
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - containerPort: 6379
```

With a headless Service, CoreDNS returns **one DNS A record per pod**, each individually addressable at a stable name:

```
redis-0.redis.default.svc.cluster.local
redis-1.redis.default.svc.cluster.local
redis-2.redis.default.svc.cluster.local
```

This is exactly how you tell a Redis Cluster node or a Kafka broker "here is your peer at a fixed address" in config — you cannot do this with a Deployment, where pod names and IPs are random and unstable across restarts.

## Ordered, sequential lifecycle

By default (`podManagementPolicy: OrderedReady`), a StatefulSet:
- **Scales up** one pod at a time, in order (`-0`, then `-1`, then `-2`) — pod `n` isn't created until pod `n-1` is `Running` and `Ready`.
- **Scales down** in strict reverse order (`-2`, then `-1`, then `-0`).
- **Rolling updates** by default also go in reverse ordinal order (highest number first), one at a time, waiting for each to become Ready before touching the next.

This matters for systems where ordinal 0 is special (e.g., often used as the initial cluster-forming node, or a designated primary before an election happens) or where an uncontrolled simultaneous restart of all replicas would break quorum (etcd, ZooKeeper, Kafka — losing quorum mid-update is exactly what ordered rollout prevents). If your app doesn't care about order, `podManagementPolicy: Parallel` gets you Deployment-like speed while keeping stable identity/storage.

## Stable storage: `volumeClaimTemplates`

Each StatefulSet pod gets its **own PersistentVolumeClaim**, generated from a template and named deterministically (`<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>`):

```yaml
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 10Gi
```

`redis-0` always gets `data-redis-0`, even after being deleted and recreated — the volume follows the identity, not the other way around. **Critically, deleting a StatefulSet (or scaling it down) does *not* delete its PVCs by default** — this is a deliberate safety choice (see Day 25 for the full PV/PVC reclaim story) so that scaling `replicas: 3` down to `1` and back up to `3` gives you your original data back on `-1` and `-2`, not empty volumes.

```bash
kubectl get statefulset redis
kubectl get pods -l app=redis -o wide          # note the -0/-1/-2 suffixes
kubectl get pvc -l app=redis                    # one PVC per pod, survives pod deletion
kubectl exec redis-0 -- hostname                 # confirm the pod's own hostname matches its ordinal name
```

## Points to Remember

- StatefulSet gives three guarantees a Deployment does not: stable pod names/ordinals, stable per-pod DNS via a headless Service, and stable per-pod storage via `volumeClaimTemplates`.
- A headless Service (`clusterIP: None`) is a prerequisite for per-pod DNS records — a normal ClusterIP Service load-balances across pods and hides individual pod identity, which is the opposite of what StatefulSets need.
- Default ordering is strict and sequential (scale up ascending, scale down/rolling-update descending) — `podManagementPolicy: Parallel` opts out if your workload doesn't need it.
- PVCs created via `volumeClaimTemplates` **outlive** pod deletion and even StatefulSet deletion by default — you must delete them manually if you actually want to wipe data.
- Use StatefulSets for anything with per-instance identity, ordering requirements, or "this replica must always reconnect to its own data" semantics — not just "because it's a database" (many databases actually run fine as a Deployment with a single replica plus an external PVC, if there's no clustering involved).

## Common Mistakes

- Reaching for a StatefulSet automatically for "anything with a database" — a single-replica Postgres instance with one PVC doesn't need per-pod ordinals or headless DNS; a StatefulSet only earns its complexity when there's actual clustering/replication with peer identity involved.
- Forgetting to create the headless Service (or mismatching `spec.serviceName`) — the StatefulSet still works, but you silently lose the stable per-pod DNS names, which breaks any clustering config that depends on them.
- Assuming scaling a StatefulSet down to 0 and back up gives you fresh, empty volumes — it doesn't; the old PVCs are still bound and get reattached, which surprises people expecting a "fresh start" and can also surprise people who *didn't* expect old data to reappear.
- Deleting a StatefulSet expecting all its data to disappear (a "clean slate") — the PVCs remain; you must explicitly `kubectl delete pvc -l app=<name>` if you actually want to reclaim/wipe the storage.
- Ignoring `podManagementPolicy` and being confused why a 5-replica StatefulSet takes noticeably longer to scale up than an equivalent Deployment — it's sequential by default, by design, not a performance bug.
