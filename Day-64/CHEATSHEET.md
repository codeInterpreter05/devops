# Day 64 — Cheatsheet: ArgoCD Deep Dive

## Install & login

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd port-forward svc/argocd-server 8080:443 &
PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
argocd login localhost:8080 --username admin --password "$PASS" --insecure
```

## Components (what to check when something's wrong)

```
repo-server            -> renders manifests from Git (no cluster access) — check if rendering/templating is slow or failing
application-controller -> diffs live vs desired, reconciles — check if sync/health status is stuck or stale
api-server              -> CLI/UI/RBAC gateway — check if UI/API calls are slow or auth is failing
```

## `Application` resource

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: my-app, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-prod
  syncPolicy:
    automated: { prune: true, selfHeal: true, allowEmpty: false }
    syncOptions: [CreateNamespace=true, PruneLast=true]
```

## CLI essentials

```bash
argocd app create my-app --repo <url> --path k8s --dest-server https://kubernetes.default.svc --dest-namespace my-app --sync-policy none
argocd app get my-app                  # sync + health status
argocd app sync my-app                 # manual sync
argocd app wait my-app                 # block until sync/health settle
argocd app set my-app --sync-policy automated --self-heal --auto-prune
argocd app history my-app              # list revisions
argocd app rollback my-app <ID>        # roll back to a previous revision
argocd app delete my-app --cascade     # delete app + its managed resources
argocd app diff my-app                 # show live vs desired diff
```

## `ApplicationSet` generators

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  generators:
    - list:                              # static, hand-maintained set
        elements: [{env: staging}, {env: production}]
    # - cluster: {}                       # one Application per registered cluster
    # - git:                              # scan a repo directory/files
    #     repoURL: https://github.com/org/repo.git
    #     directories: [{path: apps/*}]
    # - matrix:                           # cartesian product of two generators
    #     generators: [{list: {...}}, {cluster: {}}]
  template:
    metadata: { name: 'my-app-{{env}}' }
    spec: { ... }
```

## Sync waves

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # default 0; lower runs first; negative allowed
```

## Hooks

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync              # PreSync | Sync | PostSync | SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded   # or HookFailed, BeforeHookCreation
```

## Health status meanings

```
Healthy      -> resource is actually working (type-specific check)
Progressing  -> still rolling out
Degraded     -> actively broken
Suspended    -> paused (e.g., a suspended CronJob)
Missing      -> defined in Git, not present in cluster
Unknown      -> ArgoCD has no built-in health logic (common for custom CRDs)
```

## Custom health check (Lua, in `argocd-cm`)

```yaml
data:
  resource.customizations.health.mycompany.io_MyCR: |
    hs = {}
    if obj.status ~= nil and obj.status.phase == "Running" then
      hs.status = "Healthy"
      return hs
    end
    hs.status = "Progressing"
    return hs
```

## Notifications

```yaml
# argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token
  template.app-sync-failed: |
    message: "Sync failed for {{.app.metadata.name}}"
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
```

```yaml
# subscribe an Application to a trigger
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: platform-alerts
```
