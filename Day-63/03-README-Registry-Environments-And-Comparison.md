# Day 63 — GitLab CI Deep Dive: Container Registry, Environments & Deploy Boards, GitLab vs GitHub Actions

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** CI/CD | **Flag:** None

## Brief

The final piece of GitLab CI fluency is knowing what's built into the platform beyond the pipeline YAML itself — a container registry with zero extra setup, and Environments/deploy boards that give you a visual, auditable deployment history. Then, because today's interview question asks for it directly, a clear-eyed comparison of GitLab CI against GitHub Actions — the two systems you're most likely to be asked to compare in an interview or actually migrate between on the job.

## GitLab Container Registry

Every GitLab project gets an integrated container registry for free, at `registry.gitlab.com/<group>/<project>` (or your self-managed instance's registry host) — no separate ECR/Docker Hub account needed.

```yaml
build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind   # Docker-in-Docker, needed to run docker build inside a Docker executor job
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
```

- `$CI_REGISTRY`, `$CI_REGISTRY_IMAGE`, `$CI_REGISTRY_USER`, `$CI_REGISTRY_PASSWORD` are **predefined CI/CD variables** GitLab injects automatically — no manual secret setup required for basic registry push access from within the same project's pipeline.
- **Docker-in-Docker (`dind`)** is required because the `docker` CLI needs an actual Docker daemon to talk to, and the job's own container doesn't have one by default — the `docker:24-dind` service provides that daemon as a sidecar. This has real security implications: dind typically needs privileged mode, which is a meaningfully larger attack surface than a normal job — many teams instead use `kaniko` or `buildah` to build images without needing a Docker daemon or privileged containers at all.
- Registry cleanup policies (image expiration rules configurable per-project) matter in practice — without them, every commit's tagged image accumulates forever and registry storage costs grow unbounded.

## Environments & deploy boards

```yaml
deploy-staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com

deploy-production:
  stage: deploy
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

- `environment:` registers a deployment against a named target, giving you GitLab's **Environments** page — a timeline of every deploy to that target, who triggered it, and a one-click **rollback** to redeploy a prior successful job's artifacts.
- Combined with `when: manual`, this is GitLab's version of GitHub's required-reviewer gate — though GitLab's native `rules`-based manual gate doesn't have GitHub's separate "required reviewer" role-based approval concept built in at the same granularity without GitLab Premium/Ultimate features (protected environments with designated approvers).
- **Deploy boards** (available with Kubernetes integration configured) render each Environment's live pod status directly in the GitLab UI — rollout progress, pod health — without needing to `kubectl get pods` manually, useful for a quick visual sanity check during a deploy.
- **Protected environments** (a paid-tier feature) let you restrict who can deploy to `production` at the GitLab permissions level, similar in spirit to GitHub Environments' required reviewers, but enforced through GitLab's role system (Maintainer/Owner) rather than a separate reviewer list.

## GitLab CI vs. GitHub Actions — the real comparison

| | GitLab CI | GitHub Actions |
|---|---|---|
| Config file | Single `.gitlab-ci.yml` (composable via `include:`) | Multiple files under `.github/workflows/` |
| Execution unit | `stages` (parallel within a stage, sequential across stages by default) | `jobs` (parallel by default, ordered via `needs`) |
| Conditional logic | `rules:` (first-match-wins, evaluated top-down) | `if:` expressions per-job/step, `on.<event>` filters |
| Reuse | `extends`, YAML anchors, `include:` | Composite actions, reusable workflows (`workflow_call`) |
| Marketplace/ecosystem | Fewer third-party "actions" — more roll-your-own `script:` and shared templates | Enormous Marketplace of prebuilt actions for almost anything |
| Container registry | Built in, zero setup, per-project | Not built in — GitHub Container Registry (ghcr.io) exists but is a separate product, or use ECR/Docker Hub |
| Runners | `gitlab-runner` — shared, group, or project-specific, self-hostable with several executor types | GitHub-hosted or self-hosted (ARC on K8s, or plain VM-based) |
| Self-managed/on-prem | First-class, common in regulated/on-prem enterprises (GitLab self-managed) | Possible (GitHub Enterprise Server) but far less common in practice |
| Deploy visualization | Native Environments page + deploy boards (K8s) | Environments page (simpler, no built-in deploy board) |
| Variables model | Plain shell env vars (`$CI_COMMIT_SHA`), CI/CD Variables UI | Structured `${{ }}` expression contexts (`github.sha`), Secrets/Variables UI |

### Migrating a pipeline: the concrete syntax translation

| GitHub Actions | GitLab CI equivalent |
|---|---|
| `on: push: branches: [main]` | `rules: - if: '$CI_COMMIT_BRANCH == "main"'` |
| `jobs.<name>.needs: [a, b]` | `needs: [a, b]` (same concept, similar syntax) |
| `jobs.<name>.strategy.matrix` | `parallel: matrix:` |
| `secrets.MY_SECRET` | `$MY_SECRET` (as a CI/CD Variable marked "protected"/"masked") |
| Composite action | `extends` + shared `.gitlab/ci/*.yml` included template |
| Reusable workflow (`workflow_call`) | `include:` of a shared template from another project |
| `actions/cache` | `cache:` key with `paths:` |
| GitHub Environments + required reviewer | `environment:` + `when: manual` (+ protected environments on paid tiers) |

## Points to Remember

- GitLab's container registry and predefined `$CI_REGISTRY*` variables mean zero extra setup for basic image push/pull — a real convenience GitHub Actions doesn't match out of the box.
- Docker-in-Docker (`dind`) for image builds usually needs privileged mode — a real security tradeoff; `kaniko`/`buildah` avoid it entirely and are the more security-conscious default for build-only jobs.
- GitLab's `environment:` + `when: manual` is the rough equivalent of GitHub's Environment + required reviewer, but the finer-grained "who specifically can approve" control is a paid-tier (protected environments) feature.
- The two systems' core concepts map fairly directly (stages~jobs vs jobs~needs, rules vs if, extends vs composite actions) — a migration is mostly a syntax translation exercise once you know both models, not a conceptual rewrite.
- GitLab's self-managed option is a major reason large regulated enterprises choose it over GitHub Actions, which leans much more heavily on the hosted, cloud-native model.

## Common Mistakes

- Running `docker build` in a GitLab job without setting up `dind` (or without realizing dind needs privileged mode) and getting a confusing "cannot connect to the Docker daemon" error.
- Not setting registry cleanup/expiration policies, letting the built-in container registry silently accumulate every commit's image forever and blow up storage costs.
- Assuming GitLab's `when: manual` gate is identical to GitHub's required-reviewer Environment protection — without protected environments (a paid feature), any Maintainer can click the manual job, not just a designated approver.
- Treating a GitLab-to-GitHub (or reverse) migration as a full rewrite instead of a fairly mechanical translation — most teams overestimate the effort because they don't first map the syntax table above.
- Choosing `dind`/privileged builds by default instead of evaluating `kaniko`/`buildah`, especially on shared runners where a privileged container is a materially larger blast radius if a build job is ever compromised.
