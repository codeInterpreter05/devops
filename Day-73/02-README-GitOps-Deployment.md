# Day 73 — Full CI/CD Pipeline Project: GitOps Deployment with ArgoCD

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** CI/CD | **Flag:** 📌 Capstone

## Brief

The pipeline built in file 1 ends at `push` — a signed, scanned image sits in a registry. Getting it running in a cluster is a separate concern, and *how* you do that separation is one of the more nuanced design decisions in modern CI/CD: **push-based deployment** (CI directly runs `kubectl apply`/`helm upgrade`) vs. **GitOps / pull-based deployment** (a controller like ArgoCD, running inside the cluster, continuously reconciles cluster state against a Git repo). This capstone uses ArgoCD with two different sync policies for staging vs. production — a pattern you will be asked to defend in almost every senior DevOps interview.

## Push vs. pull deployment models

| | Push (CI runs `kubectl apply`) | Pull (ArgoCD/Flux reconciles) |
|---|---|---|
| Cluster credentials | Must live in CI (long-lived kubeconfig or OIDC federation) | Never leave the cluster — ArgoCD runs in-cluster |
| Source of truth | Whatever CI last applied | The Git repo — always |
| Drift detection | None — if someone `kubectl edit`s prod, CI doesn't notice | Continuous — ArgoCD flags/reverts drift automatically |
| Network direction | CI reaches into the cluster (inbound to cluster network) | Controller reaches out to Git (outbound only) |
| Rollback | Re-run an old pipeline / re-apply old manifests | `git revert` — the controller does the rest |

The pull model's security win is significant: **no CI system anywhere holds a credential capable of writing to your production cluster.** ArgoCD's in-cluster service account is the only thing with that power, and it only ever pulls (reads) from Git plus writes to its own cluster. This shrinks your attack surface from "every CI runner across every repo" down to "one controller."

## ArgoCD Application manifests: staging vs. production

Two `Application` CRs, differing only in `syncPolicy`:

```yaml
# argocd/apps/staging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-gitops.git
    targetRevision: main
    path: overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      prune: true        # delete resources removed from Git
      selfHeal: true      # revert manual kubectl edits automatically
    syncOptions:
      - CreateNamespace=true
```

```yaml
# argocd/apps/production.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-gitops.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy: {}   # NO automated block = manual sync required
```

`staging` has `syncPolicy.automated` — the moment the CI pipeline (file 1) updates the image tag in `overlays/staging/kustomization.yaml` and pushes to the GitOps repo, ArgoCD notices within its poll interval (default 3 minutes, or instantly via a webhook) and reconciles automatically. `production` has an **empty** `syncPolicy`, meaning ArgoCD will show the Application as `OutOfSync` but will *not* act — a human must click "Sync" in the UI or run `argocd app sync myapp-production` deliberately, ideally gated behind a change-approval step.

## How the CI pipeline actually triggers ArgoCD

CI never talks to the cluster or to ArgoCD's API directly for the automated path — it writes to the GitOps repo, and ArgoCD's control loop does the rest:

```yaml
  update-gitops:
    needs: sign
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: myorg/myapp-gitops
          token: ${{ secrets.GITOPS_REPO_TOKEN }}
      - name: Bump staging image tag
        run: |
          cd overlays/staging
          kustomize edit set image myapp=${{ env.IMAGE_NAME }}@${{ needs.build.outputs.digest }}
      - run: |
          git config user.name "ci-bot"
          git config user.email "ci-bot@myorg.com"
          git commit -am "chore: bump staging image to ${{ github.sha }}"
          git push
```

This is the essence of GitOps: **the CI pipeline's job ends at "propose a change to desired state in Git."** ArgoCD is the only thing that ever touches the cluster. For production, a separate, human-triggered workflow (or manual `kustomize edit` + PR + approval + `argocd app sync`) performs the equivalent bump against `overlays/production` — often gated by requiring an approving review on that GitOps-repo PR itself, giving you a full audit trail of who promoted what, when.

## Promotion strategy: environments as overlays, not branches

Use **Kustomize overlays** (or Helm value files) per environment rather than long-lived environment branches (`staging-branch`, `prod-branch`). Environment branches drift and require merge/cherry-pick gymnastics to promote a change; overlays share a common `base/` and differ only in what's explicitly overridden (replica count, resource limits, image tag), so "promoting to prod" is a one-line diff in a small overlay file, not a branch merge that could pull in unrelated changes.

```
overlays/
├── base/
│   ├── deployment.yaml
│   └── kustomization.yaml
├── staging/
│   └── kustomization.yaml   # references base, overrides image tag + replicas
└── production/
    └── kustomization.yaml   # references base, overrides image tag + replicas + resource limits
```

## Points to Remember

- GitOps (pull-based) means no CI system holds cluster-write credentials — ArgoCD's in-cluster identity is the only thing that ever writes to the cluster, shrinking the credential attack surface dramatically versus push-based `kubectl apply` from CI.
- `syncPolicy.automated` (with `selfHeal: true`) = auto-reconcile on every Git change, appropriate for staging. An empty/absent `syncPolicy` = manual sync only, appropriate for production.
- `selfHeal: true` also means ArgoCD will silently revert any manual `kubectl edit`/`patch` against a resource it manages — this is a feature (prevents drift) but surprises people who "just want to quickly patch prod."
- CI's job in a GitOps pipeline ends at committing an image-tag bump to the GitOps repo — it never calls the cluster or ArgoCD API for the automated staging path.
- Use environment **overlays** (Kustomize/Helm), not long-lived environment branches, so promotion is a small, auditable diff rather than a branch merge.

## Common Mistakes

- Giving CI a long-lived production kubeconfig "just in case," defeating the entire security rationale for adopting GitOps in the first place.
- Setting `syncPolicy.automated` on the production Application by copy-pasting the staging manifest — this removes the human approval gate that's the whole point of separating the two environments.
- Forgetting `prune: true` and being confused when deleting a resource from the Git manifests doesn't actually remove it from the cluster.
- Manually `kubectl edit`-ing a resource in a `selfHeal: true` namespace to "quickly fix" an incident, not realizing ArgoCD will revert it on the next reconciliation loop — the fix must go through Git.
- Using separate long-lived branches per environment instead of overlays, leading to painful/conflict-prone merges when "promoting" a release.
