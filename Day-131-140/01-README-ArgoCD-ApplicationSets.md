# Day 131-140 — GitOps at Scale I: ArgoCD ApplicationSets in Depth

**Phase:** 4 – Advanced/Specialization | **Week:** W23-W24 | **Domain:** Advanced | **Flag:** —

## Brief

A single ArgoCD `Application` resource maps one Git source to one destination — fine for 5 apps, unmanageable for 50+ clusters or dozens of tenant namespaces, because you'd be hand-writing (and hand-updating) one YAML file per target forever. The **ApplicationSet controller** solves this by templating `Application` resources from a **generator** — a data source that produces a list of parameters, which get substituted into an `Application` template. This is the single mechanism that makes ArgoCD viable at fleet scale: instead of managing N `Application` objects, you manage 1 `ApplicationSet` and N becomes a side effect of whatever the generator discovers (a cluster list, a directory listing, a Git branch, an API response).

This block is split into three files:

1. **This file** — ApplicationSet generators in depth (List/Cluster/Git/Matrix/SCM Provider), templating, and patterns for managing 50+ clusters.
2. **[02-README-Fleet-Drift-Progressive-Delivery.md](02-README-Fleet-Drift-Progressive-Delivery.md)** — Rancher Fleet as an alternative/complement, drift detection mechanics, and Argo Rollouts progressive delivery across many clusters.
3. **[03-README-GitOps-Security-Multi-Tenancy.md](03-README-GitOps-Security-Multi-Tenancy.md)** — repo access control as your deployment authorization model, ArgoCD Projects/RBAC for multi-tenancy.

The hands-on build (an ApplicationSet managing 5 namespaces as tenants from one repo, plus extensions) lives in [LAB.md](LAB.md).

## The core mental model: generator → params → template → Applications

An `ApplicationSet` has exactly two parts:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: example
  namespace: argocd
spec:
  generators:
    - list: {}          # <- produces a list of {{ key: value }} parameter sets
  template:              # <- one Application is rendered PER parameter set
    metadata:
      name: '{{name}}'
    spec: {}
```

The controller runs the generator(s), gets back a list of parameter dictionaries, and renders the `template` once per entry, substituting `{{key}}` placeholders. It then reconciles the *set* of rendered `Application` objects against what actually exists in the cluster — creating missing ones, updating changed ones, and (if `syncPolicy.applicationsSync` / the `preserveResourcesOnDeletion` setting isn't overridden) **deleting Applications whose generator entry disappeared**. This last point matters operationally: removing a cluster from your generator's source of truth doesn't just stop management, it can delete the corresponding `Application` (and, depending on that Application's own `syncPolicy`, cascade-delete its deployed resources) — always check `preserveResourcesOnDeletion` before pointing an ApplicationSet at a generator whose list can shrink.

## Generator 1 — List

The simplest generator: a hardcoded (or externally-templated) array of key/value pairs. Good for a small, stable, manually-curated set of targets — exactly the "5 namespaces as tenants" hands-on activity for this block.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenant-apps
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - tenant: team-alpha
            namespace: tenant-team-alpha
          - tenant: team-beta
            namespace: tenant-team-beta
          - tenant: team-gamma
            namespace: tenant-team-gamma
  template:
    metadata:
      name: '{{tenant}}-app'
    spec:
      project: '{{tenant}}'
      source:
        repoURL: https://github.com/org/gitops-tenants.git
        targetRevision: main
        path: 'tenants/{{tenant}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

Every `{{tenant}}` / `{{namespace}}` in the template gets substituted per list element, producing 3 independent `Application` objects from one manifest. Adding a 4th tenant means adding one more `elements` entry — no new `Application` file.

## Generator 2 — Cluster

Reads ArgoCD's own registered-cluster **Secrets** (the ones created by `argocd cluster add` or a GitOps-managed cluster-registration process) and produces one parameter set per registered cluster, exposing `{{name}}`, `{{server}}`, and any labels/annotations on the cluster Secret as `{{metadata.labels.<key>}}`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: platform-agent-fleet
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: production
            tier: edge
  template:
    metadata:
      name: 'platform-agent-{{name}}'
    spec:
      project: platform
      source:
        repoURL: https://github.com/org/platform-agent.git
        targetRevision: main
        path: manifests/
      destination:
        server: '{{server}}'
        namespace: platform-agent
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

This is the workhorse generator for **50+ cluster fleets**: register every cluster once (with meaningful labels — `env`, `region`, `tier`, `provider`), then every ApplicationSet just filters by `selector.matchLabels` instead of listing clusters by name. Adding cluster #51 to the fleet means one `argocd cluster add` (or a GitOps-managed cluster Secret) with the right labels — every existing ApplicationSet whose selector matches picks it up automatically on the next reconciliation, with zero changes to the ApplicationSet manifests themselves.

## Generator 3 — Git (files and directories)

Reads structure out of a Git repository at generation time — either a **directory** generator (one param set per matching directory) or a **files** generator (one param set per JSON/YAML file's *contents*, merged with the file's path).

```yaml
# Directory generator — auto-discover a new tenant by adding a folder
generators:
  - git:
      repoURL: https://github.com/org/gitops-tenants.git
      revision: main
      directories:
        - path: tenants/*
        - path: tenants/_shared     # exclude a shared/common folder
          exclude: true
```
Each matched directory yields `{{path}}`, `{{path.basename}}`, `{{path.basename}}` etc. — a new tenant becomes real just by adding `tenants/team-delta/` to the repo; no one touches the ApplicationSet.

```yaml
# Files generator — richer per-tenant metadata driven from a config file
generators:
  - git:
      repoURL: https://github.com/org/gitops-tenants.git
      revision: main
      files:
        - path: "tenants/*/config.json"
