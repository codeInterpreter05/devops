# Day 131-140 — GitOps at Scale III: Security and Multi-Tenancy

**Phase:** 4 – Advanced/Specialization | **Week:** W23-W24 | **Domain:** Advanced | **Flag:** —

## Brief

In a push-based deployment model, "who can deploy to production" is a question about who holds CI credentials and pipeline access. In GitOps, that question has a different, sharper answer: **your deployment authorization model *is* whoever can get a commit merged into the branch/path the GitOps operator watches.** There is no separate "deploy" permission layered on top — merging *is* deploying, deterministically, on the next reconciliation. This reframing is the single most important security concept in this block, and it's what most "just install ArgoCD" tutorials skip entirely. This file covers repo access control as that authorization model, and ArgoCD's own Projects/RBAC layer for multi-tenancy once many teams share one ArgoCD instance.

## "Who can merge to this repo" IS your deployment authorization model

Walk the causal chain: a `selfHeal`-enabled Application continuously reconciles the live cluster to match a Git path. There is no manual "click deploy" step and no separate CD approval gate outside Git — the moment a commit lands on the watched branch/path, it becomes live cluster state (typically within the reconciliation interval, or immediately via a webhook-triggered refresh). This means:

- **Branch protection rules** (require PR review, require passing CI status checks, disallow force-push, disallow direct pushes to `main`) are not a "nice to have" code-hygiene practice for a GitOps config repo — they are your **production change-control gate**, full stop. If anyone can push directly to the branch ArgoCD watches, your entire RBAC/Projects setup downstream is decorative.
- **CODEOWNERS** files scoped per directory let you enforce "only the `payments` team can approve changes under `tenants/payments/`" — this is how you implement per-tenant change authorization *inside a shared monorepo* without needing separate repos per tenant.
  ```
  # .github/CODEOWNERS in a shared GitOps config repo
  /tenants/payments/       @org/payments-team
  /tenants/checkout/       @org/checkout-team
  /clusters/prod/          @org/platform-leads     # prod changes need platform sign-off
  /shared/                 @org/platform-leads
  ```
