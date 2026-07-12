# Day 64 — ArgoCD Deep Dive: Architecture & Application vs ApplicationSet

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** GitOps | **Flag:** ⚡ Interview-critical

## Brief

ArgoCD is the dominant open-source GitOps continuous-delivery tool for Kubernetes, and "what is GitOps, and how does ArgoCD implement pull vs push deployment" is one of the most consistently asked questions in DevOps/platform interviews right now — because GitOps is both genuinely different from traditional CI/CD push-deploys and widely (often shallowly) name-dropped on résumés. Today builds the real mental model: what ArgoCD's components actually do, and the difference between a single `Application` and the `ApplicationSet` pattern used to manage many of them at scale.

This day is split into three files:

1. **This file** — ArgoCD's three core components and the `Application`/`ApplicationSet` resources.
2. **[02-README-Sync-Policies-Waves-Hooks.md](02-README-Sync-Policies-Waves-Hooks.md)** — automated vs. manual sync policies, sync waves, and hooks.
3. **[03-README-Health-Checks-And-Notifications.md](03-README-Health-Checks-And-Notifications.md)** — health checks and the Notifications engine.

## Pull vs. push deployment — the core GitOps distinction

**Traditional CI/CD ("push" model)**: your CI pipeline, after building an artifact, runs `kubectl apply` or `helm upgrade` directly against the cluster. The pipeline needs cluster credentials, and the cluster's actual live state is only as correct as the last successful pipeline run — there's no continuous verification that the cluster still matches what's declared in Git.

**GitOps ("pull" model)**: an agent running **inside the cluster** (ArgoCD) continuously watches a Git repository containing the desired state (Kubernetes manifests, Helm charts, Kustomize overlays), and reconciles the live cluster to match it. Nothing outside the cluster needs cluster-admin credentials — the agent pulls, it's never pushed to.

Why this matters in practice:
- **No CI system needs a kubeconfig with cluster-admin.** The blast radius of a compromised CI pipeline no longer automatically includes "can modify production Kubernetes state directly."
- **Git is the single source of truth and audit log.** Every change to cluster state is a Git commit — reviewable, revertible (`git revert`), and diffable, rather than buried in CI pipeline logs.
- **Continuous reconciliation, not one-shot deploys.** If someone manually `kubectl edit`s a Deployment in prod (config drift), ArgoCD detects the live state has diverged from Git and flags (or auto-corrects) it — a traditional push pipeline has no ongoing awareness of drift after the deploy finishes.

## ArgoCD's three core components

```
Git Repo (desired state) ---> [repo-server] ---> [application-controller] ---> Kubernetes cluster (live state)
                                                          ^
                                                          |
                                                     [api-server] <---> CLI / UI / SSO users
```

- **`repo-server`** — clones and caches the Git repository, and renders the actual Kubernetes manifests from whatever templating tool is used (plain YAML, Helm, Kustomize, or a plugin). It does not talk to the cluster at all — its only job is "given this Git ref, produce the final manifests."
- **`application-controller`** — the actual reconciliation engine. It continuously compares the live cluster state (via the Kubernetes API) against the desired state (rendered manifests from `repo-server`), computes the diff, and either reports "OutOfSync" or actively applies changes to converge them, depending on the sync policy (file 2). This is the component doing the real "pull" work.
- **`api-server`** — the gRPC/REST API and gateway that the CLI (`argocd`), Web UI, and SSO/RBAC layer all talk to. It's the only component external users/tools interact with directly; it in turn talks to the controller and repo-server internally.

This separation matters operationally: `repo-server` scaling addresses "rendering is slow for huge Helm charts," `application-controller` scaling (sharding across multiple controller replicas, each owning a subset of `Application` resources) addresses "we manage thousands of Applications and reconciliation is falling behind," and `api-server` scaling addresses "too many concurrent UI/CLI/webhook users." Knowing which component is the bottleneck is a real operational skill, not trivia.

