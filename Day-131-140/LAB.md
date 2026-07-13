# Day 131-140 — Lab: GitOps at Scale Build-Out

**Goal:** Leave this 10-day block with a working ArgoCD ApplicationSet that manages 5 Kubernetes namespaces as separate tenants from one Git repo, auto-discovery of new tenants via a Git directory generator, real per-tenant RBAC enforcement via ArgoCD Projects, working drift detection with `selfHeal`, and a progressive-delivery rollout using Argo Rollouts with an analysis gate.

**Prerequisites:** A Kubernetes cluster (`kind`/`minikube` is fine — this lab doesn't require 50 real clusters, it teaches the *mechanism* you'd point at 50+), `kubectl`, `helm` v3, `argocd` CLI, a Git repo you can push to (a personal GitHub repo works — the "one repo" in the hands-on activity), and Argo Rollouts' `kubectl argo rollouts` plugin.

This lab is a single 10-day arc — do the parts in order; each builds on the previous one's resources.

---

### Setup — install ArgoCD and the Rollouts controller

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# CLI plugins
brew install argocd                     # or the release binary for your OS
kubectl krew install argo-rollouts       # or download the plugin binary directly

# Port-forward and log in
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
argocd login localhost:8080 --username admin \
  --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)" \
  --insecure
```

Create the working Git repo for this lab (referred to below as `gitops-tenants`, replace with your actual remote):

```bash
mkdir -p gitops-tenants/tenants/{team-alpha,team-beta,team-gamma,team-delta,team-epsilon}
cd gitops-tenants && git init && git remote add origin <your-repo-url>
```

---

### Part 1 — ApplicationSet with a List generator across 5 namespaces-as-tenants

This is the assigned core hands-on activity: manage 5 different namespaces as separate tenants from one repo.

1. For each of the 5 tenants, add a minimal manifest (a Deployment + Service is enough — a simple `nginx` or `httpd` placeholder is fine, the point is the ApplicationSet mechanism, not the workload):
   ```yaml
   # tenants/team-alpha/deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: tenant-app
   spec:
     replicas: 1
     selector: { matchLabels: { app: tenant-app } }
     template:
       metadata: { labels: { app: tenant-app } }
       spec:
         containers:
           - name: tenant-app
             image: nginxinc/nginx-unprivileged:latest
             ports: [{ containerPort: 8080 }]
   ```
   Repeat (or symlink the same content) for `team-beta`, `team-gamma`, `team-delta`, `team-epsilon`. Commit and push.

2. Write the ApplicationSet using a **List generator**:
   ```yaml
   # applicationset-list.yaml
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
             - tenant: team-beta
             - tenant: team-gamma
             - tenant: team-delta
             - tenant: team-epsilon
     template:
       metadata:
         name: '{{tenant}}-app'
       spec:
         project: default        # replaced with per-tenant projects in Part 3
         source:
           repoURL: <your-repo-url>
           targetRevision: main
           path: 'tenants/{{tenant}}'
         destination:
           server: https://kubernetes.default.svc
           namespace: 'tenant-{{tenant}}'
         syncPolicy:
           automated:
             prune: true
             selfHeal: true
           syncOptions:
             - CreateNamespace=true
   ```
   ```bash
   kubectl apply -f applicationset-list.yaml
   ```

3. Verify:
   ```bash
   argocd appset list
   argocd app list
   kubectl get ns | grep tenant-
   kubectl get pods -n tenant-team-alpha
   ```

**Success criteria:** `argocd app list` shows exactly 5 `Application` objects (`team-alpha-app` through `team-epsilon-app`), all `Synced`/`Healthy`, and `kubectl get ns` shows 5 `tenant-*` namespaces auto-created by `CreateNamespace=true`.

---

### Part 2 — Add a Git directory generator to auto-discover new tenants

Replace the hardcoded list with discovery, so onboarding tenant #6 is a Git commit, not an ApplicationSet edit.

1. Update the ApplicationSet's generator:
   ```yaml
   # applicationset-git.yaml (same metadata.name: tenant-apps to replace, not duplicate)
   spec:
     generators:
       - git:
           repoURL: <your-repo-url>
           revision: main
           directories:
             - path: tenants/*
     template:
       metadata:
         name: '{{path.basename}}-app'
       spec:
         project: default
         source:
           repoURL: <your-repo-url>
           targetRevision: main
           path: '{{path}}'
         destination:
           server: https://kubernetes.default.svc
           namespace: 'tenant-{{path.basename}}'
         syncPolicy:
           automated:
             prune: true
             selfHeal: true
           syncOptions:
             - CreateNamespace=true
   ```
   ```bash
   kubectl apply -f applicationset-git.yaml
   argocd app list     # should still show the same 5 Applications, now generator-discovered
   ```

2. Prove auto-discovery: add a 6th tenant directory and push, without touching the ApplicationSet again.
   ```bash
   mkdir -p tenants/team-zeta
   cp tenants/team-alpha/deployment.yaml tenants/team-zeta/
   git add tenants/team-zeta && git commit -m "onboard team-zeta" && git push
   ```
   ```bash
   sleep 200   # or trigger a manual refresh:
   argocd appset get tenant-apps --refresh
   argocd app list   # should now show 6 apps, team-zeta-app included
   ```

3. Prove decommissioning: remove `tenants/team-zeta`, push, and confirm the `team-zeta-app` Application (and its namespace's resources, since `prune: true`) is removed on the next reconciliation. **Read this before running it** — this is the exact "shrinking generator list deletes the Application" behavior called out in file 1; confirm you understand the blast radius before triggering it against anything beyond this lab.

**Success criteria:** Adding a directory and pushing produces a 6th `Application` with zero ApplicationSet edits; removing the directory removes the Application and its namespace's resources.

---

### Part 3 — Multi-tenant RBAC with ArgoCD Projects

Move each tenant out of the permissive `default` Project into its own scoped `AppProject`.

1. Create one `AppProject` per tenant (repeat for all 5-6):
   ```yaml
   # project-team-alpha.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AppProject
   metadata:
     name: team-alpha
     namespace: argocd
   spec:
     sourceRepos:
       - <your-repo-url>
     destinations:
       - namespace: 'tenant-team-alpha'
         server: 'https://kubernetes.default.svc'
     clusterResourceWhitelist: []
     roles:
       - name: developer
         policies:
           - p, proj:team-alpha:developer, applications, get, team-alpha/*, allow
           - p, proj:team-alpha:developer, applications, sync, team-alpha/*, allow
         groups:
           - team-alpha-developers
   ```
   ```bash
   kubectl apply -f project-team-alpha.yaml   # repeat per tenant
   ```

2. Update the ApplicationSet template's `project` field to derive the project name from the same path param used for the namespace, instead of hardcoding `default`:
   ```yaml
   spec:
     project: '{{path.basename}}'
   ```
   Re-apply and confirm each Application now shows its correct scoped project:
   ```bash
   argocd app get team-alpha-app -o json | jq '.spec.project'
   ```

3. Prove the boundary actually holds: attempt to point `team-alpha`'s Application at `team-beta`'s namespace by hand-editing one Application's `destination.namespace`, and confirm ArgoCD **rejects** the sync with a project-boundary violation error (`namespace 'tenant-team-beta' is not permitted in project 'team-alpha'`).
   ```bash
   argocd app set team-alpha-app --dest-namespace tenant-team-beta
   argocd app sync team-alpha-app   # expect a permission error, not a successful sync
   ```
   Revert the change afterward.

4. Add instance-level RBAC in `argocd-rbac-cm` so unmapped users default to read-only (see file 3 for the full ConfigMap), and confirm a login with no group mapping can view but not sync any Application.

**Success criteria:** Every tenant Application belongs to its own scoped `AppProject`; the cross-namespace destination test in step 3 is explicitly rejected by ArgoCD (not just "wouldn't have worked anyway" — you need to see the actual rejection); `argocd-rbac-cm`'s default policy is `role:readonly`.

---

### Part 4 — Drift detection and selfHeal at scale

1. With `selfHeal: true` already set from Part 1, manually drift one tenant's live state:
   ```bash
   kubectl scale deployment/tenant-app -n tenant-team-alpha --replicas=5
   ```
2. Immediately check status — it should show as drifted before self-heal catches it:
   ```bash
   argocd app get team-alpha-app
   argocd app diff team-alpha-app
   ```
3. Wait for the next reconciliation (or force one) and confirm ArgoCD reverts the manual scale back to the Git-declared replica count:
   ```bash
   argocd app sync team-alpha-app --dry-run    # preview what would change
   kubectl get deployment/tenant-app -n tenant-team-alpha -w    # watch it get reverted
   ```
4. Turn `selfHeal` off for one tenant (edit that Application's `syncPolicy.automated`, removing `selfHeal`), repeat the drift, and confirm this time it stays drifted (`OutOfSync`) until a manual `argocd app sync` — this demonstrates the actual difference `selfHeal` makes, rather than just taking the concept on faith.
5. Add `syncOptions: [ApplyOutOfSyncOnly=true]` to the ApplicationSet template and note (from the ArgoCD UI's sync operation details) that in-sync resources are skipped on the next reconciliation rather than being re-applied — the scale-relevant optimization from file 2.

**Success criteria:** You've directly observed `argocd app diff` showing a real drift, watched `selfHeal: true` auto-revert it without manual intervention, and confirmed (by disabling it) that drift persists without `selfHeal` until a manual sync — three distinct, directly observed states, not assumed behavior.

---

### Part 5 — Progressive delivery with Argo Rollouts

1. Convert one tenant's `Deployment` into a `Rollout` (same `spec.template`, different `kind` and `strategy`):
   ```yaml
   # tenants/team-alpha/rollout.yaml — replaces deployment.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Rollout
   metadata:
     name: tenant-app
   spec:
     replicas: 5
     strategy:
       canary:
         steps:
           - setWeight: 20
           - pause: { duration: 30s }
           - setWeight: 60
           - pause: { duration: 30s }
           - setWeight: 100
     selector: { matchLabels: { app: tenant-app } }
     template:
       metadata: { labels: { app: tenant-app } }
       spec:
         containers:
           - name: tenant-app
             image: nginxinc/nginx-unprivileged:latest
             ports: [{ containerPort: 8080 }]
   ```
   Remove the old `deployment.yaml`, commit, push, let ArgoCD sync.

2. Trigger a rollout by changing the image tag (a real code change in a real setup — here, any tag change proves the mechanism):
   ```bash
   sed -i '' 's/latest/1.25/' tenants/team-alpha/rollout.yaml
   git add -A && git commit -m "bump tenant-app image" && git push
   argocd app sync team-alpha-app
   kubectl argo rollouts get rollout tenant-app -n tenant-team-alpha --watch
   ```
   Watch the canary steps progress in real time (20% → pause → 60% → pause → 100%).

3. Add an `AnalysisTemplate` gate (requires a metrics source — if you don't have Prometheus in this lab cluster, use a trivial `job` provider or `web` provider hitting a health endpoint instead, to prove the mechanism without deploying a full observability stack):
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AnalysisTemplate
   metadata:
     name: health-check
     namespace: tenant-team-alpha
   spec:
     metrics:
       - name: web-check
         interval: 10s
         count: 3
         successCondition: result == true
         provider:
           web:
             url: "http://tenant-app.tenant-team-alpha.svc.cluster.local/"
             jsonPath: "{$.ok}"
   ```
   Add it as a step in the `Rollout`'s `canary.steps` (`- analysis: { templates: [{ templateName: health-check }] }`) between two `setWeight` steps, push, and re-trigger.

4. Prove the abort path: force the analysis to fail (point the `web` provider's URL at a path that doesn't exist, or set an impossible `successCondition`), re-trigger a rollout, and confirm Argo Rollouts **automatically aborts and rolls back** rather than proceeding:
   ```bash
   kubectl argo rollouts get rollout tenant-app -n tenant-team-alpha --watch
   # expect: Degraded status, automatic rollback to the previous stable ReplicaSet
   ```

**Success criteria:** You've watched a canary rollout progress through weighted steps in real time, and separately watched a deliberately-failing analysis gate abort and roll back the rollout automatically — without you manually intervening to stop it.

---

## Cleanup

```bash
kubectl delete -f applicationset-git.yaml
kubectl delete appproject --all -n argocd
kubectl delete namespace argocd argo-rollouts
kubectl get ns | grep tenant- | awk '{print $1}' | xargs -r kubectl delete namespace
```

## Final checklist

- [ ] One ApplicationSet manages 5+ tenant namespaces from a single repo, using a List generator initially and a Git directory generator after Part 2.
- [ ] Adding a new `tenants/<name>/` directory and pushing creates a new Application with zero ApplicationSet edits; removing it deletes the Application and its resources.
- [ ] Every tenant Application is scoped to its own `AppProject`, and I've directly observed ArgoCD reject a cross-tenant destination change, not just assumed it would.
- [ ] `argocd-rbac-cm` has `policy.default: role:readonly` and I've confirmed an unmapped identity gets read-only access.
- [ ] I've directly observed `argocd app diff` showing real drift, watched `selfHeal: true` auto-correct it, and confirmed (by disabling it) that drift persists without `selfHeal`.
- [ ] I've watched a canary `Rollout` progress through weighted traffic steps, and separately watched a failing `AnalysisTemplate` gate trigger an automatic abort/rollback.
- [ ] I can explain, without notes, how this exact mechanism (Cluster/Matrix generators, `RollingSync`, per-cluster Rollouts) extends from "5 namespaces on 1 cluster" to "5 tenants across 50+ clusters" — same primitives, different generator inputs.
