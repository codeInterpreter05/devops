# Day 63 — GitLab CI Deep Dive: DRY Pipelines with `extends`/Anchors & GitLab Runners

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** CI/CD | **Flag:** None

## Brief

A `.gitlab-ci.yml` with ten near-identical jobs (same `image`, same `before_script`, slightly different `script`) is a maintenance trap — one Docker image version bump means editing ten places. GitLab gives you two complementary DRY mechanisms (`extends` and native YAML anchors) plus a runner registration model that determines *where* your jobs actually execute — both are core "have you actually run GitLab CI at scale" skills.

## `extends` — GitLab-native job inheritance

```yaml
.node-defaults:
  image: node:20-alpine
  before_script:
    - npm ci
  tags: [docker]

test-job:
  extends: .node-defaults
  stage: test
  script:
    - npm test

lint-job:
  extends: .node-defaults
  stage: test
  script:
    - npx eslint .
```

- A job name prefixed with `.` (like `.node-defaults`) is a **hidden job** — GitLab parses it as a template but never runs it directly as a pipeline job.
- `extends:` performs a **deep merge**: the child job inherits every key from the parent, and any key the child also defines **overrides** the parent's value for that key (arrays like `script` are replaced wholesale, not concatenated — a common surprise).
- `extends` supports a list for multiple inheritance: `extends: [.node-defaults, .docker-cache]` — merged in order, later entries in the list win on conflicting keys.

## YAML anchors — plain YAML reuse (no GitLab-specific behavior)

```yaml
.test-template: &test-template
  image: node:20-alpine
  before_script:
    - npm ci

test-job:
  <<: *test-template
  stage: test
  script:
    - npm test
```

- `&test-template` defines an **anchor**; `<<: *test-template` **merges** that anchor's keys into the current mapping. This is native YAML functionality, not a GitLab feature — it works because `.gitlab-ci.yml` is just YAML, so anything valid YAML (including anchors/aliases) is available.
- The practical difference from `extends`: anchors are a **textual/structural merge at YAML-parse time**, entirely oblivious to GitLab's job semantics, while `extends` is GitLab-aware and does deep merging with defined precedence rules, plus works cleanly with `include:`-ed files from other projects (anchors don't survive across separate `include:`-ed YAML documents, because YAML anchors are scoped to a single document).
- **Rule of thumb**: use `extends` for anything crossing file/`include:` boundaries or needing GitLab's merge semantics; anchors are fine for quick, single-file reuse (like a small shared array of `variables`).

## `include:` — composing pipelines from multiple files

```yaml
include:
  - local: '.gitlab/ci/test.yml'
  - project: 'my-group/ci-templates'
    ref: main
    file: '/templates/deploy.yml'
  - template: 'Security/SAST.gitlab-ci.yml'   # GitLab-maintained template
```

This is how large organizations centralize shared pipeline logic (a "golden path" deploy template every microservice repo includes) — analogous in spirit to GitHub's reusable workflows, but composed at the YAML-file level rather than the job-call level, and evaluated *before* the pipeline even starts (the includes are resolved into one merged configuration).

## GitLab Runners — shared vs. specific

A **Runner** is the actual agent process that executes job `script:`s — GitLab's equivalent of a GitHub Actions runner.

| | Shared Runners | Specific (Project/Group) Runners |
|---|---|---|
| Managed by | GitLab (on GitLab.com) or your platform team (self-managed) | The project/group team itself |
| Available to | Every project in the instance (or that opts in) | Only the project(s)/group(s) it's registered against |
| Typical use | General CI (test/lint) with no special hardware/network needs | Jobs needing specific hardware (GPU), network access (private VPC), or compliance isolation |
| Registration | Pre-existing, you just use them | You register them yourself with a runner token |

```bash
# Register a project-specific runner (run on the machine that will act as the runner)
gitlab-runner register \
  --url https://gitlab.com/ \
  --registration-token <TOKEN> \
  --executor docker \
  --docker-image alpine:latest \
  --tag-list "docker,internal-vpc"
```

```yaml
# Target it explicitly in a job via tags
deploy-job:
  stage: deploy
  tags:
    - internal-vpc
  script:
    - ./deploy.sh
```

- **`tags:`** is how a job selects which runner(s) it's allowed to run on — a runner only picks up jobs whose tags are a subset of the runner's own configured tags. Forgetting to tag a job (or mismatching tags) is the single most common "why is my pipeline stuck in `pending` forever" support ticket — the job is waiting for a runner that matches its tags, and none exists.
- Executors matter: `docker` (each job in its own container — most common, most isolated), `shell` (runs directly on the runner host — fast, but no isolation between jobs, and job A's leftover files/state can affect job B), `kubernetes` (runs each job as a pod — GitLab's equivalent of GitHub's ARC-based EKS runners), `docker+machine`/`docker-autoscaler` (auto-provisions ephemeral VMs per job for strong isolation at scale).

## Points to Remember

- `extends` is GitLab-aware deep-merge inheritance that works across `include:`-ed files; YAML anchors are plain-YAML, single-document-scoped reuse — pick `extends` for anything shared across files.
- Arrays (like `script`) are **replaced**, not merged/concatenated, when a child job overrides a key its parent also defines — a common gotcha when you expect "append" behavior.
- `include:` composes a pipeline from multiple YAML sources (local files, other projects, GitLab-maintained templates) resolved before the pipeline runs — the rough equivalent of a shared "golden path" pipeline.
- A job's `tags:` must match a subset of a runner's configured tags for that runner to pick up the job — a mismatched or missing tag is the most common cause of a job stuck in `pending`.
- Executor choice (`docker`, `shell`, `kubernetes`, autoscaling) determines job isolation — `shell` executors share host state between unrelated jobs, a real security/reliability consideration for shared runners.

## Common Mistakes

- Expecting `extends`/anchors to merge/append array values (like adding to `before_script`) — they replace the array entirely, silently dropping the parent's steps unless the child re-includes them.
- Using YAML anchors across files pulled in via `include:` and being confused when the anchor "doesn't exist" — anchors don't survive across separate included YAML documents; use `extends` instead for cross-file reuse.
- Forgetting to add `tags:` to a job meant for a specific runner, so it either never runs (stuck pending, no matching runner) or accidentally runs on a shared runner it wasn't intended for.
- Using a `shell` executor for jobs from different teams/trust levels on the same runner host, letting one job's leftover credentials or files leak into another's execution environment.
- Registering a runner with an overly broad tag set "to be safe," causing it to pick up jobs it wasn't provisioned for (wrong hardware, wrong network zone) and fail unpredictably.