## The `Application` custom resource

Everything ArgoCD manages is represented as an `Application` — a Kubernetes Custom Resource (CRD) that itself lives in Git-manageable YAML (the classic "app of apps" pattern: ArgoCD managing its own `Application` definitions via GitOps too).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-service-manifests.git
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-service-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- **`source`** — where the desired state lives (repo, ref/branch/tag, and path within it).
- **`destination`** — which cluster (`server`, can be a remote cluster ArgoCD is registered against) and namespace to apply it to.
- **`syncPolicy`** — automated vs. manual, and whether to `prune` (delete resources removed from Git) and `selfHeal` (revert manual cluster drift) — covered fully in file 2.

## `ApplicationSet` — managing many `Application`s from one template

Manually writing one `Application` YAML per environment/cluster/microservice doesn't scale. `ApplicationSet` is a controller that **generates** many `Application` resources from a single template plus a **generator** (the source of "what varies").

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-service-envs
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: staging
            cluster: https://staging-cluster.example.com
          - env: production
            cluster: https://prod-cluster.example.com
  template:
    metadata:
      name: 'my-service-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/my-service-manifests.git
        targetRevision: main
        path: 'k8s/overlays/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: 'my-service-{{env}}'
      syncPolicy:
        automated: {}
```

Common generator types:
- **`list`** — a static, explicit list you maintain by hand (shown above) — simplest, good for a small fixed set of environments.
- **`cluster`** — automatically generates one `Application` per Kubernetes cluster ArgoCD knows about (via its cluster Secrets) — ideal for "deploy this same app to every registered cluster."
- **`git` (directory or file generator)** — scans a Git repo's directory structure (e.g., one folder per microservice under `apps/`) or a set of config files, and generates an `Application` per match — the standard pattern for "one `Application` per microservice, discovered automatically as new folders are added," so onboarding a new service is just adding a directory, not writing new ArgoCD YAML by hand.
- **Matrix generator** — combines two generators (e.g., `list` × `cluster`) to produce the cartesian product, for "deploy every app to every cluster" style fan-out.

## Points to Remember

- GitOps's defining property is **pull, not push**: an in-cluster agent reconciles toward Git, so no external system needs standing cluster-admin credentials.
- `repo-server` renders manifests (no cluster access); `application-controller` diffs and reconciles live vs. desired state (the actual GitOps engine); `api-server` is the single external-facing gateway for CLI/UI/RBAC.
- An `Application` is itself a Kubernetes CRD — meaning ArgoCD's own configuration can (and typically should) be managed via GitOps too ("app of apps").
- `ApplicationSet` solves the "N environments/clusters/microservices" scaling problem via generators (`list`, `cluster`, `git`, matrix) instead of hand-maintained `Application` YAML per instance.
- The `git` directory generator is the standard pattern for "new microservice folder appears -> ArgoCD automatically creates and manages its `Application`" without manual ArgoCD configuration per service.

## Common Mistakes

- Describing GitOps as "just CI/CD but using Git" without being able to articulate the pull-vs-push distinction and why it changes the credential/blast-radius model — this is usually the exact gap an interviewer is probing for.
- Assuming `repo-server` talks to the Kubernetes cluster — it doesn't; confusing its role with `application-controller`'s leads to misdiagnosing which component is actually slow/failing during an incident.
- Manually maintaining dozens of near-identical `Application` YAML files instead of adopting `ApplicationSet` once the number of environments/services grows past a handful — a maintenance burden `ApplicationSet` exists specifically to remove.
- Using a `list` generator for something that should be a `git` directory generator (e.g., microservices that are added frequently) — every new service then requires a manual `ApplicationSet` edit instead of "just add a folder."
- Forgetting that `Application` resources are themselves cluster objects with RBAC implications — anyone who can create/edit `Application` CRs in the `argocd` namespace can effectively point ArgoCD at an arbitrary Git repo and have it deployed with the controller's permissions.
