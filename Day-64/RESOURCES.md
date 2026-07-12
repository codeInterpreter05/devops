# Day 64 — Resources: ArgoCD Deep Dive

## Primary (assigned)

- **ArgoCD documentation** (argo-cd.readthedocs.io) — free, the assigned starting point. Start with "Core Concepts," then "Application," "Sync Policies," and "Sync Waves and Hooks" sections directly relevant to today.

## Deepen your understanding

- **"App of Apps" pattern guide** (argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) — the canonical pattern for managing ArgoCD's own `Application` resources via GitOps, referenced in file 1's architecture discussion.
- **ArgoCD `ApplicationSet` documentation** (argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/) — the full generator reference (`list`, `cluster`, `git`, matrix, and more advanced ones like SCM provider and Pull Request generators).
- **ArgoCD Notifications documentation** (argocd-notifications.readthedocs.io or the bundled docs under argo-cd.readthedocs.io) — full reference on triggers, templates, and supported notification services.
- **CNCF "GitOps Principles"** (opengitops.dev) — the vendor-neutral definition of what actually qualifies as GitOps (declarative, versioned, pulled automatically, continuously reconciled) — useful for grounding the interview answer beyond "ArgoCD-specific" trivia.

## Reference / lookup

- **ArgoCD resource health documentation** (argo-cd.readthedocs.io/en/stable/operator-manual/health/) — the authoritative list of built-in health checks per resource type, and the Lua custom health check syntax.
- **ArgoCD CLI reference** (argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd_app/) — every `argocd app *` subcommand and flag.

## Practice

- **ArgoCD's own "Getting Started" tutorial** (argo-cd.readthedocs.io/en/stable/getting_started/) — a guided, hands-on first deployment, good to run once end-to-end before diving into today's lab for extra repetition.
- Break something on purpose: push a manifest with an invalid image tag, watch health go `Degraded`, then practice `argocd app rollback` for real — doing this once yourself builds far more confidence than reading about the rollback command.
