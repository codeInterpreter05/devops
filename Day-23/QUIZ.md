# Day 23 — Quiz: Workloads — Deployments & StatefulSets

Try to answer without looking at your notes. Answers are at the bottom.

1. Does a Deployment manage Pods directly? If not, what's the actual ownership chain?
2. What mechanism makes `kubectl rollout undo` fast, and why doesn't it need to re-pull images or rebuild anything?
3. What do `maxUnavailable` and `maxSurge` each control during a rolling update?
4. Why is a missing or weak readiness probe the most common cause of a rollout that "succeeds" but serves errors?
5. What guarantees does a StatefulSet provide that a Deployment does not? Name all three.
6. What role does a headless Service (`clusterIP: None`) play for a StatefulSet, and what happens if you forget to create one?
7. What is the default pod creation/deletion order for a StatefulSet, and why does that ordering matter for real clustered systems like etcd or Kafka?
8. If you delete a StatefulSet entirely, what happens to its PersistentVolumeClaims by default?
9. What problem does a DaemonSet solve that a Deployment cannot, even with a very high replica count?
10. Why do DaemonSets commonly need explicit `tolerations`?
11. What's the difference in timing/ordering guarantees between a `postStart` hook and a `preStop` hook, and why does that difference matter for using them correctly?
12. **Interview question:** When would you use a StatefulSet over a Deployment? What guarantees does it provide?

---

## Answers

1. No — a Deployment manages **ReplicaSets**, and each ReplicaSet manages Pods. The chain is Deployment → ReplicaSet → Pod. A rolling update works by creating a new ReplicaSet (new pod-template-hash) and shifting replica counts between the old and new ReplicaSet, not by mutating existing pods.
2. Because the previous ReplicaSet (with its full pod template) is still present, just scaled to 0 — it wasn't deleted, only scaled down. Rolling back is simply scaling the old ReplicaSet back up and the current one down, which is why it's fast and doesn't involve re-pulling images or rebuilding anything.
3. `maxUnavailable` caps how many pods below the desired `replicas` count can be down at once during the rollout (limits the capacity dip). `maxSurge` caps how many extra pods above `replicas` can exist temporarily while new and old versions coexist (limits the temporary over-capacity). Both can be absolute numbers or percentages.
4. Kubernetes only knows a new pod is "safe to count as available" once it passes its readiness probe (and stays ready for `minReadySeconds`) — the rollout controller uses readiness as its only signal to gate progression. If there's no readiness probe (or one that only checks "process is alive" rather than "app can actually serve requests"), Kubernetes will happily mark the rollout complete even though the new version is silently broken.
5. (1) Stable, unique pod identity/ordinals (`name-0`, `name-1`, ...) that persist across restarts. (2) Stable per-pod DNS names via a headless Service. (3) Stable per-pod storage via `volumeClaimTemplates` — each ordinal keeps its own PVC across pod recreation.
6. A headless Service is what makes CoreDNS return one DNS A record per individual pod (`pod-name.service.namespace.svc.cluster.local`) instead of load-balancing across a single virtual IP. Without it (or with a mismatched `spec.serviceName`), the StatefulSet still works mechanically, but you silently lose per-pod DNS addressability, which breaks any clustering configuration that depends on peers finding each other by stable hostname.
7. Default (`OrderedReady`): scale-up creates pods in ascending order (`-0` before `-1` before `-2`, each waiting for the previous to be Ready), scale-down and rolling updates proceed in descending order. This matters because clustered systems (etcd, ZooKeeper, Kafka) can lose quorum or break bootstrap assumptions if all replicas restart simultaneously or if ordinal 0 (often special, e.g. a cluster-forming/seed node) isn't guaranteed to exist before others join.
8. By default the PVCs are **not** deleted — they remain bound and intact even after the StatefulSet itself is deleted. This is a deliberate safety default so that recreating the StatefulSet (or scaling back up) reattaches to existing data rather than losing it. You must explicitly delete the PVCs if you want to wipe the data.
9. A DaemonSet guarantees exactly one pod per matching node, automatically adjusting as nodes are added or removed — no manual replica count to keep in sync with cluster size. A Deployment with a high replica count has no concept of "nodes" at all — the scheduler could pack multiple replicas onto the same node and leave others with none, which defeats the purpose of a per-node agent like a log collector or CNI daemon.
10. Nodes (especially control-plane/master nodes) are often tainted to repel ordinary workloads by default. Infrastructure-level DaemonSets (logging, monitoring, CNI, CSI node plugins) typically need to run on *every* node including control-plane nodes, so they must explicitly declare `tolerations` to be scheduled there despite the taint.
11. `postStart` fires asynchronously relative to the container's main process — there is no guarantee it completes (or even starts) before the app begins accepting traffic, so it should never be used for "must happen before ready" logic. `preStop` is the opposite: the kubelet runs it and **waits for it to complete** before sending `SIGTERM` — this is a real, honored ordering guarantee, which is why `preStop` (not `postStart`) is the correct place to add a graceful-shutdown delay.
12. Strong answer: "I'd use a StatefulSet when replicas aren't interchangeable — when the app needs stable network identity (each replica reachable at a consistent hostname via a headless Service), stable per-replica storage that survives pod restarts, and/or ordered startup/shutdown. Classic examples are Kafka, ZooKeeper/etcd-style clustered systems, and Elasticsearch/Redis Cluster nodes that need to know their peers by a fixed address. A Deployment assumes every pod is disposable and anonymous, which breaks the moment an instance needs to reconnect to *its own* data or be addressed individually rather than load-balanced. The tradeoff is StatefulSets are slower to scale (sequential by default) and more operationally involved — I wouldn't use one for a stateless API just because it happens to have a database dependency."
