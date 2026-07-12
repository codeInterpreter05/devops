# Day 61 — Lab: GitHub Actions Fundamentals

**Goal:** Build a real CI pipeline — test → lint → build Docker → push to ECR on every PR — and understand exactly why each part of the YAML behaves the way it does.

**Prerequisites:**
- A GitHub repo you control (can be a throwaway test repo) with a small app (any language — a trivial Node/Python "hello world" with one test is enough).
- An AWS account with an ECR repository created (`aws ecr create-repository --repository-name gha-lab-demo`), or substitute Docker Hub if you don't want to touch AWS yet (Day 62 covers OIDC auth to AWS properly — for today, IAM access keys stored as repo secrets is fine, just don't leave them long-lived).
- `act` installed locally for fast iteration: `brew install act` (macOS) or see `github.com/nektos/act`.
- Docker installed and running (required by `act` and for the Docker build step).

---

### Lab 1 — The skeleton: `on` / `jobs` / `steps`

1. Create `.github/workflows/ci.yml` with:
   ```yaml
   name: CI
   on:
     pull_request:
       branches: [main]
     push:
       branches: [main]
   jobs:
     test:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: '20'
         - run: npm ci
         - run: npm test
   ```
2. Push to a branch, open a PR, and watch the "Checks" tab populate in real time.
3. Break the test on purpose (make an assertion fail), push again, and confirm the PR shows a red X and blocks merge (if you've enabled required checks in branch protection — set that up if not already).

**Success criteria:** You can explain, from the Actions run log, exactly which event triggered the run and which commit SHA it ran against.

---

### Lab 2 — Add a lint job that runs in parallel, not sequentially

1. Add a second job to the same workflow:
   ```yaml
     lint:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: '20'
         - run: npx eslint . || true   # `|| true` for lab purposes if you don't have eslint configured
   ```
2. Push and confirm in the Actions UI that `test` and `lint` run **at the same time**, not one after another.
3. Now make `lint` a prerequisite for a new `build` job using `needs: [test, lint]`, and confirm `build` waits for both to finish.

**Success criteria:** You can point at the workflow visualization graph and correctly predict execution order before running it.

---

### Lab 3 — The core hands-on activity: full pipeline to ECR

This is today's assigned hands-on activity — build it end-to-end.

1. Store AWS credentials as repo secrets: Settings → Secrets and variables → Actions → New repository secret: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and a repo **variable** `AWS_REGION` (non-secret, so use Variables not Secrets) and `ECR_REPOSITORY`.
2. Extend the workflow with a `build` job:
   ```yaml
     build:
       needs: [test, lint]
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - name: Configure AWS credentials
           uses: aws-actions/configure-aws-credentials@v4
           with:
             aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
             aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
             aws-region: ${{ vars.AWS_REGION }}
         - name: Login to ECR
           id: ecr-login
           uses: aws-actions/amazon-ecr-login@v2
         - name: Build and push
           env:
             REGISTRY: ${{ steps.ecr-login.outputs.registry }}
             REPOSITORY: ${{ vars.ECR_REPOSITORY }}
             IMAGE_TAG: ${{ github.sha }}
           run: |
             docker build -t "$REGISTRY/$REPOSITORY:$IMAGE_TAG" .
             docker push "$REGISTRY/$REPOSITORY:$IMAGE_TAG"
   ```
3. Push a PR, verify all three jobs run in the right order, and confirm the image lands in ECR: `aws ecr list-images --repository-name gha-lab-demo`.
4. Gate the `build` job so it only runs on `push` to `main` (not on every PR) using `if: github.event_name == 'push'` — confirm PRs now only run `test`/`lint`.

**Success criteria:** A merged PR to `main` produces a new image tag in ECR named after the commit SHA, and the pipeline correctly skips the Docker build for PRs that haven't merged yet.

---

### Lab 4 — Run it locally with `act` before pushing

1. Run the workflow locally without pushing to GitHub at all:
   ```bash
   act pull_request -j test
   ```
2. Pass in the secrets `act` needs from a local `.secrets` file (never commit this file — add it to `.gitignore`):
   ```bash
   echo "AWS_ACCESS_KEY_ID=test" >> .secrets
   echo "AWS_SECRET_ACCESS_KEY=test" >> .secrets
   act push -j build --secret-file .secrets
   ```
3. Compare how long the `act` run takes locally versus waiting for a real GitHub-hosted runner — note the tradeoff (fast local iteration vs. not being 100% identical to GitHub's actual runner image).

**Success criteria:** You can iterate on workflow YAML syntax errors locally in seconds instead of waiting on a push-and-wait loop.

---

### Cleanup

```bash
# Delete the test ECR images so you're not paying for storage
aws ecr batch-delete-image --repository-name gha-lab-demo --image-ids imageTag=<sha>

# Remove the local .secrets file used by act
rm -f .secrets

# Revoke/rotate the temporary IAM access keys used in this lab
aws iam delete-access-key --access-key-id <AKIA...> --user-name <lab-user>
```

### Stretch challenge

Add `paths-ignore: ['**.md']` to the `push`/`pull_request` triggers so documentation-only commits skip the entire pipeline, then add a matrix to the `test` job that runs on Node 18, 20, and 22 with `fail-fast: false`, and confirm the PR check list shows three separate "test" entries.
