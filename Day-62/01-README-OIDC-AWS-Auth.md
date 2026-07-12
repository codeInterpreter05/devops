# Day 62 — Advanced GitHub Actions: OIDC Authentication to AWS

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** CI/CD | **Flag:** None

## Brief

Storing long-lived AWS access keys as GitHub secrets is one of the most common real-world breach vectors in CI/CD — a leaked or over-scoped static key sitting in a repo's secrets is a standing liability with no expiry and no built-in blast-radius control. OpenID Connect (OIDC) federation lets a GitHub Actions workflow request short-lived, scoped AWS credentials **on demand**, with zero long-lived secrets stored anywhere. This is now the expected, interview-tested best practice — "how do you securely authenticate CI to a cloud provider" is asked precisely because so many real breaches trace back to getting this wrong.

This day is split into three files:

1. **This file** — how OIDC federation to AWS actually works, and secrets vs. variables as the surrounding secret-hygiene layer.
2. **[02-README-Self-Hosted-Runners-And-Environments.md](02-README-Self-Hosted-Runners-And-Environments.md)** — self-hosted runners on EKS and Environments with required reviewers.
3. **[03-README-Concurrency-And-Debugging.md](03-README-Concurrency-And-Debugging.md)** — concurrency control and interactive workflow debugging with `tmate`.

## Why static keys in CI are a persistent risk

A static IAM access key/secret pair stored as a GitHub secret:
- **Never expires on its own** — if it leaks (accidentally logged, exfiltrated via a compromised dependency, exposed via a misconfigured `run:` that echoes env vars), it remains valid until someone notices and manually rotates it.
- **Requires manual rotation discipline** — in practice, teams rotate these keys rarely, if ever, because it's a manual, disruptive process.
- **Is repo/org-secret-store-shaped, not identity-shaped** — anyone with write access to the repo (or the ability to modify a workflow file, which is itself often gated more loosely than "who can read prod AWS credentials" should be) can potentially exfiltrate it via a modified workflow run.

## How OIDC federation actually works

GitHub Actions can act as an **OIDC identity provider**. Every workflow run can request a signed JWT token from GitHub's OIDC provider (`token.actions.githubusercontent.com`), asserting claims like "this is workflow X, running on branch Y, in repo Z, triggered by event E." AWS is configured to **trust that specific identity provider** and to hand out temporary credentials via `sts:AssumeRoleWithWebIdentity` — but only to JWTs whose claims match a **trust policy condition** you define.

The flow, end to end:

1. **One-time setup**: register GitHub's OIDC provider in AWS IAM (`token.actions.githubusercontent.com`), and create an IAM role with a trust policy scoped to your specific repo/branch.
2. **At workflow runtime**: the `aws-actions/configure-aws-credentials` action requests a JWT from GitHub's OIDC endpoint, then calls AWS STS `AssumeRoleWithWebIdentity` with that JWT, proving "I am this exact workflow in this exact repo."
3. AWS validates the JWT's signature against GitHub's public keys, checks the trust policy conditions match the JWT claims, and if so, issues **short-lived** (default 1 hour, configurable) temporary credentials.
4. Those credentials are scoped only to whatever IAM permissions the assumed role has — least privilege, defined by you, not by whatever a static key happened to have when it was created.

### AWS IAM setup

```bash
# 1. Create the OIDC identity provider (one-time per AWS account)
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

```json
// 2. Trust policy for the IAM role — THIS is where scoping happens
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

```yaml
# 3. In the workflow — no static keys anywhere
permissions:
  id-token: write   # REQUIRED — grants the job permission to request the OIDC JWT
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: us-east-1
      - run: aws s3 ls   # now authenticated with short-lived, scoped creds
```

### The `sub` claim is the whole security model

The trust policy's `sub` condition is doing all the real work. Common patterns:
- `repo:my-org/my-repo:ref:refs/heads/main` — only workflows running on `main` in this exact repo can assume the role (typical for a prod-deploy role).
- `repo:my-org/my-repo:pull_request` — allow any PR from this repo (looser — usually paired with a lower-privilege role, since PRs from forks have restricted secrets/token scope anyway).
- `repo:my-org/my-repo:environment:production` — scoped to a specific GitHub Environment (see file 2), combining OIDC scoping with Environment-level required reviewers for defense in depth.

Getting the `sub` condition wrong is the single most common OIDC misconfiguration: using `StringEquals` with a wildcard-containing value (silently never matches, since `StringEquals` doesn't support wildcards — you need `StringLike` for `*`), or scoping too broadly (e.g., matching the whole org instead of one repo/branch), which defeats the purpose of fine-grained trust.

## Secrets vs. Variables

GitHub Actions has two separate configuration stores that look similar but serve different purposes:

| | Secrets | Variables |
|---|---|---|
| Visibility in logs | Automatically masked (`***`) | Shown in plain text |
| Intended for | Credentials, tokens, anything sensitive | Non-sensitive config: region, environment name, feature flags |
| Access syntax | `${{ secrets.NAME }}` | `${{ vars.NAME }}` |
| Scope levels | Repo, Environment, Organization | Repo, Environment, Organization |
| Auditability | Value itself is opaque even to maintainers after creation (write-only) | Fully visible/auditable in the UI and API |

Putting non-sensitive config into `secrets` just to "be safe" makes debugging harder (masked values are painful to troubleshoot) and hides configuration that should be transparent (e.g., which AWS region a workflow deploys to is not a secret — it should be a `vars` entry anyone on the team can see without needing secret-read permission).

## Points to Remember

- OIDC eliminates long-lived static AWS keys from CI entirely — credentials are minted per-run, scoped, and expire automatically (default 1 hour).
- `permissions: id-token: write` at the job (or workflow) level is mandatory — without it, the runner cannot request the OIDC JWT at all, and `configure-aws-credentials` fails immediately.
- The trust policy's `sub` condition is the actual access-control boundary — scope it to the tightest reasonable match (specific repo + branch/environment), not the whole org.
- `StringEquals` doesn't support wildcards; use `StringLike` if your `sub` condition needs a `*`.
- Secrets are for credentials (masked, write-only); Variables are for non-sensitive config (visible, auditable) — don't conflate the two.

## Common Mistakes

- Forgetting `permissions: id-token: write` and getting an opaque "Not authorized to perform sts:AssumeRoleWithWebIdentity" error that looks like an IAM problem but is actually a missing workflow permission.
- Scoping the trust policy's `sub` condition too broadly (e.g., matching `repo:my-org/*` or omitting the branch/environment qualifier entirely) — any workflow in any repo in the org can then assume a role meant for one specific production deploy pipeline.
- Using `StringEquals` with a wildcard pattern in the `sub` condition — it silently never matches (no error, just permanent `AccessDenied`), instead of the correct `StringLike`.
- Leaving the old static-key secrets (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`) in the repo "just in case" after migrating to OIDC — an unused-but-valid credential is still a live risk sitting in the secret store.
- Putting genuinely non-sensitive values (region, bucket name, environment label) into `secrets` instead of `vars`, making routine debugging harder for no security benefit.
