# Day 73 — Lab: Full CI/CD Pipeline Project

**Goal:** Build the complete production-grade pipeline for your Phase 1 infrastructure project — test → SAST → build → scan → sign → push, deployed via ArgoCD with staging auto-sync and production manual-sync, with Slack failure alerts and PR preview environments.

**Prerequisites:**
- A GitHub repo containing your Phase 1 infrastructure project (or any app + Dockerfile if you don't have one handy).
- A Kubernetes cluster (kind/minikube is fine) with ArgoCD installed (`kubectl create namespace argocd && kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`).
- A Slack workspace where you can create an incoming webhook or bot token.
- `cosign`, `kustomize`, `helm`, and the GitHub CLI (`gh`) installed locally.

---

### Lab 1 — Build the test → SAST → build stages

1. In your repo, create `.github/workflows/ci-cd.yml` with `test`, `sast`, and `build` jobs as shown in `01-README-Pipeline-Architecture.md`, adapted to your project's actual language/test runner.
2. Push a commit that intentionally fails a test. Confirm in the Actions tab that `sast` and `build` show as **skipped**, not failed — this proves the `needs:` gate is working.
3. Fix the test, push again, and confirm all three jobs go green in sequence.

**Success criteria:** You can point at the Actions run graph and explain, for a failing commit, exactly which jobs ran and which were skipped, and why.

---

### Lab 2 — Add scan and sign, gated on the image digest

1. Add the `scan` job using Trivy against `needs.build.outputs.digest` (not a tag).
2. Deliberately use a base image with a known CVE (e.g., an old `node:14` or `python:3.6`) to confirm the scan job fails and blocks `sign`.
3. Switch to a current, patched base image; confirm `scan` passes and `sign` runs, producing a cosign signature.
4. Verify the signature independently:
   ```bash
   cosign verify --certificate-identity-regexp ".*" \
     --certificate-oidc-issuer https://token.actions.githubusercontent.com \
     ghcr.io/<you>/<repo>@<digest>
   ```

**Success criteria:** A vulnerable image cannot reach the `sign` stage; a clean image gets a verifiable cosign signature tied to your GitHub Actions OIDC identity.

---

### Lab 3 — ArgoCD: staging auto-sync, production manual-sync

1. Create a separate `myapp-gitops` repo with `overlays/base`, `overlays/staging`, `overlays/production` Kustomize directories.
2. Apply both `Application` manifests from `02-README-GitOps-Deployment.md` (`argocd app list` should show both).
3. Add an `update-gitops` job to your pipeline that bumps `overlays/staging/kustomization.yaml`'s image tag and pushes. Confirm ArgoCD auto-syncs staging within ~3 minutes (or trigger immediately via `argocd app sync myapp-staging` if you don't want to wait).
4. Manually bump `overlays/production/kustomization.yaml` in a PR, merge it, and confirm the `myapp-production` Application shows `OutOfSync` but does **not** auto-deploy. Run `argocd app sync myapp-production` yourself to promote it.
5. Test `selfHeal`: `kubectl edit deployment` directly in the `staging` namespace to change the replica count, then watch ArgoCD revert it within its next reconciliation loop.

**Success criteria:** You can demonstrate, live, the difference in behavior between the two `syncPolicy` configurations, and explain why production requires a manual step.

---

### Lab 4 — Slack notifications and a broken build

1. Create a Slack app / bot token with `chat:write` scope, store it as `SLACK_BOT_TOKEN` in repo secrets.
2. Add the `notify` job from `03-README-Notifications-Preview-Envs.md`.
3. Push a commit that fails the `test` job. Confirm a Slack message arrives with a working link to the failed run.
4. Push a fix. Confirm no Slack message fires on success (or optionally add a "recovered" message — stretch).

**Success criteria:** You receive exactly one Slack message for the failing commit, containing a clickable link to the correct run, and zero noise for the passing commit.

---

### Lab 5 — PR preview environments

1. Add the `preview` and `preview-cleanup` jobs, targeting your kind/minikube cluster (use `kubectl port-forward` or a local ingress controller like `ingress-nginx` with `nip.io` hostnames if you don't have real DNS).
2. Open a PR, confirm the bot comments a working preview URL, and confirm you can actually load the app at that URL.
3. Close the PR without merging. Confirm the preview namespace is deleted.
4. Add the scheduled TTL-cleanup workflow and manually verify it would catch a namespace older than your threshold (e.g., by temporarily lowering the threshold to 1 minute for testing).

**Success criteria:** Every open PR gets its own reachable environment and its own comment; closing (merged or not) always tears it down; a time-based backstop exists independent of the explicit cleanup step.

---

### Cleanup

```bash
kubectl delete namespace staging production --ignore-not-found
kubectl get namespaces | grep '^pr-' | awk '{print $1}' | xargs -r kubectl delete namespace
argocd app delete myapp-staging myapp-production --yes
```

### Stretch challenge

Add a "recovered" Slack message: track the previous run's conclusion (via the GitHub API, `gh run list --status failure --limit 1` or a small state file committed to a `ci-state` branch) and send a 🟢 "pipeline recovered" message only on the first success after one or more failures — not on every green run.
