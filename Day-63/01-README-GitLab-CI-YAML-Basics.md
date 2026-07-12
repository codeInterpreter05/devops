# Day 63 — GitLab CI Deep Dive: YAML Basics — Stages, Jobs, Rules

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** CI/CD | **Flag:** None

## Brief

GitLab CI is the other CI system you're highly likely to meet professionally — a huge share of enterprises, especially ones with strict on-prem/self-hosted compliance requirements, run GitLab (often self-managed) rather than GitHub. The core concepts overlap heavily with GitHub Actions (a pipeline is stages of jobs), but the syntax, execution model, and defaults differ in ways that matter — `rules:` versus `if:`/`workflow_dispatch`, `stages` versus GitHub's implicit `needs`-based DAG, and GitLab's own runner registration model. Knowing both systems fluently, and being able to translate between them, is exactly what today's interview question tests.

This day is split into three files:

1. **This file** — `.gitlab-ci.yml` structure: `stages`, `jobs`, and the `rules` system that controls when jobs run.
2. **[02-README-DRY-Pipelines-And-Runners.md](02-README-DRY-Pipelines-And-Runners.md)** — `extends`/YAML anchors for DRY pipelines, and shared vs. specific GitLab Runners.
3. **[03-README-Registry-Environments-And-Comparison.md](03-README-Registry-Environments-And-Comparison.md)** — GitLab Container Registry, Environments & deploy boards, and a direct GitLab-vs-GitHub-Actions comparison.

## The anatomy of `.gitlab-ci.yml`

Every GitLab project can have a `.gitlab-ci.yml` at its root (or referenced via `include:` — see file 2). The core structure is **stages → jobs**, which is subtly different from GitHub's job graph:

```yaml
stages:
  - test
  - build
  - deploy

test-job:
  stage: test
  script:
    - npm ci
    - npm test

lint-job:
  stage: test
  script:
    - npx eslint .

build-job:
  stage: build
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .

deploy-job:
  stage: deploy
  script:
    - ./deploy.sh
  only:
    - main
```

- **`stages`** defines an ordered list of phases. **All jobs in the same stage run in parallel**; the next stage only starts once every job in the current stage has finished (and, by default, only if none failed). This is GitLab's default execution model — it's stage-ordered by default, unlike GitHub Actions where jobs are unordered by default and you opt into ordering via `needs`.
- Each **job** (`test-job`, `lint-job`, etc.) is a top-level YAML key with a `script:` — the actual shell commands, GitLab's equivalent of GitHub's `run:` steps, but a job is a flatter unit (no nested per-job "steps" list the way GitHub structures it — everything is one `script` array of shell lines, plus optional `before_script`/`after_script`).
- **`needs:`** exists in GitLab too, and lets a job skip waiting for its entire stage to complete, instead depending on specific other jobs directly — creating a DAG-like pipeline that starts jobs as soon as their specific dependencies finish, rather than waiting for the whole stage. This closes most of the gap with GitHub's needs-based model.

```yaml
build-job:
  stage: build
  needs: [test-job]   # starts as soon as test-job finishes, doesn't wait for lint-job too
  script:
    - docker build -t myapp .
```

## `rules:` — GitLab's conditional execution system

`rules:` is GitLab's modern (and now strongly preferred over the legacy `only`/`except`) mechanism for controlling whether/when a job runs, based on pipeline context:

```yaml
deploy-prod:
  stage: deploy
  script:
    - ./deploy.sh production
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never
    - when: on_success   # fallback default for anything else
```

Mechanics:
- Rules are evaluated **top to bottom**, and the **first matching rule wins** — this is a critical difference from a list of independent conditions; order matters enormously.
- `when: manual` makes the job appear in the pipeline UI but require a manual click to actually run — GitLab's equivalent of a deploy gate (paired with `allow_failure: false` if a skipped manual job shouldn't block the pipeline from being considered "green").
- `when: never` explicitly excludes the job for that condition — necessary because without an explicit terminal rule, a job with *no* matching rule simply doesn't run at all (there's no implicit "else this job runs anyway").
- Built-in predefined variables like `$CI_COMMIT_BRANCH`, `$CI_PIPELINE_SOURCE`, `$CI_MERGE_REQUEST_ID` are how you branch pipeline behavior — analogous to GitHub's `github.ref`/`github.event_name` contexts, but exposed as plain shell environment variables rather than a structured `${{ }}` expression context.

### `rules:changes` — path-based conditional execution

```yaml
build-frontend:
  stage: build
  script: [npm run build]
  rules:
    - changes:
        - frontend/**/*
```

This is GitLab's equivalent of GitHub's `paths:` trigger filter, but scoped to an individual **job** rather than the whole pipeline — meaning a single pipeline can selectively run only the jobs relevant to what actually changed in that commit/MR (e.g., skip `build-frontend` entirely if only backend files changed), which is more granular than GitHub Actions' workflow-level path filtering.

## Points to Remember

- Stages are ordered and block sequentially by default (all jobs in a stage finish before the next stage starts); `needs:` lets specific jobs bypass strict stage-ordering and start as soon as their actual dependencies are done.
- `rules:` evaluates top-to-bottom, first match wins — unlike independent boolean conditions, rule order is part of the logic, not just style.
- A job with no matching rule simply does not run — there's no implicit fallback "runs by default" behavior unless you add a catch-all `- when: on_success` (or similar) as the last rule.
- `rules:changes` gives per-job, path-based conditional execution — more granular than GitHub Actions' workflow-level `paths:` filter.
- `only`/`except` still work but are legacy — `rules:` is the current, more expressive, and actively maintained mechanism; new pipelines should use `rules:` exclusively.

## Common Mistakes

- Mixing `only`/`except` and `rules:` on the same job — GitLab explicitly disallows or produces confusing precedence when both are present; pick one mechanism per job (and standardize on `rules:` project-wide).
- Assuming `rules:` conditions are evaluated independently (like a switch statement with fallthrough) rather than short-circuiting on first match — reordering rules can silently change which one applies.
- Forgetting `when: manual` jobs still block the pipeline (by default `allow_failure: false` if left unspecified for a manual job depends on GitLab version/config) — check whether a skipped manual approval step should or shouldn't count against overall pipeline status.
- Writing a stage-heavy pipeline (`test` -> `build` -> `deploy`, each waiting for the full previous stage) when a `needs:`-based DAG would let independent jobs start earlier — needlessly slow pipelines on large projects.
- Forgetting that `rules:changes` on merge request pipelines needs `CI_MERGE_REQUEST_DIFF_BASE_SHA` context to compare correctly — on some pipeline sources (e.g., a fresh branch pipeline with no prior commit to diff against) `changes:` can behave unexpectedly and match everything.
