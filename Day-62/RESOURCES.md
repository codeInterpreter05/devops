# Day 62 — Resources: Advanced GitHub Actions

## Primary (assigned)

- **GitHub OIDC with AWS documentation** (docs.github.com/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) — free, the assigned starting point. Walks through the exact trust policy and workflow configuration covered in file 1.

## Deepen your understanding

- **`aws-actions/configure-aws-credentials` GitHub repo README** (github.com/aws-actions/configure-aws-credentials) — the authoritative reference for every input option, including how it chooses between OIDC and static-key auth and how it handles role chaining.
- **Actions Runner Controller (ARC) documentation** (github.com/actions/actions-runner-controller) — the canonical guide for running self-hosted runners on Kubernetes/EKS, including the `gha-runner-scale-set` Helm chart used in today's notes.
- **GitHub "Security hardening for GitHub Actions"** (docs.github.com/actions/security-guides/security-hardening-for-github-actions) — covers self-hosted runner risks on public repos, and required-reviewer/Environment security patterns in one place.
- **`mxschmitt/action-tmate` GitHub repo** (github.com/mxschmitt/action-tmate) — README covers the full set of options, including limiting the debug session to specific users via `limit-access-to-actor`.

## Reference / lookup

- **GitHub Environments documentation** (docs.github.com/actions/deployment/targeting-different-environments/using-environments-for-deployment) — full reference on required reviewers, wait timers, and branch/tag deployment restrictions.
- **`concurrency` syntax reference** (docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions#concurrency) — exact behavior of `group`/`cancel-in-progress` at workflow vs. job level.

## Practice

- Set up a real (even if minimal) IAM role with OIDC trust in a personal AWS sandbox account, deliberately misconfigure the `sub` claim once, and observe the exact `AccessDenied` error — the fastest way to build intuition for debugging this in production.
- Trigger two rapid pushes against a workflow with `cancel-in-progress: true` and separately against one with `false`, and compare the Actions UI's run history for each — seeing the cancellation behavior firsthand cements it far better than reading about it.
