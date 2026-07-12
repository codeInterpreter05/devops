# Day 28 — Cheatsheet: Helm & Kustomize

## Helm — chart lifecycle

```bash
helm create mychart                        # scaffold a new chart
helm lint ./mychart                          # validate chart structure/templates
helm template rel ./mychart -f values.yaml    # render locally, no cluster touch — debug FIRST
helm install rel ./mychart -f values-prod.yaml --atomic --timeout 5m
helm upgrade rel ./mychart -f values-prod.yaml --atomic --timeout 5m
helm upgrade rel ./mychart --dry-run --debug   # validate against live cluster without applying
helm rollback rel 2                            # revert to revision 2
helm history rel                                # list all revisions
helm get values rel                              # values a running release was installed with
helm get manifest rel                             # rendered YAML currently applied for a release
helm uninstall rel
```

## Chart structure

```
mychart/
├── Chart.yaml         # name, version (chart), appVersion (app), dependencies
├── values.yaml          # defaults
├── charts/                # vendored sub-chart dependencies
└── templates/
    ├── deployment.yaml
    ├── _helpers.tpl        # reusable named templates
    └── NOTES.txt
```

## Template syntax essentials

```
{{ .Values.replicaCount }}
{{ .Release.Name }} / .Release.Namespace / .Release.IsUpgrade
{{ .Chart.Name }} / .Chart.Version / .Chart.AppVersion
{{ .Values.image.tag | default .Chart.AppVersion }}
{{- toYaml .Values.resources | nindent 12 }}
{{- if .Values.ingress.enabled }} ... {{- end }}
```

## Helm hooks

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```
Hook points: `pre-install post-install pre-upgrade post-upgrade pre-rollback post-rollback pre-delete post-delete`

## OCI registries

```bash
helm registry login <registry-host>
helm package ./mychart                                # -> mychart-1.4.0.tgz
helm push mychart-1.4.0.tgz oci://<registry>/helm-charts
helm pull oci://<registry>/helm-charts/mychart --version 1.4.0
helm install rel oci://<registry>/helm-charts/mychart --version 1.4.0
```

## Kustomize

```bash
kubectl kustomize overlays/prod                  # render only
kubectl apply -k overlays/prod                     # render + apply
kubectl diff -k overlays/prod                       # preview change vs live cluster
kustomize build overlays/prod                        # standalone binary equivalent of kubectl kustomize
```

```yaml
# base/kustomization.yaml
resources: [deployment.yaml, service.yaml]
```

```yaml
# overlays/prod/kustomization.yaml
resources: [../../base]
namePrefix: prod-
commonLabels: { environment: prod }
images: [{ name: myapp, newTag: v2.3.1 }]
patches:
  - path: replica-patch.yaml
    target: { kind: Deployment, name: myapp }
```

```yaml
# JSON 6902 patch alternative
patches:
  - target: { kind: Deployment, name: myapp }
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 6
```

## Helm vs Kustomize — quick decision

```
Templating vs patching plain YAML       -> Helm vs Kustomize
Need packaging/versioning/distribution   -> Helm
Need dependency management (sub-charts)  -> Helm
Built into kubectl, zero extra binary     -> Kustomize
Consuming 3rd-party charts (redis, etc.)  -> Helm
Simple per-env tweaks to YAML you own       -> Kustomize
GitOps (ArgoCD/Flux) rendering pipeline      -> often BOTH (helm template | kustomize patches)
```
