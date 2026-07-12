# Day 65 — Deployment Strategies: Rolling Update & Recreate

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** GitOps | **Flag:** ⚡ Interview-critical

## Brief

Every deployment strategy is fundamentally a tradeoff between **speed**, **resource cost**, and **availability during the transition**. Kubernetes' two built-in strategies — Rolling Update (the default) and Recreate — sit at opposite ends of that tradeoff, and understanding exactly *why* each behaves the way it does is the foundation for everything more advanced (blue/green, canary — covered in file 2) since those are really just more sophisticated variations on "how do we replace old Pods with new ones safely."

This day is split into three files:

1. **This file** — Rolling Update (the Kubernetes default) and Recreate.
2. **[02-README-Blue-Green-And-Canary.md](02-README-Blue-Green-And-Canary.md)** — Blue/Green with Argo Rollouts and Canary with traffic splitting.
3. **[03-README-Feature-Flags-And-Rollback.md](03-README-Feature-Flags-And-Rollback.md)** — Feature flags (LaunchDarkly, Unleash) and automated rollback on failure.

## Rolling Update — the Kubernetes default

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # how many EXTRA pods can exist above `replicas` during rollout
      maxUnavailable: 25%  # how many pods can be unavailable below `replicas` during rollout
  template:
    spec:
      containers:
        - name: app
          image: my-app:v2
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 5
```

**Mechanism**: the Deployment controller creates a *new* ReplicaSet (for `v2`) alongside the *old* one (`v1`), and incrementally scales the new one up while scaling the old one down, bounded by `maxSurge`/`maxUnavailable`. With 10 replicas and the defaults shown (25%/25%), Kubernetes can briefly run up to 12–13 pods total (surge) while never dropping below 7–8 healthy pods (unavailable) — the exact numbers are rounded from percentages, which is worth knowing because `maxSurge: 25%` on a *small* replica count (say, 2) rounds to a surge of 1, not a fraction of a pod.

**Why it needs a working readiness probe**: the controller only considers a new pod "up" and proceeds to remove an old one once the new pod passes its `readinessProbe`. Without a real readiness probe (or with one that returns `200 OK` before the app has actually finished initializing — e.g., before it's connected to its database), Kubernetes will happily route traffic to a not-actually-ready pod, or worse, keep terminating old (working) pods faster than new ones are genuinely ready, producing a visible availability dip precisely because the safety mechanism was fed bad information.

**Tradeoffs**:
- **Pros**: zero planned downtime (assuming correct readiness probes and enough surge capacity), gradual and automatically self-limiting, built into vanilla Kubernetes with no extra tooling.
- **Cons**: **both old and new versions run simultaneously** for the duration of the rollout — this is fine for a stateless app with no breaking API/schema changes, but a real problem if `v1` and `v2` can't safely coexist (e.g., `v2` requires a database schema change that `v1`'s code can't handle, or they can't share a cache format). Rolling update gives you **no control over which users/requests hit old vs. new** — it's a blunt instrument for anything beyond "just replace the pods safely."

## Recreate — the blunt, downtime-accepting strategy

```yaml
spec:
  strategy:
    type: Recreate
```

**Mechanism**: Kubernetes terminates **all** existing pods first, waits for them to fully terminate, and only then creates the new version's pods. There's no overlap between old and new — at any given moment, either all pods are old, all are new, or (briefly) none exist at all.

**Why you'd deliberately choose downtime**: 
- **Incompatible schema/data migrations** — if `v1` and `v2` genuinely cannot run against the same underlying data simultaneously (e.g., a destructive or non-backward-compatible database migration), running both at once (as Rolling Update does) isn't just suboptimal, it's actively broken — old pods would crash or corrupt data against the new schema. Recreate guarantees no such overlap exists.
- **Singleton workloads** — an app that can't have two instances running against the same resource at once (a leader-only batch processor, a resource holding an exclusive lock) may need a clean stop-then-start rather than a gradual replacement.
- **Resource-constrained environments** — Rolling Update needs headroom for the surge (`maxSurge`) — extra CPU/memory to run old and new simultaneously. In a resource-starved cluster (dev/test environments, or cost-optimized clusters running near capacity), Recreate avoids needing that extra headroom at all, at the cost of downtime.

## Choosing between them

| | Rolling Update | Recreate |
|---|---|---|
| Downtime | None (with correct readiness probes) | Yes — full stop, then start |
| Resource overhead during deploy | Extra (surge capacity) | None extra |
| Old/new coexistence | Yes, temporarily | Never |
| Right for | Stateless, backward-compatible changes | Breaking schema changes, singleton workloads, resource-starved environments |
| Rollback speed | Fast (reverse the same rolling process) | Slow (full stop/start again) |

The single most important interview-relevant point: **Rolling Update is not automatically "safe" for every kind of change** — its safety guarantee is about *pod availability*, not about *application-level compatibility* between versions running concurrently. A rolling update of a breaking database migration is a classic, real production incident, not a hypothetical.

## Points to Remember

- Rolling Update runs old and new versions **simultaneously** during rollout — safe only when both versions can coexist against shared dependencies (database, cache, message schema).
- `maxSurge`/`maxUnavailable` are percentages (or absolute numbers) that get rounded for small replica counts — check the actual numbers for low-replica-count Deployments, don't assume the percentage behaves smoothly.
- A rollout only progresses based on `readinessProbe` passing — a probe that lies (returns healthy before real initialization is done) breaks the entire safety model, either causing traffic to hit an unready pod or old pods being removed too fast.
- Recreate accepts full downtime specifically to guarantee zero version overlap — the right choice for breaking migrations or singleton workloads, not a "worse" default to avoid on principle.
- Rolling Update's rollback is just another rolling update in reverse (fast); Recreate's rollback repeats the full stop-then-start cycle (slow, another downtime window).

## Common Mistakes

- Assuming Rolling Update is inherently "zero risk" and using it for a deployment paired with a non-backward-compatible database migration — producing a window where old code hits new schema (or vice versa) and errors/corrupts data.
- Writing a readiness probe that just checks "is the process running" (e.g., a TCP port check) instead of "is the application actually ready to serve traffic correctly" (e.g., DB connection established, caches warmed) — defeats the entire point of gating the rollout on readiness.
- Leaving `maxUnavailable` at a default/high percentage on a low-replica-count Deployment (e.g., 2 replicas, 25% unavailable rounds to 0 or 1) without checking what that actually means for availability during a rollout.
- Reaching for Recreate "because it's simpler to reason about" on a stateless app that has no actual compatibility problem — needlessly introducing planned downtime that Rolling Update would have avoided for free.
- Not setting resource requests/limits correctly, so `maxSurge`'s extra pods during a rolling update can't actually schedule (insufficient cluster capacity), silently stalling the rollout instead of failing loudly.
