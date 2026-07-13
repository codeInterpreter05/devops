# Day 131-140 — Cheatsheet: GitOps at Scale

## ApplicationSet generator syntax quick reference

```yaml
# List — hardcoded elements
generators:
  - list:
      elements:
        - tenant: team-alpha
          namespace: tenant-team-alpha

# Cluster — one entry per registered cluster Secret, filterable by label
generators:
  - clusters:
      selector:
        matchLabels:
          env: production
      values:                       # extra static params merged into every result
        customLabel: platform-fleet

# Git — directories (one entry per matched directory)
generators:
  - git:
      repoURL: https://github.com/org/repo.git
      revision: main
      directories:
        - path: tenants/*
        - path: tenants/_shared
          exclude: true

# Git — files (one entry per matched file's parsed contents)
generators:
  - git:
      repoURL: https://github.com/org/repo.git
      revision: main
      files:
        - path: "tenants/*/config.json"

# Matrix — Cartesian product of 2 child generators
generators:
  - matrix:
      generators:
        - git: { repoURL: "...", revision: main, directories: [{ path: apps/* }] }
        - clusters: { selector: { matchLabels: { fleet: production } } }

# SCM Provider — one entry per matched repository
generators:
  - scmProvider:
      github:
        organization: my-org
        tokenRef: { secretName: github-scm-token, key: token }
      filters:
        - repositoryMatch: '^svc-.*'

# Pull Request — one entry per open PR (preview environments)
generators:
  - pullRequest:
      github:
        owner: my-org
        repo: my-service
        tokenRef: { secretName: github-scm-token, key: token }
```

Common template fields: `{{name}}` / `{{server}}` (Cluster), `{{path}}` / `{{path.basename}}` (Git directories), any JSON/YAML key (Git files), `{{metadata.labels.<key>}}` (Cluster/SCM). Under `spec.goTemplate: true`, switch to `{{ .name }}` / `{{ .path.basename }}` dot-notation and unlock `{{ if }}` / `{{ range }}`.

```yaml
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
```

`RollingSync` strategy (stagger cluster waves, distinct from a Rollout's own per-cluster canary):
```yaml
spec:
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - { key: rollout-group, operator: In, values: [canary] }
          maxUpdate: 100%
        - matchExpressions:
            - { key: rollout-group, operator: In, values: [standard] }
          maxUpdate: 25%
```

## ArgoCD Project / RBAC YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-alpha
  namespace: argocd
spec:
  sourceRepos: ['https://github.com/org/repo.git']   # never '*' for a real tenant
  destinations:
    - { namespace: tenant-team-alpha, server: https://kubernetes.default.svc }
  clusterResourceWhitelist: []                        # deny cluster-scoped by default
  namespaceResourceBlacklist:
    - { group: '', kind: ResourceQuota }
  roles:
    - name: developer
      policies:
        - p, proj:team-alpha:developer, applications, get, team-alpha/*, allow
        - p, proj:team-alpha:developer, applications, sync, team-alpha/*, allow
      groups: [team-alpha-developers]
```

```yaml
# argocd-rbac-cm ConfigMap — instance-wide RBAC
apiVersion: v1
kind: ConfigMap
metadata: { name: argocd-rbac-cm, namespace: argocd }
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:platform-admin, applications, *, */*, allow
    p, role:platform-admin, projects, *, *, allow
    g, org-sso:platform-team, role:platform-admin
  scopes: '[groups]'
```

Policy verb reference: `get`, `create`, `update`, `delete`, `sync`, `override`, `action/*` — resources: `applications`, `applicationsets`, `projects`, `clusters`, `repositories`, `certificates`, `accounts`, `gpgkeys`.

## Argo Rollouts canary / analysis snippets

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: { duration: 2m }
        - analysis:
            templates: [{ templateName: success-rate }]
            args: [{ name: service-name, value: checkout-service }]
        - setWeight: 100
      trafficRouting:
        istio: { virtualService: { name: my-vsvc, routes: [primary] } }
        # or: nginx: {}, smi: {}, alb: {}, appmesh: {}
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate }
spec:
  args: [{ name: service-name }]
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result >= 0.95
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus.observability:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",code!~"5.."}[5m]))
            / sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

```bash
kubectl argo rollouts get rollout <name> -n <ns> --watch     # live canary progress view
kubectl argo rollouts promote <name> -n <ns>                  # manually skip current pause
kubectl argo rollouts abort <name> -n <ns>                    # manually abort/rollback
kubectl argo rollouts retry rollout <name> -n <ns>             # retry after a fixed failure
kubectl argo rollouts undo <name> -n <ns>                      # roll back to previous stable
```

## Drift-detection commands

```bash
argocd app diff <app>                     # field-level delta: live vs. Git-declared
argocd app get <app>                      # sync/health status summary
argocd app get <app> --show-operation      # last sync operation detail
argocd app sync <app>                      # manual sync (reconcile now)
argocd app sync <app> --dry-run             # preview what a sync would change
argocd appset get <appset-name> --refresh   # force generator re-evaluation now
argocd app set <app> --sync-policy automated --self-heal   # enable selfHeal via CLI
argocd app unset <app> --self-heal                          # disable selfHeal via CLI

# Prometheus metric to alert on fleet-wide drift (exposed by argocd-metrics)
argocd_app_info{sync_status="OutOfSync"}
argocd_app_info{health_status!="Healthy"}
```

```yaml
syncPolicy:
  automated:
    prune: true          # delete resources removed from Git
    selfHeal: true         # auto-revert live drift back to Git's declared state
  syncOptions:
    - CreateNamespace=true
    - ApplyOutOfSyncOnly=true    # skip resources already in sync (scale optimization)
    - PruneLast=true              # prune only after everything else applies successfully
```

## Rancher Fleet quick reference

```yaml
# fleet.yaml
defaultNamespace: my-app
helm: { chart: ./chart }
targetCustomizations:
  - name: prod
    helm: { values: { replicaCount: 5 } }
    clusterSelector: { matchLabels: { env: production } }
```

```bash
fleet get gitrepo                 # list watched Git repos
fleet get bundles                  # list compiled Bundles
fleet get bundledeployments         # per-cluster deployment status
```