- **Required status checks** (a policy-as-code check — OPA/Conftest/Kyverno's CLI mode running in CI against the manifests, not just at cluster admission time) catch a bad manifest *before* merge, when it's still cheap to fix, rather than relying purely on cluster-side admission control as the only backstop.
- **Deploy keys / repo credentials scoped read-only** — the ArgoCD instance itself only ever needs **read** access to the Git repo (it pulls, it never pushes). Grant ArgoCD's repo credential the narrowest possible scope (read-only deploy key, or a machine-user token with read-only repo access) — this is a direct application of least privilege, and it means even a fully compromised ArgoCD instance can't push malicious commits back into your source of truth.
- **Signed commits/tags** (`git commit -S`) plus a CI check that rejects unsigned commits on the protected branch adds non-repudiation — useful in regulated environments where "who actually authored this production change" needs to survive an audit, not just "someone with merge rights."

**The interview-critical statement to be able to say out loud:** *"In a GitOps model, repository access control (branch protection, required reviews, CODEOWNERS) replaces the traditional 'who has deploy credentials to the CD tool' question — because merging to the watched path is the deploy. If you get repo access control wrong, no amount of ArgoCD RBAC configuration downstream fixes it, because the operator will faithfully apply whatever lands in Git."*

## ArgoCD Projects — the multi-tenancy primitive

An `AppProject` scopes what any `Application` assigned to it is **allowed** to do — which source repos it may pull from, which destination clusters/namespaces it may deploy to, and which Kubernetes resource kinds it may create. This is what lets one shared ArgoCD instance safely host many tenants without a misconfigured (or malicious) Application from Team A being able to touch Team B's namespace or a cluster-scoped resource it has no business touching.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-alpha
  namespace: argocd
spec:
  description: "Team Alpha tenant project"
  sourceRepos:
    - 'https://github.com/org/gitops-tenants.git'   # only this repo, not '*'
  destinations:
    - namespace: 'tenant-team-alpha'
      server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist: []          # deny ALL cluster-scoped resources by default
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota                # tenants can't touch their own quota
    - group: ''
      kind: LimitRange
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'                          # allow everything else namespace-scoped
  roles:
    - name: developer
      description: "Team Alpha devs — sync and view only"
      policies:
        - p, proj:team-alpha:developer, applications, sync, team-alpha/*, allow
        - p, proj:team-alpha:developer, applications, get, team-alpha/*, allow
      groups:
        - org-sso:team-alpha-developers
    - name: lead
      description: "Team Alpha leads — full app management within the project"
      policies:
        - p, proj:team-alpha:lead, applications, *, team-alpha/*, allow
      groups:
        - org-sso:team-alpha-leads
```

Key fields that actually enforce tenancy boundaries:
- **`sourceRepos`** — an Application in this Project can only source from listed repos. Without this scoped down, any tenant could point their Application at an arbitrary external repo.
- **`destinations`** — the (server, namespace) pairs this Project's Applications may target. This is what stops Team Alpha's Application from being pointed at `tenant-team-beta` or a management-plane namespace.
- **`clusterResourceWhitelist`** (empty by default is the safe posture) — cluster-scoped resources (`ClusterRole`, `Namespace`, `CustomResourceDefinition`, `PersistentVolume`) are fleet-wide blast-radius risks; most tenants should not be able to create or modify them at all. Leave this empty unless a tenant has a specific, reviewed need.
- **`roles` + `policies`** — Project-scoped RBAC, mapped to SSO groups. This is what differentiates "can view and sync" from "can fully manage" within a tenant's own boundary, without granting anything outside it.

## ArgoCD instance-level RBAC — `argocd-rbac-cm`

Project roles handle *within-project* permissions. Instance-wide RBAC (who can create Projects, who's an admin, default permissions for anyone not explicitly matched) lives in the `argocd-rbac-cm` ConfigMap, using a Casbin-style policy CSV:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly    # anyone not explicitly matched gets read-only — safe default
  policy.csv: |
    p, role:platform-admin, applications, *, */*, allow
    p, role:platform-admin, projects, *, *, allow
    p, role:platform-admin, clusters, *, *, allow
    p, role:platform-admin, repositories, *, *, allow

    p, role:team-alpha-dev, applications, get, team-alpha/*, allow
    p, role:team-alpha-dev, applications, sync, team-alpha/*, allow

    g, org-sso:platform-team, role:platform-admin
    g, org-sso:team-alpha-developers, role:team-alpha-dev
  scopes: '[groups]'
```

`policy.default: role:readonly` is the load-bearing line for a safe multi-tenant default — anyone authenticated via SSO but not mapped into a specific role can look, not touch. The `g,` lines map SSO groups (from OIDC claims) to roles — production multi-tenancy at 50+ teams is only operationally sane when access is driven by SSO group membership, not by hand-maintained individual user policies that go stale the moment someone changes teams.

## Multi-tenancy topology choices

Three common ways to actually structure tenants once you have Projects/RBAC as the enforcement mechanism:

| Model | Description | Tradeoff |
|---|---|---|
| **One AppProject per tenant, shared monorepo** | All tenants' manifests live in one Git repo under per-tenant directories; CODEOWNERS scopes approval per directory; one AppProject per tenant scopes runtime blast radius | Simplest repo topology to onboard new tenants into (Git generator picks up a new directory); requires disciplined CODEOWNERS + branch protection since it's one repo |
| **One AppProject + one repo per tenant** | Each tenant owns their own Git repo entirely; ArgoCD's `sourceRepos` on their Project restricts them to just that repo | Strongest isolation (a tenant literally cannot see another tenant's manifests); more repos to template/maintain, needs an SCM Provider generator or per-tenant ApplicationSet bootstrap |
| **Shared AppProject, namespace-only isolation** | All tenants share one Project, isolated only by destination namespace | Weakest isolation — a bug or over-broad role in the shared Project affects every tenant; generally only acceptable for trusted-internal, single-org use, not for isolating genuinely separate teams/customers |

For the hands-on activity in this block (5 K8s namespaces as separate tenants from one repo), the first model — one AppProject per tenant, shared monorepo with per-tenant directories — is the pattern that matches both the ApplicationSet Git-generator approach from file 1 and real multi-tenant practice.

## Points to Remember

- **Repo access control is your deployment authorization model in GitOps** — branch protection, required reviews, and CODEOWNERS are production change-control, not code hygiene. Say this explicitly in interviews; it's the concept most candidates miss.
- ArgoCD's own Git credential should be **read-only** — the operator only ever pulls; it never needs write access to the source of truth it's reconciling from.
- `AppProject.sourceRepos` / `destinations` / `clusterResourceWhitelist` are what actually enforce tenant boundaries at the ArgoCD layer — an Application with no Project restrictions (the `default` Project) can touch anything the ArgoCD service account can touch.
- `clusterResourceWhitelist: []` (deny all cluster-scoped resources) is the safe default per tenant Project — cluster-scoped resources are fleet-wide blast radius and most tenants have no legitimate need to create them.
- `argocd-rbac-cm`'s `policy.default: role:readonly` plus SSO-group-driven `g,` mappings is how instance-wide access stays sane at many-team scale — hand-maintained per-user policies don't.
- Three tenancy topologies (shared monorepo + per-tenant Project, per-tenant repo + Project, shared Project with only namespace isolation) trade onboarding simplicity against isolation strength — know which one you're choosing and why.

## Common Mistakes

- Configuring careful ArgoCD Projects/RBAC while leaving the GitOps config repo's `main` branch open to direct pushes with no required review — the RBAC is irrelevant if anyone can bypass Git entirely and merge straight to the watched branch.
- Giving ArgoCD's repo credential write access "just in case" — unnecessary, and turns a compromised ArgoCD instance into something that can tamper with its own source of truth.
- Leaving every Application in the `default` AppProject (ArgoCD's permissive built-in project) instead of creating scoped per-tenant Projects — this is the single most common way "we have multi-tenancy" turns out to mean "we have namespaces but no actual enforced isolation."
- Setting `clusterResourceWhitelist` to `['*']`/`*` "to avoid support tickets" instead of reviewing and whitelisting the specific cluster-scoped kinds a tenant actually needs — this quietly removes the one control that limits fleet-wide blast radius per tenant.
- Managing ArgoCD RBAC via individually-granted user policies instead of SSO-group mappings — this silently rots as people change teams, and nobody notices an ex-team-member still has standing access until an audit or incident surfaces it.
- Treating CODEOWNERS as a suggestion rather than pairing it with a required-reviewers branch protection rule — GitHub/GitLab CODEOWNERS only actually blocks a merge if the branch protection rule requiring CODEOWNERS review is turned on; adding the file alone does not enforce anything.
