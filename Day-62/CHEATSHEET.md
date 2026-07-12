# Day 62 — Cheatsheet: Advanced GitHub Actions

## OIDC to AWS — full setup

```bash
# 1. Register GitHub's OIDC provider (one-time per AWS account)
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

```json
// 2. Trust policy — the actual access-control boundary
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"},
      "StringLike": {"token.actions.githubusercontent.com:sub": "repo:org/repo:ref:refs/heads/main"}
    }
  }]
}
```

Common `sub` patterns:
```
repo:ORG/REPO:ref:refs/heads/main          # only main branch
repo:ORG/REPO:pull_request                  # any PR
repo:ORG/REPO:environment:production        # only jobs bound to this Environment
repo:ORG/REPO:ref:refs/tags/v*              # tag-based (needs StringLike, not StringEquals)
```

```yaml
# 3. Workflow — no static keys
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-actions-deploy
      aws-region: us-east-1
```

## Self-hosted runners (ARC on EKS)

```yaml
# gha-runner-scale-set values
githubConfigUrl: https://github.com/my-org/my-repo
githubConfigSecret: gha-runner-secret
minRunners: 0
maxRunners: 10
runnerScaleSetName: eks-runners
```

```yaml
# target the pool in a workflow
jobs:
  deploy:
    runs-on: eks-runners
```

## GitHub Environments

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://app.example.com
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```
Configure in Settings -> Environments:
- Required reviewers
- Wait timer
- Deployment branch/tag restrictions
- Environment-scoped secrets/variables

## Secrets vs Variables

```yaml
${{ secrets.MY_SECRET }}   # masked in logs, write-only after creation
${{ vars.MY_VAR }}          # plain text, visible, auditable
```

## Concurrency control

```yaml
# CI: cancel stale runs on new pushes to same branch/PR
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Deploy: queue, never cancel mid-flight
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

Job-level scoping (only this job serializes):
```yaml
jobs:
  deploy:
    concurrency:
      group: deploy-production
      cancel-in-progress: false
```

## `tmate` interactive debugging

```yaml
- name: Debug on failure
  if: failure()
  uses: mxschmitt/action-tmate@v3
  timeout-minutes: 15
```

```yaml
# opt-in via manual dispatch instead of always-on
on:
  workflow_dispatch:
    inputs:
      debug_enabled: { type: boolean, default: false }
steps:
  - if: ${{ github.event.inputs.debug_enabled == 'true' }}
    uses: mxschmitt/action-tmate@v3
```

## Lighter-weight debug logging (try before tmate)

```
# Set as repo secrets to enable verbose logs on next run:
ACTIONS_STEP_DEBUG=true
ACTIONS_RUNNER_DEBUG=true
```
Or use "Re-run jobs -> Enable debug logging" in the Actions UI.
