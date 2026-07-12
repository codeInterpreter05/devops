# Day 62 — Lab: Advanced GitHub Actions

**Goal:** Replace static AWS keys in CI with OIDC, and set up a GitHub Environment with a required reviewer gating production deploys.

**Prerequisites:**
- Completed Day 61's lab (a working CI pipeline that builds/pushes a Docker image).
- AWS CLI configured locally with an account you can create IAM roles/OIDC providers in.
- Admin access to the GitHub repo (to configure Environments and required reviewers).

---

### Lab 1 — The core hands-on activity: static keys to OIDC

This is today's assigned hands-on activity.

1. Register GitHub's OIDC provider in your AWS account (skip if already present — check first):
   ```bash
   aws iam list-open-id-connect-providers
   aws iam create-open-id-connect-provider \
     --url https://token.actions.githubusercontent.com \
     --client-id-list sts.amazonaws.com \
     --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
   ```
2. Create a trust policy file `trust-policy.json` scoped to your repo and `main` branch:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": {"Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"},
       "Action": "sts:AssumeRoleWithWebIdentity",
       "Condition": {
         "StringEquals": {"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"},
         "StringLike": {"token.actions.githubusercontent.com:sub": "repo:<YOUR_ORG>/<YOUR_REPO>:ref:refs/heads/main"}
       }
     }]
   }
   ```
3. Create the role and attach a minimally-scoped permissions policy (e.g., ECR push only):
   ```bash
   aws iam create-role --role-name github-actions-deploy --assume-role-policy-document file://trust-policy.json
   aws iam attach-role-policy --role-name github-actions-deploy \
     --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
   ```
4. Update the workflow to drop the static-key step entirely:
   ```yaml
   permissions:
     id-token: write
     contents: read
   steps:
     - uses: aws-actions/configure-aws-credentials@v4
       with:
         role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-actions-deploy
         aws-region: us-east-1
   ```
5. Delete the old `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` repo secrets entirely, push, and confirm the pipeline still authenticates and pushes to ECR successfully.
6. Deliberately break the trust policy's `sub` value (point it at a different branch) and confirm the workflow now fails with an `AssumeRoleWithWebIdentity`/`AccessDenied` error — this proves the scoping is actually enforced, not decorative.

**Success criteria:** No AWS static credentials exist anywhere in the repo's secrets, and the pipeline still authenticates successfully via a short-lived, scoped session.

---

### Lab 2 — Set up a GitHub Environment with a required reviewer

1. In repo Settings → Environments, create an environment named `production`.
2. Add yourself (or a teammate) as a required reviewer.
3. Add a deployment branch rule restricting `production` to only the `main` branch.
4. Update the deploy job to reference the environment:
   ```yaml
   jobs:
     deploy:
       environment:
         name: production
         url: https://example.com
       runs-on: ubuntu-latest
       steps:
         - run: echo "Deploying to production"
   ```
5. Push to `main` and confirm the job pauses with "Waiting for review" until you approve it in the Actions UI.
6. Try pushing to a non-`main` branch and confirm the job is blocked entirely by the branch restriction, before it even reaches the reviewer gate.

**Success criteria:** You can demonstrate a deploy job that is provably blocked by both a branch restriction and a human-approval gate.

---

### Lab 3 — Concurrency control: prevent overlapping deploys

1. Add a concurrency block to the deploy job:
   ```yaml
   concurrency:
     group: deploy-production
     cancel-in-progress: false
   ```
2. Trigger two pushes to `main` in quick succession (e.g., two empty commits a few seconds apart) and observe the second run queue behind the first in the Actions UI instead of running concurrently.
3. Now change `cancel-in-progress` to `true` on a separate, non-deploy CI job (`group: ci-${{ github.ref }}`) and push twice quickly — confirm the first run gets cancelled instead of queuing.

**Success criteria:** You can explain, using your own two test runs as evidence, the difference in observed behavior between `cancel-in-progress: true` and `false`.

---

### Lab 4 — Interactive debugging with `tmate`

1. Add a deliberately-failing step to a throwaway workflow, followed by a gated debug step:
   ```yaml
   - run: exit 1
   - if: failure()
     uses: mxschmitt/action-tmate@v3
     timeout-minutes: 5
   ```
2. Push, watch the Actions log for the `ssh <session>@nyc1.tmate.io` connection string, and connect from your local terminal.
3. Inside the session, inspect the failed environment (`env`, `ls`, `cat` any relevant files) to see how you'd diagnose a real failure this way.
4. Exit the session (or let the 5-minute timeout expire) and confirm the job completes and doesn't hang the pipeline.

**Success criteria:** You've connected to a live GitHub Actions runner via SSH at least once and understand what real information it exposes that plain log output doesn't.

---

### Cleanup

```bash
# Remove the lab IAM role and OIDC provider if this was a throwaway account
aws iam detach-role-policy --role-name github-actions-deploy --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
aws iam delete-role --role-name github-actions-deploy

# Remove the test GitHub Environment
# (Settings -> Environments -> production -> Delete environment)
```

### Stretch challenge

Scope the OIDC trust policy's `sub` condition to a specific GitHub Environment instead of a branch (`repo:org/repo:environment:production`), and confirm in the AWS CloudTrail event history that the `AssumeRoleWithWebIdentity` call's claims reflect the Environment name.
