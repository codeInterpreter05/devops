# Day 28 — Helm & Kustomize: Helm Fundamentals

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** (none)

## Brief

Raw YAML manifests get unwieldy fast once you need the same application deployed across dev/staging/prod with different replica counts, resource limits, and hostnames — copy-pasting and hand-editing YAML across environments is exactly how config drift and "it worked in staging" incidents happen. **Helm** is Kubernetes' de facto package manager: it templates YAML with a values file per environment, tracks releases as first-class objects you can install/upgrade/rollback, and packages complex multi-object applications (a chart) as a single versioned artifact. Nearly every real-world Kubernetes deployment pipeline touches Helm somewhere, making this foundational, practical knowledge rather than an optional extra tool.

This day is split into three files:

1. **This file** — chart structure, templating, values, and hooks.
2. **[02-README-OCI-Registries-Best-Practices.md](02-README-OCI-Registries-Best-Practices.md)** — OCI chart registries and Helm operational best practices.
3. **[03-README-Kustomize-Helm-vs-Kustomize.md](03-README-Kustomize-Helm-vs-Kustomize.md)** — Kustomize overlays/patches, and when to choose Helm vs Kustomize vs both.

## Chart anatomy

```
mychart/
├── Chart.yaml          # metadata: name, version, appVersion, dependencies
├── values.yaml           # default configuration values
├── values-dev.yaml        # environment-specific overrides (convention, not required by Helm itself)
├── values-staging.yaml
├── values-prod.yaml
├── charts/                 # vendored sub-charts (dependencies)
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl        # reusable named templates (helper functions)
│   └── NOTES.txt            # printed to the user after install/upgrade
└── .helmignore
```

`Chart.yaml` carries two distinct version fields people frequently conflate: **`version`** is the chart's own version (bump this when you change the templates/structure), and **`appVersion`** is the version of the application the chart deploys (e.g., the Docker image tag) — these are independent and can (and often do) change on different cadences.

```yaml
# Chart.yaml
apiVersion: v2
name: mychart
version: 1.4.0        # chart version
appVersion: "2.3.1"     # application version being deployed
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

## Templating: Go templates + Sprig functions

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```yaml
# values.yaml (defaults)
replicaCount: 2
image:
  repository: myapp
  tag: ""
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits: { memory: 256Mi }
```

```yaml
# values-prod.yaml (override, merged on top of values.yaml)
replicaCount: 6
resources:
  requests: { cpu: 500m, memory: 512Mi }
  limits: { memory: 1Gi }
```

Built-in objects available in every template: `.Release` (name, namespace, isInstall/isUpgrade), `.Chart` (Chart.yaml contents), `.Values` (merged values), `.Files` (access to non-template files in the chart), `.Capabilities` (cluster API versions available, useful for conditionally rendering resources only supported on newer clusters).

```bash
helm install myapp ./mychart -f values-prod.yaml
helm upgrade myapp ./mychart -f values-prod.yaml
helm template myapp ./mychart -f values-prod.yaml     # render locally WITHOUT installing — the #1 debugging tool
helm install myapp ./mychart -f values-prod.yaml --dry-run --debug   # render + validate against the live cluster
helm get values myapp                                    # see what values a running release was actually installed with
helm rollback myapp 2                                     # roll back to revision 2
helm history myapp                                          # list every revision
```

**`helm template`/`--dry-run` should be your default first debugging step** for any "my chart isn't rendering what I expect" problem — it shows you exactly the final YAML Kubernetes would receive, without touching the cluster at all.

## Helm hooks: running jobs at specific points in a release lifecycle

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./migrate.sh"]
```

Common hook points: `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-rollback`, `post-rollback`, `pre-delete`, `post-delete`. The canonical use is a **database migration Job that must complete before the new application version's pods start** — `pre-upgrade` guarantees the migration runs and (if it fails) blocks the upgrade from proceeding at all, rather than rolling out app code that expects a schema the database doesn't have yet. `hook-weight` controls ordering when multiple hooks share the same point (lower runs first); `hook-delete-policy` controls whether the hook resource is cleaned up automatically.

## Points to Remember

- `Chart.yaml`'s `version` (chart version) and `appVersion` (application version) are independent fields — don't conflate bumping one with bumping the other.
- Values merge in a defined precedence order: chart's own `values.yaml` defaults, overridden by `-f <file>` files (later files win over earlier ones), overridden by `--set` command-line flags (highest precedence).
- `helm template`/`--dry-run --debug` render the final YAML without touching the cluster — always reach for this first when a chart isn't producing the output you expect.
- Helm hooks let you run one-off Jobs/Pods at specific release lifecycle points (commonly `pre-upgrade` for DB migrations) — they are **not** regular chart resources; Helm manages their execution and (per `hook-delete-policy`) cleanup specially.
- `helm rollback` reverts a release to a previous revision's rendered manifests and values — Helm tracks release history as first-class objects (stored as Secrets in-cluster by default), which is what makes this possible without external state.

## Common Mistakes

- Editing a running release's resources directly with `kubectl edit`/`kubectl patch` instead of through `helm upgrade` — the next `helm upgrade` (even with no real changes) will silently revert the manual edit back to whatever the chart/values define, since Helm reconciles against its own tracked state, not the live cluster's current state.
- Confusing chart `version` with `appVersion` and bumping the wrong one, leading to confusing chart registry version numbers that don't reflect actual application changes (or vice versa).
- Not running `helm template`/`--dry-run` before a real `helm upgrade` in production, and discovering a templating error (a typo in a `{{ }}` expression, a missing values key) only when the upgrade actually fails mid-flight.
- Using a hook Job for something that isn't truly one-off/lifecycle-bound (e.g., putting an actual long-running service in a `post-install` hook) — hooks are meant for setup/teardown tasks, not regular application components.
- Forgetting `hook-delete-policy`, leaving completed migration Jobs accumulating in the cluster indefinitely across every upgrade, cluttering `kubectl get jobs` output and occasionally causing name-collision failures on the next release.
