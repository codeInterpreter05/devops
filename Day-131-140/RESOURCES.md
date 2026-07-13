# Day 131-140 — Resources: GitOps at Scale

## Primary (assigned)

- **ArgoCD ApplicationSet documentation** (argo-cd.readthedocs.io/en/stable/user-guide/application-set) — the assigned starting point for this block. Read the "Generators" pages for List, Cluster, Git, Matrix, and SCM Provider end to end — this repo's notes and lab compress a lot of nuance (especially around `goTemplate`, `preserveResourcesOnDeletion`, and the `RollingSync` strategy) that's worth confirming directly against the source.

## Rancher Fleet

- **Fleet documentation** (fleet.rancher.io) — the official docs for `GitRepo`, `Bundle`, `BundleDeployment`, and the Fleet Agent architecture covered in file 2. The "Concepts" section is the clearest place to see the cluster-centric model contrasted with an Application-centric one.
- **Fleet `fleet.yaml` reference** (fleet.rancher.io/ref-fleet-yaml) — full syntax for `targetCustomizations`, `dependsOn`, and Helm/Kustomize options used to drive per-cluster overrides without forking manifests.
- **Rancher documentation — Continuous Delivery** (ranchermanager.docs.rancher.com) — Fleet as integrated into the Rancher UI, useful context for how most orgs actually operate Fleet day-to-day (via Rancher, not the raw CLI).

## Argo Rollouts (progressive delivery)

- **Argo Rollouts documentation** (argo-rollouts.readthedocs.io) — start with "Concepts" then "Progressive Delivery" and "Analysis" — the `AnalysisTemplate`/`AnalysisRun` pages are the ones that matter most for the metric-gated promotion/abort mechanism in file 2.
- **Argo Rollouts — Traffic Management** (argo-rollouts.readthedocs.io/en/stable/features/traffic-management) — the integration pages for Istio/NGINX/SMI/ALB/AppMesh traffic routing, needed once canary weighting requires actual traffic splitting rather than pod-count approximation.
- **Argo Rollouts Kubectl Plugin reference** (argo-rollouts.readthedocs.io/en/stable/generated/kubectl-argo-rollouts) — full command reference for `get`, `promote`, `abort`, `retry`, `undo` used throughout the lab and cheatsheet.

## ArgoCD multi-tenancy and RBAC

- **ArgoCD documentation — Projects** (argo-cd.readthedocs.io/en/stable/user-guide/projects) — the authoritative reference for `AppProject` fields (`sourceRepos`, `destinations`, `clusterResourceWhitelist`/`Blacklist`, `orphanedResources`) used in file 3.
- **ArgoCD documentation — RBAC** (argo-cd.readthedocs.io/en/stable/operator-manual/rbac) — the full Casbin policy syntax for `argocd-rbac-cm`, including SSO scope configuration (`scopes: '[groups]'`) and built-in roles (`role:readonly`, `role:admin`).
- **ArgoCD documentation — Cluster Bootstrapping / App of Apps and ApplicationSet Go Template** (argo-cd.readthedocs.io) — the `goTemplate: true` semantics referenced in file 1's templating section, straight from source since this is an area the tool's behavior has shifted across versions.

## GitOps security best practices

- **OpenGitOps principles** (opengitops.dev) — the CNCF GitOps Working Group's vendor-neutral definition of GitOps (declarative, versioned, pulled automatically, continuously reconciled) — the baseline vocabulary this entire block assumes.
- **GitHub documentation — About protected branches / About code owners** (docs.github.com) — the exact mechanics (required reviews, required status checks, CODEOWNERS enforcement) behind file 3's "repo access control is your deployment authorization model" argument; read this even if your org uses GitLab/Bitbucket, since the equivalent settings (merge request approval rules, CODEOWNERS-equivalent) map directly.
- **CNCF GitOps Working Group — GitOps security whitepaper** (search "CNCF GitOps security" or check the OpenGitOps GitHub org) — a vendor-neutral treatment of pull-vs-push credential exposure, repo access control, and supply-chain concerns specific to GitOps operators.
- **Sigstore / cosign documentation** (docs.sigstore.dev) — relevant follow-on once GitOps security discussions extend into "is the image being deployed actually the one that passed CI" — pairs with the admission-control policy-as-code material from earlier phases.

## Reference / lookup

- **ArgoCD `argocd` CLI reference** (argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd) — full flag reference for every `argocd app` / `argocd appset` / `argocd proj` subcommand used in this block's cheatsheet and lab.
- **ArgoCD Notifications documentation** (argocd-notifications.readthedocs.io or the ArgoCD docs' "Notifications" section) — for alerting on sustained `OutOfSync`/`Degraded` status fleet-wide, referenced in file 2's drift-detection-at-scale discussion.
