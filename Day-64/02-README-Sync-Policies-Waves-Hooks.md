# Day 64 — ArgoCD Deep Dive: Sync Policies, Sync Waves & Hooks

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** GitOps | **Flag:** ⚡ Interview-critical

## Brief

Knowing ArgoCD's architecture (file 1) tells you *what* reconciles state — sync policies, waves, and hooks tell you *how* and *in what order* that reconciliation actually happens, which matters enormously the moment your deployment isn't "just apply everything at once" (e.g., a database migration must run before the new app version, or a CRD must exist before the resources using it). This is where ArgoCD stops being "fancy `kubectl apply`" and becomes an actual deployment orchestration tool.

## Sync policies: automated vs. manual

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
  syncOptions:
    - CreateNamespace=true
    - PruneLast=true
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

- **Manual sync** (no `automated:` block) — ArgoCD detects and reports drift (`OutOfSync` status) but takes no action until a human clicks "Sync" (or runs `argocd app sync`). Safer default for production, especially early in an ArgoCD rollout when trust in the pipeline is still being established.
- **Automated sync** (`automated: {}`) — the moment the controller detects the live state differs from Git, it applies the change **without human intervention**. This is the "true" continuous-deployment end of GitOps — merge to `main`, and it's live, no separate deploy step at all.
- **`prune: true`** — resources that existed in a previous sync but have since been **removed from Git** get deleted from the cluster too. Without this, deleting a manifest from Git only stops ArgoCD from managing it going forward — the actual Kubernetes resource is orphaned in the cluster forever, a very common source of "why is this old resource still running" confusion.
- **`selfHeal: true`** — if someone manually edits a live resource (`kubectl edit`, `kubectl scale`, a well-meaning but out-of-process hotfix), ArgoCD reverts it back to match Git on the next reconciliation loop. This is the property that makes config drift structurally impossible rather than just detectable — but it also means "I'll just quickly patch this in prod to unblock an incident" silently gets reverted minutes later unless you also pause automated sync first.
- **`allowEmpty: false`** — a safety guard preventing a sync from deleting *everything* if the rendered manifest set is unexpectedly empty (e.g., a broken Helm template that renders to nothing) — without it, a rendering bug plus `prune: true` could wipe an entire application's resources.

## Sync waves — ordering resources within one sync

By default, ArgoCD applies all resources in a sync roughly together (respecting basic Kubernetes dependency ordering like namespaces before namespaced resources). **Sync waves** let you impose explicit ordering when the default isn't good enough — e.g., a CRD must exist before a Custom Resource using it, or a database migration Job must complete before the new application Deployment rolls out.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    argocd.argoproj.io/sync-wave: "2"   # runs after wave 0 and 1

---
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # runs before the Deployment

---
apiVersion: v1
kind: Namespace
metadata:
  name: my-app-ns
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # default wave, runs first
```

- Waves are just integers (default `0`) — ArgoCD applies all resources in the lowest wave first, **waits for them to be `Healthy`** (see file 3 for what "healthy" means per resource type), then proceeds to the next wave, and so on.
- Negative wave numbers are valid — useful for "this must run before literally everything else," like a `PreSync` hook (below) or a namespace/CRD that everything else depends on.
- This is the mechanism you reach for whenever "apply order matters" — schema migrations before app rollout, a ConfigMap/Secret before the Deployment that mounts it (though ArgoCD's default ordering usually handles this specific case already), or a CRD before instances of that CRD.

## Hooks — running actions at specific points in the sync lifecycle

Hooks let you run a Job (or other resource) at a specific **phase** of the sync process, independent of the regular wave-ordered resource set:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: my-app:latest
          command: ['./migrate.sh']
      restartPolicy: Never
```

| Hook phase | Runs |
|---|---|
| `PreSync` | Before the sync applies any resources — classic use: database migrations that must complete before new app code that depends on the new schema starts |
| `Sync` | Interleaved with the main resource application, ordered by sync wave alongside regular resources |
| `PostSync` | After all resources are applied and healthy — classic use: smoke tests, cache warm-up, sending a deployment notification |
| `SyncFail` | Only runs if the sync fails — classic use: automated rollback triggers, incident alerting |

- **`hook-delete-policy`** controls cleanup: `HookSucceeded` deletes the hook resource (like the migration Job) once it completes successfully — keeping the cluster from accumulating a completed Job object per deploy forever; `HookFailed` / `BeforeHookCreation` are the other common policies for different cleanup timing needs.
- **Hooks vs. sync waves**: waves order the *regular* manifest set relative to each other; hooks are a separate lifecycle concept for actions tied to *before/during/after the whole sync*, not just "another resource in a particular wave" — though a `PreSync` hook and "wave -1" often achieve a similar practical outcome for simple cases, hooks give you the explicit success/failure semantics (a failed `PreSync` hook blocks the rest of the sync from proceeding at all) that a plain wave doesn't.

## Points to Remember

- Manual sync reports drift but waits for a human; automated sync applies Git state as soon as it's detected, with no separate deploy step — the actual "continuous" part of continuous deployment.
- `prune: true` deletes resources removed from Git; without it, deleted manifests leave orphaned resources running in the cluster indefinitely.
- `selfHeal: true` reverts manual cluster edits back to Git state on the next reconcile loop — great for drift prevention, but it will silently undo an emergency `kubectl` hotfix unless sync is paused first.
- Sync waves (`argocd.argoproj.io/sync-wave` annotation, integer, default 0, negative allowed) order resource application within a sync, waiting for each wave to be healthy before proceeding to the next.
- Hooks (`PreSync`/`Sync`/`PostSync`/`SyncFail`) run actions tied to the sync lifecycle itself (migrations, smoke tests, rollback triggers), distinct from ordering the regular resource set via waves.

## Common Mistakes

- Enabling `prune: true` without realizing pre-existing manifests removed from Git will actually be deleted from the cluster — a surprise the first time someone deletes a YAML file "just to clean up the repo."
- Turning on `selfHeal: true` and then being confused when an emergency production hotfix applied directly via `kubectl` gets silently reverted a minute later — forgetting to pause automated sync (or use `argocd app set --sync-policy none` temporarily) during an active incident.
- Not setting `allowEmpty: false`, then having a broken Helm template render to zero resources and, combined with `prune: true`, delete the entire application from the cluster.
- Assuming sync waves alone give you "run this and wait for success/failure before continuing" semantics — waves order and wait for health, but hooks are what give you actual success/failure gating with dedicated `SyncFail` handling.
- Forgetting `hook-delete-policy`, leaving a new completed migration Job object behind after every single deploy, cluttering the namespace over time.
