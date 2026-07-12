# Day 63 — Lab: GitLab CI Deep Dive

**Goal:** Migrate the GitHub Actions pipeline from Day 61 to GitLab CI, and hit the real syntax differences hands-on instead of just reading about them.

**Prerequisites:**
- A free GitLab.com account (or a self-managed instance if you have access to one), with a new empty project.
- The same small app from Day 61's lab (or a fresh trivial app with one test) pushed to this new GitLab project.
- `gitlab-runner` installed locally if you want to try registering your own runner: `brew install gitlab-runner` (macOS).

---

### Lab 1 — The core hands-on activity: migrate GitHub Actions to GitLab CI

This is today's assigned hands-on activity.

1. Create `.gitlab-ci.yml` at the project root, translating the Day 61 workflow:
   ```yaml
   stages:
     - test
     - build

   test-job:
     stage: test
     image: node:20-alpine
     script:
       - npm ci
       - npm test

   lint-job:
     stage: test
     image: node:20-alpine
     script:
       - npx eslint . || true

   build-job:
     stage: build
     image: docker:24
     services:
       - docker:24-dind
     variables:
       DOCKER_TLS_CERTDIR: "/certs"
     script:
       - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
       - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
       - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
     rules:
       - if: '$CI_COMMIT_BRANCH == "main"'
   ```
2. Push and watch the pipeline run under CI/CD -> Pipelines. Confirm `test-job` and `lint-job` run in parallel (same stage), and `build-job` waits for both.
3. Open Packages & Registry -> Container Registry and confirm the pushed image is visible with the commit SHA tag.
4. Write down every syntax difference you hit versus the GitHub Actions version: `stages` vs implicit parallel jobs, `script` vs `run`, `rules` vs `on`/`if`, `$CI_COMMIT_SHA` vs `${{ github.sha }}`.

**Success criteria:** A pipeline that mirrors Day 61's test -> lint -> build-and-push flow, running natively in GitLab, with a documented list of at least 6 concrete syntax differences you personally encountered.

---

### Lab 2 — `rules:` precedence, hands-on

1. Add a deploy job with intentionally overlapping rules to observe first-match-wins behavior:
   ```yaml
   deploy-job:
     stage: build
     script:
       - echo "Deploying"
     rules:
       - if: '$CI_COMMIT_BRANCH == "main"'
         when: manual
       - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
         when: never
       - when: never
   ```
2. Push to `main` directly and confirm the job appears as a manual, clickable action (not auto-run).
3. Open a merge request instead and confirm the job doesn't appear at all (blocked by the second rule) — even though, if rules were evaluated independently rather than first-match, you might expect different behavior.
4. Reorder the two `if:` rules (branch rule second, MR rule first) and push again — observe how the outcome changes purely from reordering.

**Success criteria:** You can predict, before pushing, exactly which rule will match and what `when:` value results, for any given `CI_COMMIT_BRANCH`/`CI_PIPELINE_SOURCE` combination.

---

### Lab 3 — DRY it up with `extends`

1. Refactor Lab 1's `test-job`/`lint-job` to share a template:
   ```yaml
   .node-defaults:
     image: node:20-alpine
     before_script:
       - npm ci

   test-job:
     extends: .node-defaults
     stage: test
     script:
       - npm test

   lint-job:
     extends: .node-defaults
     stage: test
     script:
       - npx eslint . || true
   ```
2. Push and confirm both jobs still behave identically to Lab 1.
3. Bump the Node version in `.node-defaults` only (e.g., `node:22-alpine`) and confirm both jobs pick up the change without editing them individually.

**Success criteria:** One line change in the shared template propagates to both jobs — demonstrating the actual DRY payoff, not just syntax familiarity.

---

### Lab 4 — Register your own project-specific runner (optional, if you have Docker locally)

1. Get a registration token from Settings -> CI/CD -> Runners in your GitLab project.
2. Register a local Docker-executor runner:
   ```bash
   gitlab-runner register \
     --url https://gitlab.com/ \
     --registration-token <TOKEN> \
     --executor docker \
     --docker-image alpine:latest \
     --tag-list "local-lab"
   ```
3. Tag a job to require it: add `tags: [local-lab]` to `test-job`, push, and confirm the job now runs on your local runner (check `gitlab-runner run` output) instead of a shared GitLab.com runner.
4. Remove the tag (or stop the runner) and push again — observe the job either falling back to a shared runner or getting stuck `pending` if no runner matches, and explain which happened and why.

**Success criteria:** You've seen firsthand how `tags:` binds a job to a specific runner, and what happens when no runner satisfies the required tags.

---

### Cleanup

```bash
# Unregister the local runner
gitlab-runner unregister --name <runner-name>

# Delete test images from the GitLab Container Registry via the UI
# (Packages & Registry -> Container Registry -> select tag -> Delete)

# Optionally delete the whole lab project once done
```

### Stretch challenge

Add a registry cleanup policy (Settings -> Packages and registries -> Clean up image tags) that expires any tag older than 7 days except `main`-tagged images, then explain in one sentence why this matters for cost control on a busy repo.
