# Day 23 — Workloads: DaemonSets & Pod Lifecycle Hooks

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Rounding out the workload types: `DaemonSet` solves a completely different problem than Deployments/StatefulSets — not "run N copies of my app," but "run exactly one copy of this agent on every node, always." And **pod lifecycle hooks** (`postStart`/`preStop`) are the fine-grained control you have over what happens at the exact moment a container starts or is about to be killed — small in surface area, but responsible for a huge share of "why did my rolling deploy drop requests for a second" incidents.

## DaemonSet: one pod per (matching) node, automatically

A DaemonSet doesn't take a `replicas:` count — instead, the DaemonSet controller ensures exactly one pod runs on every node that matches its (optional) node selector/affinity, and automatically adds a pod when a new node joins the cluster, and removes it when a node leaves. You never manually scale a DaemonSet to match your node count — that's the entire point.

Canonical real-world uses:
- **Log collection agents**: Fluent Bit, Fluentd, Filebeat — must run on every node to tail that node's container logs.
- **Node-level metrics/monitoring**: `node-exporter` (Prometheus), Datadog agent — needs per-node OS/kernel metrics, not something a central Deployment could see.
- **CNI/networking agents**: Calico's `calico-node`, Cilium's agent — literally *is* the pod networking implementation on each node, must exist everywhere.
- **Storage daemons**: CSI node plugins (e.g., `ebs-csi-node`) — handle attach/mount operations local to each node.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      tolerations:                     # DaemonSets typically need to tolerate control-plane taints
        - operator: Exists
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          ports:
            - containerPort: 9100
              hostPort: 9100            # often binds to the host's network directly
          resources:
            requests: { cpu: 50m, memory: 50Mi }
            limits: { memory: 100Mi }
```

Note the `tolerations` — by default, control-plane/master nodes are tainted to repel ordinary pods; a monitoring/log DaemonSet usually *wants* to run on control-plane nodes too (you need metrics from those nodes as well), so it must explicitly tolerate that taint. This is the most common gotcha: "why is my DaemonSet missing a pod on the control-plane node" is almost always a missing toleration.

```bash
kubectl get daemonset -A
kubectl get pods -o wide -l app=node-exporter    # one pod per node, matching node count exactly
kubectl rollout status daemonset/node-exporter    # DaemonSets support rolling updates too (RollingUpdate/OnDelete)
```

## Pod lifecycle hooks: `postStart` and `preStop`

These are container-level hooks, defined per-container in the pod spec, that let you run a command or HTTP request at specific points in a container's life — separate from and in addition to liveness/readiness probes.

```yaml
spec:
  containers:
    - name: app
      image: myapp:v2
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo 'container started' >> /var/log/lifecycle.log"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 15"]
      terminationGracePeriodSeconds: 30
```

### `postStart` — fires immediately after the container is created

Runs **asynchronously** with the container's own `ENTRYPOINT`/`CMD` — there's no guarantee it runs *before* your app's main process starts serving traffic, only that it's triggered roughly at the same time. It's commonly used for last-mile setup that can't be baked into the image (registering with a service registry, warming a local cache) but you should never rely on it for "must happen before the app is reachable" — a readiness probe is the right tool for that, not `postStart`.

### `preStop` — the most operationally important hook

Fires **before** the container receives `SIGTERM`, and the kubelet **waits for it to finish** before sending `SIGTERM` at all. This is the standard mechanism for graceful shutdown of a pod that's about to be removed (scale-down, rolling update, node drain):

1. Pod is marked for termination → **removed from Service endpoints immediately** (this happens in parallel with, not after, the hook).
2. `preStop` hook runs (if defined) and the kubelet waits for it.
3. `SIGTERM` sent to the container's main process.
4. If the process hasn't exited after `terminationGracePeriodSeconds` (default 30s) minus however long `preStop` took, `SIGKILL` is sent.

**Why `preStop: sleep N` is such a common pattern:** step 1 (endpoint removal) and the actual iptables/IPVS rule propagation to every node's kube-proxy are **not instantaneous** — there's a real propagation delay, typically sub-second to a few seconds depending on cluster size. Without a brief `preStop` delay, a pod can receive `SIGTERM` and start shutting down *before* every node's kube-proxy has stopped routing new connections to it, causing a small burst of connection-refused errors during every rolling deploy. A `preStop` hook that just sleeps 5-15 seconds before allowing `SIGTERM` gives that propagation time to complete, at the cost of a slightly slower shutdown.

```bash
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].lifecycle}'
kubectl logs <pod> --previous          # check what happened right before a container was killed/restarted
```

## Points to Remember

- DaemonSet = exactly one pod per matching node, automatically added/removed as nodes join/leave — never manually scaled.
- DaemonSets commonly need explicit `tolerations` to also run on tainted control-plane nodes, since infra-level agents (logging, monitoring, CNI, CSI) usually need presence everywhere.
- `postStart` runs asynchronously with container startup — no ordering guarantee relative to your app's own readiness; don't use it to gate traffic.
- `preStop` runs and **completes** before `SIGTERM` is sent, and is the standard place to add a short delay for graceful, zero-dropped-request shutdowns during rolling updates.
- Total shutdown budget is `terminationGracePeriodSeconds` (default 30s); if `preStop` + graceful app shutdown exceed that, the process gets `SIGKILL`'d anyway — tune the grace period to fit your app's real shutdown time, not just the default.

## Common Mistakes

- Forgetting a DaemonSet needs a toleration for the control-plane taint, then wondering why `kubectl get pods -o wide` shows one fewer pod than node count.
- Using `postStart` for critical initialization and assuming it blocks the main process from starting — it doesn't; there is no ordering guarantee, and a slow/failing `postStart` can even kill the container (a failed `postStart` hook is treated as a container failure) without your app ever getting a chance to start cleanly.
- Not setting any `preStop` hook and being confused by a small spike of 502/connection-refused errors on every deploy — this is almost always Service-endpoint-removal/kube-proxy-propagation racing against `SIGTERM`, fixed with a short `preStop: sleep`.
- Setting `terminationGracePeriodSeconds` far too low (or leaving the 30s default) for an app that needs longer to drain in-flight requests (e.g., long-running jobs, WebSocket connections) — causing `SIGKILL` to cut off work mid-request.
- Treating DaemonSets as immune to resource pressure — they still consume CPU/memory on every node they run on; an unbounded or overly generous DaemonSet resource request multiplied across a large cluster is a common source of unexpectedly high baseline resource consumption.
