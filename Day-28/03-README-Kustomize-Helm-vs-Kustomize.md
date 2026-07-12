# Day 28 — Helm & Kustomize: Kustomize, and Helm vs Kustomize

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** (none)

## Brief

Kustomize takes a philosophically opposite approach to Helm's templating model: instead of parameterizing YAML with `{{ }}` placeholders, you write **plain, valid YAML** for a base configuration, then declaratively **patch** it per environment — no templating language at all. It ships built into `kubectl` (`kubectl apply -k`), making it a zero-extra-dependency option, and understanding *why* you'd reach for it instead of (or alongside) Helm is a very commonly asked practical Kubernetes tooling question.

## Base and overlays: the core Kustomize model

```
myapp/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

```yaml
# base/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# base/deployment.yaml — plain, valid, standalone YAML, no template syntax
apiVersion: apps/v1
kind: Deployment
metadata: { name: myapp }
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          resources:
            requests: { cpu: 100m, memory: 128Mi }
```

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base
namePrefix: prod-
commonLabels:
  environment: prod
images:
  - name: myapp
    newTag: v2.3.1
patches:
  - path: replica-patch.yaml
    target: { kind: Deployment, name: myapp }
```

```yaml
# overlays/prod/replica-patch.yaml — strategic merge patch
apiVersion: apps/v1
kind: Deployment
metadata: { name: myapp }
spec:
  replicas: 6
  template:
    spec:
      containers:
        - name: myapp
          resources:
            requests: { cpu: 500m, memory: 512Mi }
```

```bash
kubectl kustomize overlays/prod                 # render final YAML, don't apply
kubectl apply -k overlays/prod                    # render AND apply directly
kustomize build overlays/prod | kubectl diff -f -   # preview exactly what would change against the live cluster
```

## Patch types: strategic merge vs JSON 6902

Kustomize supports two patch mechanisms:
- **Strategic merge patch** (shown above) — YAML that looks like a partial version of the target resource; Kustomize merges it in using Kubernetes' own field-merge semantics (arrays keyed by name/type where applicable, not blindly overwritten).
- **JSON 6902 patch** — an explicit, path-based patch operation list, useful for precise operations a strategic merge can't express cleanly (e.g., deleting a specific array element by index, renaming a field):
  ```yaml
  patches:
    - target: { kind: Deployment, name: myapp }
      patch: |-
        - op: replace
          path: /spec/replicas
          value: 6
        - op: remove
          path: /spec/template/spec/containers/0/env/2
  ```

`components` (a newer Kustomize feature) let you factor out reusable partial configuration (e.g., "add a sidecar container") that multiple overlays can opt into, without duplicating patch logic across every environment overlay.

## Helm vs Kustomize vs both: making the actual decision

| | Helm | Kustomize |
|---|---|---|
| Config mechanism | Templating (`{{ }}`, Go templates + Sprig) | Patching plain YAML (strategic merge / JSON 6902) |
| Packaging/distribution | Yes — versioned charts, OCI/chart-repo distribution, dependencies | No — just a directory structure you version in git yourself |
| Release tracking (history, rollback) | Yes — built-in (`helm rollback`, `helm history`) | No — relies on your own GitOps/CI history (git log, ArgoCD history) |
| Learning curve for authors | Higher — templating logic can get complex (nested conditionals, helpers) | Lower — you're mostly writing/reading plain Kubernetes YAML |
| Best fit | Distributing reusable, configurable applications (your own services shipped to multiple teams/environments, or consuming third-party charts like `bitnami/redis`) | Customizing YAML you own per-environment without needing packaging/distribution, especially in GitOps setups |
| Built into kubectl | No (separate binary) | Yes (`kubectl apply -k`, `kubectl kustomize`) |

**A very common, pragmatic real-world combination**: consume third-party software as Helm charts (databases, ingress controllers, cert-manager — things with genuine packaging/versioning/dependency needs, per file 1/2) and use **Kustomize on top of `helm template` output** for your *own* application's environment-specific tweaks, especially inside GitOps tools like ArgoCD/Flux that natively support rendering a Helm chart and then applying Kustomize patches to the result — combining Helm's packaging story with Kustomize's simpler, template-free per-environment patching.

```bash
# The "both" pattern
helm template myapp ./mychart -f values.yaml > base/deployment-rendered.yaml
kubectl apply -k overlays/prod    # overlay applies environment patches on top of the rendered Helm output
```

## Points to Remember

- Kustomize has no templating language at all — every base file is plain, valid Kubernetes YAML; environment differences are expressed as declarative patches layered on top via overlays.
- Kustomize is built into `kubectl` (`-k` flag) — no extra binary/dependency required, unlike Helm.
- Helm provides packaging, versioning, distribution (including OCI, file 2), and built-in release/rollback tracking that Kustomize deliberately does not attempt to provide.
- Strategic merge patches merge YAML using Kubernetes' own field semantics; JSON 6902 patches give precise, path-based operations for edits a strategic merge can't express (like deleting one array element).
- Combining both — Helm for packaging/distributing (especially third-party charts), Kustomize for final per-environment tweaks (especially in GitOps pipelines) — is a common, pragmatic pattern, not an either/or requirement.

## Common Mistakes

- Reaching for Helm's full templating complexity to manage what's really just "the same YAML with three fields different per environment" — Kustomize's patch model is simpler and lower-maintenance for that exact use case.
- Trying to use Kustomize to package and distribute a reusable application to other teams/orgs — Kustomize has no chart-versioning, dependency-management, or distribution story at all; that's specifically what Helm is for.
- Forgetting that Kustomize's strategic merge patches for **containers arrays** must match on the container `name` field to merge correctly — a typo'd or mismatched `name` in a patch silently creates an unintended second container instead of patching the existing one.
- Assuming `kubectl apply -k` and `kubectl diff -k`/`kustomize build | kubectl diff -f -` behave identically — always render and diff before applying an overlay change in a shared/production cluster, exactly as you would with `helm template`/`--dry-run`.
- Not version-controlling Kustomize overlays with the same rigor as Helm charts (since there's no packaged artifact/version number to point at) — without git tagging/branching discipline, it's easy to lose track of which overlay state was actually deployed to which environment at a given point in time.
