# Day 23 — Workloads: Deployments & ReplicaSets

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

`Deployment` is the workload object you'll use for the vast majority of stateless services in your career — APIs, web frontends, workers that don't care which replica handles a request. Understanding *how* a Deployment achieves rolling updates and rollback — via a chain of ReplicaSets it owns and manages, not through some special "update" mechanism — is what separates "I click the button in a CI pipeline and it deploys" from actually understanding what you'd debug when a rollout hangs at 2am.

This day is split into three files:

1. **This file** — Deployments, the ReplicaSet relationship, rolling updates, and rollback.
2. **[02-README-StatefulSets.md](02-README-StatefulSets.md)** — StatefulSets: ordered deployment and stable network identity.
3. **[03-README-DaemonSets-Lifecycle-Hooks.md](03-README-DaemonSets-Lifecycle-Hooks.md)** — DaemonSets and Pod lifecycle hooks (`postStart`/`preStop`).

## The ownership chain: Deployment → ReplicaSet → Pod

A `Deployment` never creates or manages Pods directly. It manages **ReplicaSets**, and each ReplicaSet manages Pods. This indirection is the entire mechanism behind rolling updates:

```
Deployment (desired state, strategy, history)
   └── ReplicaSet v1 (old template hash)  — scaled to 0 after rollout completes
   └── ReplicaSet v2 (new template hash)  — scaled to N (current)
          └── Pod, Pod, Pod ...
```

Every ReplicaSet is stamped with a **pod-template-hash** label derived from its pod template. When you change a Deployment's `spec.template` (e.g., bump the image tag), the Deployment controller:

1. Computes a new hash for the new template.
2. Creates a **new ReplicaSet** with that hash (starting at 0 replicas), rather than mutating the existing one.
3. Gradually scales the new ReplicaSet up and the old one down, according to the rollout strategy.
4. Leaves the old ReplicaSet around (scaled to 0) — this is exactly what makes `kubectl rollout undo` fast: rolling back doesn't recreate anything, it just scales the *previous* ReplicaSet back up and the current one down.

```bash
kubectl get rs -l app=myapp                       # see every ReplicaSet a Deployment has ever created
kubectl describe deployment myapp | grep -A3 OldReplicaSets
```

## Rolling updates: `RollingUpdate` strategy mechanics

```yaml
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # at most 1 pod below `replicas` during the rollout
      maxSurge: 1           # at most 1 pod above `replicas` during the rollout
  minReadySeconds: 10        # a new pod must stay Ready this long before counting toward "available"
```

- `maxUnavailable` caps how far capacity can dip during the rollout (as an absolute number or `%`).
- `maxSurge` caps how many *extra* pods can exist temporarily above the desired count while the new version comes up alongside the old one.
- With the defaults (both 25%), a rollout of a 4-replica Deployment can briefly run 5 pods (1 surge) while never dropping below 3 (1 unavailable) — this is why rolling updates don't require you to over-provision capacity manually.
- The rollout only proceeds to replace more old pods once new pods report `Ready` (pass their readiness probe) **and** stay ready for `minReadySeconds`. **This is why a missing or too-lenient readiness probe is the #1 cause of a rollout that "completes" but immediately serves errors** — Kubernetes has no way to know the new version is actually broken if you don't tell it via a probe.

```bash
kubectl set image deployment/myapp myapp=myapp:v2
kubectl rollout status deployment/myapp             # blocks until rollout finishes or fails
kubectl rollout history deployment/myapp             # see revision numbers
kubectl rollout history deployment/myapp --revision=3   # see exactly what changed in that revision
kubectl rollout undo deployment/myapp                  # roll back to the previous revision
kubectl rollout undo deployment/myapp --to-revision=2   # roll back to a specific revision
kubectl rollout pause deployment/myapp                  # freeze mid-rollout (e.g., to batch several changes)
kubectl rollout resume deployment/myapp
```

`revisionHistoryLimit` (default 10) controls how many old ReplicaSets are retained for rollback purposes — set it deliberately; too low and you lose rollback targets, too high and you accumulate clutter (though scaled-to-0 ReplicaSets cost almost nothing).

## Other update strategies and why `RollingUpdate` is the default

- **`Recreate`**: kills all old pods first, then creates new ones. Guarantees no two versions run simultaneously (useful when your app can't tolerate two schema-incompatible versions serving traffic at once) but causes downtime — there's a window with zero replicas.
- **Blue/Green and Canary** are *not* native Deployment strategies — they're patterns you build on top of Deployments (two parallel Deployments + a Service cutover, or a service mesh/Ingress traffic-splitting layer like Argo Rollouts/Flagger). Vanilla Kubernetes Deployments only give you `RollingUpdate` and `Recreate`.

## Points to Remember

- A Deployment manages ReplicaSets, not Pods directly — a rolling update works by creating a *new* ReplicaSet and shifting replica counts between old and new, never by mutating pods in place.
- `maxUnavailable`/`maxSurge` control the shape of the rollout (how much capacity dips vs. how much extra is allowed), not whether it happens safely — safety comes from readiness probes gating progression.
- `kubectl rollout undo` is fast and cheap because the old ReplicaSet (and its pod template) is still sitting there scaled to 0 — rollback is a scale operation, not a rebuild.
- `Recreate` trades availability for the guarantee that old and new versions never run concurrently — use it only when your app truly can't handle mixed versions (e.g., an in-place schema migration with no backward compatibility).
- True blue/green and canary deployments require additional tooling (Argo Rollouts, Flagger, service mesh traffic splitting, or manual dual-Deployment + Service switch) — they aren't a `strategy.type` option on a plain Deployment.

## Common Mistakes

- Shipping a Deployment with no readiness probe (or a readiness probe that just checks "process is up," not "app can actually serve requests") — the rollout will report success while serving broken responses, because Kubernetes has no signal to gate on besides what you give it.
- Setting `revisionHistoryLimit: 0` (or a very low number) and then being unable to `rollout undo` past one step because the old ReplicaSets were garbage collected.
- Assuming `kubectl rollout undo` re-pulls and redeploys the old image from scratch — it doesn't; it's just a ReplicaSet scale-up/down, which is why it's fast, and also why it won't help if the problem was external (e.g., a downstream dependency), not the app version itself.
- Using `Recreate` "just to be safe" for a normal stateless web app — this introduces unnecessary downtime for services that would have rolled fine with `RollingUpdate`.
- Editing a ReplicaSet directly instead of the Deployment — the Deployment controller will notice the drift and reconcile the ReplicaSet back, silently undoing your manual change, which looks like "Kubernetes randomly reverted my edit."