```
```json
// tenants/team-delta/config.json
{ "tenant": "team-delta", "namespace": "tenant-team-delta", "replicas": 3, "region": "us-east-1" }
```
Every field in `config.json` becomes a `{{ }}` param — this is how you drive per-tenant *values* (replica counts, resource limits, region) from data instead of writing a bespoke template per tenant.

**This is exactly the Part 2 extension in this block's lab**: swap a hardcoded `list` generator for a `git: directories` generator so a brand-new tenant is onboarded by a Git commit (`mkdir tenants/team-delta && git push`), not an edit to the ApplicationSet controller's own manifest.

## Generator 4 — Matrix (combining generators)

**Matrix** takes 2+ generators and produces the **Cartesian product** of their outputs — the standard pattern for "deploy every app-directory to every matching cluster," which is the actual shape of the 50-cluster problem.

```yaml
generators:
  - matrix:
      generators:
        - git:
            repoURL: https://github.com/org/platform-apps.git
            revision: main
            directories:
              - path: apps/*
        - clusters:
            selector:
              matchLabels:
                fleet: production
  template:
    metadata:
      name: '{{path.basename}}-{{name}}'
    spec:
      project: platform
      source:
        repoURL: https://github.com/org/platform-apps.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: '{{server}}'
        namespace: '{{path.basename}}'
```
If `apps/*` matches 8 app directories and the cluster selector matches 50 clusters, this single ApplicationSet renders **400 Applications** — every app on every matching cluster — and stays at exactly one manifest to maintain regardless of fleet size. Matrix generators nest up to 2 levels deep (a matrix can itself be one leg of another matrix) but avoid going deeper than that; the parameter surface becomes unreadable and debugging "why didn't cluster X get app Y" gets much harder.

## Generator 5 — SCM Provider (and Pull Request generator)

Discovers repositories directly from a source-code host (GitHub/GitLab/Gitea/Bitbucket/AzureDevOps API) rather than from paths inside one repo — one parameter set per matching *repository*.

```yaml
generators:
  - scmProvider:
      github:
        organization: acme-platform-teams
        tokenRef:
          secretName: github-scm-token
          key: token
      filters:
        - repositoryMatch: '^svc-.*'
          labelMatch: 'gitops-managed'
```
Use this when tenants/teams each own their own repo (polyrepo-per-team) rather than a shared monorepo with per-tenant directories — new team repos matching the filter become new `Application`s automatically. The related **Pull Request generator** does the same discovery scoped to *open PRs* against one repo — its main use is spinning up ephemeral preview-environment Applications per open PR, torn down automatically when the PR closes.

## Templating: `{{ }}` vs `{{ }}` (go template) mode

By default ApplicationSet uses simple `fasttemplate` substitution (`{{key}}`). Setting `spec.goTemplate: true` switches to Go's `text/template` engine, which unlocks conditionals, loops, and functions (`{{ if eq .cluster.labels.env "prod" }}...{{ end }}`) — required once your per-cluster/per-tenant logic goes beyond flat substitution (e.g., "only add this extra sync option if the tenant's tier label is `premium`"). Field access syntax also changes under `goTemplate: true` — nested fields become `{{ .metadata.labels.env }}` instead of `{{metadata.labels.env}}` — so pick one mode per ApplicationSet and don't mix syntaxes.

## Points to Remember

- ApplicationSet = generator(s) + template; the generator's output list drives how many `Application` objects get rendered — this is the whole scaling mechanism.
- **Cluster generator + label selectors** is the standard pattern at 50+ clusters: register clusters once with meaningful labels, then every ApplicationSet filters by selector instead of naming clusters.
- **Matrix** generators produce a Cartesian product — `N` apps × `M` clusters = `N×M` Applications from one manifest; this is how "every app on every matching cluster" stays maintainable.
- **Git directory/files generators** turn "onboard a new tenant" into a Git commit instead of an edit to the ApplicationSet controller manifest — this is the difference between GitOps-native scaling and still hand-maintaining a control list.
- Removing an entry from a generator's source (deregistering a cluster, deleting a tenant directory) can **delete** the rendered `Application` and cascade to its deployed resources — check `preserveResourcesOnDeletion` before relying on a shrinking generator list.
- `goTemplate: true` is required the moment your templating needs conditionals/loops, not just flat key substitution — decide this up front, mixing syntaxes across the same ApplicationSet is a common source of "why isn't this substituting" bugs.

## Common Mistakes

- Hand-writing one `Application` per cluster/tenant "because it's simpler to understand," then having 50+ near-identical YAML files drift subtly out of sync with each other over time — the entire point of ApplicationSet is that this class of drift becomes structurally impossible.
- Forgetting that a generator list shrinking (a cluster secret deleted, a directory removed) deletes the corresponding `Application`, and being surprised when deployed resources vanish along with it because `syncPolicy.automated.prune: true` was also set on the template.
- Using a `list` generator for something that should be a `git: directories` generator — a `list` generator's `elements` still needs a manual ApplicationSet edit for every new entry, which is exactly the toil the Git generator is designed to eliminate.
- Nesting `matrix` generators more than 2 levels deep, or combining `matrix` with heavy `goTemplate` conditionals, producing an ApplicationSet where nobody can predict which Applications will render without running it.
- Not labeling clusters consistently at registration time, then writing selectors that accidentally match (or miss) clusters because of inconsistent label keys (`env` vs `environment` vs `Environment`) across a 50+ cluster fleet.
