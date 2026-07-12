# Day 62 — Quiz: Advanced GitHub Actions

Try to answer without looking at your notes. Answers are at the bottom.

1. What AWS STS API call does `aws-actions/configure-aws-credentials` use under the hood when authenticating via OIDC, and what does it require from the workflow's `permissions` block?
2. What part of the AWS IAM trust policy actually enforces which workflows can assume a role, and give one concrete example of a `sub` claim pattern.
3. Why does `StringEquals` fail silently (rather than error) when used with a wildcard in the `sub` condition, and what should you use instead?
4. Name two concrete advantages of running self-hosted runners as ephemeral pods on EKS instead of using GitHub-hosted runners.
5. Why is it dangerous to let self-hosted runners execute workflows triggered by `pull_request` from forks on a public repo?
6. What three things can a GitHub Environment configure that a plain job cannot?
7. What's the difference between `secrets.*` and `vars.*` in terms of both purpose and log visibility?
8. What does `concurrency.group` control, and why would you include `github.ref` in it for a CI workflow but omit it for a shared production deploy target?
9. What's the practical difference in outcome between `cancel-in-progress: true` and `cancel-in-progress: false` for a deploy pipeline specifically?
10. Why must a `tmate` debugging step always carry a `timeout-minutes`, and why should it typically be gated with `if: failure()` or a manual toggle rather than always running?
11. What's a lower-risk, non-interactive alternative to try before reaching for a full `tmate` session?
12. **Interview question:** How do you securely authenticate a GitHub Actions workflow to AWS without storing credentials?

---

## Answers

1. `sts:AssumeRoleWithWebIdentity`. It requires `permissions: id-token: write` (and typically `contents: read`) in the workflow/job — without `id-token: write`, GitHub won't let the job request the OIDC JWT at all, and the STS call fails before it even reaches AWS.
2. The `Condition` block, specifically the `sub` (subject) claim check — e.g., `"StringLike": {"token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"}` restricts the role to only be assumable by workflows running on `main` in that exact repo.
3. `StringEquals` performs exact string matching only — it has no wildcard/glob support, so a pattern like `repo:org/*` under `StringEquals` never matches anything containing text after `org/`, resulting in permanent (and silent, non-obvious) `AccessDenied`. `StringLike` supports `*`/`?` wildcards and is the correct operator when the `sub` pattern needs to match a range of values.
4. Any two of: elastic scaling (scale to zero when idle, scale up on demand, avoiding paying for constantly-idle EC2 instances); VPC-native network access to private resources (RDS, internal APIs) that GitHub-hosted runners can't reach; full control over the runner's base image/toolchain instead of relying on GitHub's shared, periodically-changing image; potential cost savings at high CI volume.
5. Any external contributor can open a PR, and if a workflow targeting self-hosted runners runs automatically on that PR's `pull_request` event, their (potentially malicious) code executes with access to your actual infrastructure (network, secrets available to the runner, etc.) — effectively handing arbitrary code execution inside your environment to anyone who can open a PR.
6. Required reviewers (pause for human approval), a wait timer (mandatory delay before running), and deployment branch/tag restrictions (only specific branches/tags may target this environment) — plus environment-scoped secrets/variables that differ from the repo's general secrets.
7. `secrets.*` values are automatically masked in logs and intended for credentials/sensitive data (write-only after creation in the UI). `vars.*` values are shown in plain text and intended for non-sensitive configuration (region, environment name, flags) that should be visible/auditable rather than hidden.
8. `concurrency.group` defines which runs are considered "overlapping" and therefore serialized (only one active at a time, others queued or cancelled). Including `github.ref` scopes serialization to a specific branch/PR (so unrelated branches don't block each other) — right for CI. Omitting it (using a fixed name like `deploy-production`) scopes serialization globally across the whole shared target, which is correct when only one deploy to that environment should ever be in flight regardless of which branch triggered it.
9. `cancel-in-progress: true` would abort an in-progress deploy the moment a new one is queued — risking leaving infrastructure in a half-applied/inconsistent state. `false` makes the new deploy wait in a queue until the current one finishes cleanly, which is the safe behavior for anything that mutates shared infrastructure state.
10. Without `timeout-minutes`, an abandoned or forgotten debug session can hold the runner (and any concurrency group it's part of) indefinitely, wasting CI minutes and blocking subsequent runs. Gating with `if: failure()` (or a manual opt-in toggle) prevents every single run from unconditionally opening a live shell — especially important since anyone who can read the workflow log while the session is open can grab the SSH connection string.
11. Enabling verbose debug logging via the `ACTIONS_STEP_DEBUG`/`ACTIONS_RUNNER_DEBUG` secrets (or the "Enable debug logging" option on a manual re-run) — it surfaces much more detailed log output without granting anyone actual shell access to the runner.
12. Strong answer: "Configure AWS IAM to trust GitHub's OIDC provider (`token.actions.githubusercontent.com`), create an IAM role with a trust policy scoped via the `sub` claim to the specific repo/branch/environment that should be allowed to assume it, grant `permissions: id-token: write` in the workflow, and use `aws-actions/configure-aws-credentials` with `role-to-assume` instead of static keys. This gets you short-lived (default 1-hour), automatically-expiring, tightly-scoped credentials instead of a long-lived static key sitting in a secret store indefinitely — eliminating rotation burden and drastically shrinking the blast radius if a workflow or log is ever compromised."
