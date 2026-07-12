# Day 16 — Python for Automation II — AWS: Boto3 & IAM Fundamentals

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Python DevOps | **Flag:** ⚡ Interview-critical

## Brief

Every AWS automation script, CI/CD pipeline, and infra tool eventually boils down to one question: *how does this code prove to AWS who it is, and what is it allowed to do?* Get this wrong and you either leak long-lived credentials (a top-5 cause of real cloud breaches) or write scripts that break the moment they move from your laptop to a CI runner, a Lambda, or an EC2 instance. This is also one of the most reliably asked DevOps/Cloud interview questions — "how does your script authenticate to AWS, and why not just hardcode keys?" — because the answer separates people who've only run `aws configure` once from people who've actually operated production automation. This note covers boto3's core object model and goes deep on IAM authentication mechanics, since that's where the interview-critical content lives.

This day is split into three focused files:

1. **This file** — boto3 fundamentals (sessions, client vs. resource, the credential resolution chain) and IAM authentication in depth: access keys vs. roles, STS `AssumeRole`, EC2 instance profiles, and why role assumption is the production default.
2. **[02-README-S3-With-Boto3.md](02-README-S3-With-Boto3.md)** — S3 operations via boto3: upload/download methods, presigned URLs, multipart uploads, and common gotchas (region mismatches, ACLs vs. bucket policies).
3. **[03-README-EC2-And-AWS-CLI-Profiles.md](03-README-EC2-And-AWS-CLI-Profiles.md)** — EC2 automation (describe/start/stop, pagination, cross-region iteration, tag filtering), AWS CLI named profiles, MFA-protected profiles, and `aws-vault`.

## Boto3's object model: sessions, clients, and resources

**`boto3.Session`** is the object that holds configuration: which credentials to use, which region, which profile. If you never create one explicitly, boto3 creates a default session behind the scenes the first time you call `boto3.client(...)`.

```python
import boto3

# Implicit default session — picks up credentials via the resolution chain below
s3 = boto3.client("s3")

# Explicit session — useful when a script needs to juggle multiple accounts/regions/profiles
session = boto3.Session(profile_name="staging", region_name="ap-south-1")
s3_staging = session.client("s3")
ec2_staging = session.resource("ec2")
```

Every service can be accessed two ways:

| | **Client** (`boto3.client("s3")`) | **Resource** (`boto3.resource("s3")`) |
|---|---|---|
| Level | Low-level — a 1:1 mapping to the AWS API | Higher-level, object-oriented wrapper over the client |
| Style | `s3.list_objects_v2(Bucket="x")` | `bucket = s3.Bucket("x"); bucket.objects.all()` |
| Coverage | Every AWS service, every operation | Only a subset of services (S3, EC2, DynamoDB, SQS, SNS, IAM, CloudFormation, ...) — many newer services have **no** resource interface at all |
| Response | Raw `dict` matching the API's JSON shape | Python objects with attributes/methods, but still backed by client calls under the hood |
| Status | Actively maintained, gets every new API immediately | AWS has stated no new resource models are being added — existing ones still work, but new services/operations only ship as clients |

**Practical guidance:** default to `client` for new code — it's guaranteed to expose whatever the API supports, and its raw-dict responses are easier to reason about in scripts that need to be precise (e.g., checking `response["ResponseMetadata"]["HTTPStatusCode"]`, handling `NextToken` for pagination). Reach for `resource` when the object-oriented style genuinely reads better, most commonly for S3 (`bucket.objects.filter(Prefix="logs/")` reads nicer than looping over `list_objects_v2` pages) — but know that under the hood it's still making the same calls, with the same pagination limits.

## The credential resolution chain

This is the single most interview-relevant boto3 mechanic: when you call `boto3.client("s3")` with no explicit credentials, botocore searches for credentials in a fixed order and uses the **first one it finds** — it does not merge or fall back partially. In order:

1. **Explicit parameters** passed directly to `boto3.client()` or `boto3.Session()` (`aws_access_key_id=`, `aws_secret_access_key=`, `aws_session_token=`).
2. **Environment variables** — `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`.
3. **Shared credential file** — `~/.aws/credentials`, using the profile named by `AWS_PROFILE` (defaults to `[default]`).
4. **AWS config file** — `~/.aws/config`, same profile resolution, can also hold `role_arn`/`source_profile` for assume-role chaining.
5. **Assume-role provider** — if the resolved profile has a `role_arn`, botocore transparently calls STS `AssumeRole` using the `source_profile`'s credentials and caches the resulting temporary token.
6. **Container credentials** — ECS task role, served from a link-local endpoint referenced by `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`.
7. **Instance metadata service (IMDS)** — on an EC2 instance with an IAM instance profile attached, credentials are fetched automatically from `169.254.169.254`.
8. **Web identity token** — EKS IAM Roles for Service Accounts (IRSA): botocore reads `AWS_WEB_IDENTITY_TOKEN_FILE` + `AWS_ROLE_ARN` and calls `AssumeRoleWithWebIdentity`.

**Why this order matters in practice:** it's why a script that works perfectly on your laptop (picks up your `~/.aws/credentials` profile) "just works" unmodified on an EC2 instance or in an ECS task, provided the instance/task has an IAM role attached — no code change, no env var, nothing. The chain resolves to whatever context the code happens to run in. This is the entire architectural argument for never hardcoding credentials: write the script to *not specify credentials at all*, and let the environment (laptop profile, EC2 role, ECS task role, EKS service account) supply them.

## IAM authentication: keys vs. roles

### Long-lived access keys

An IAM user can have up to two active **access key pairs** (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`), generated via the console or `aws iam create-access-key`. They are:

- **Static** — valid until manually deleted or rotated. No built-in expiry.
- **Bearer credentials** — whoever holds them can act as that IAM user, no additional proof required. This is exactly why they leak so often: committed to a public GitHub repo, baked into a Docker image layer, pasted into a Slack message.
- Your responsibility to rotate, store securely, and revoke — none of which AWS does for you automatically.

```bash
aws iam create-access-key --user-name automation-bot
aws iam list-access-keys --user-name automation-bot
aws iam delete-access-key --user-name automation-bot --access-key-id AKIA...
```

### IAM roles and STS `AssumeRole`

A **role** is not a set of static secrets — it's an identity with two attached policies:

- A **trust policy** (who is allowed to assume it) — a resource-based policy on the role itself, specifying a `Principal` (an AWS account, IAM user/role ARN, an AWS service like `ec2.amazonaws.com`, or a federated OIDC/SAML provider).
- A **permissions policy** (what the role can do once assumed) — identical in shape to a normal IAM policy.

Assuming a role calls STS, which hands back **temporary credentials**:

```python
import boto3

sts = boto3.client("sts")

response = sts.assume_role(
    RoleArn="arn:aws:iam::111122223333:role/ci-deploy-role",
    RoleSessionName="github-actions-run-4821",   # shows up in CloudTrail — make it identifiable
    DurationSeconds=3600,                          # 15 min–12 h, capped by the role's MaxSessionDuration
)

creds = response["Credentials"]   # AccessKeyId, SecretAccessKey, SessionToken, Expiration

assumed_session = boto3.Session(
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretAccessKey"],
    aws_session_token=creds["SessionToken"],
)
ec2 = assumed_session.client("ec2", region_name="us-east-1")
```

Key mechanics:

- The returned credentials always include a **`SessionToken`** in addition to the key pair — this is what distinguishes temporary credentials from long-lived ones, and every AWS API call made with them must include it.
- They **expire** — default max is 1 hour unless the role's `MaxSessionDuration` is raised (up to 12 hours). There is no way to make assumed-role credentials permanent.
- **Role chaining** (assuming role B using credentials obtained by assuming role A) caps the session at **1 hour regardless** of either role's configured max — a specific gotcha worth knowing.
- Every assumed session gets a unique `RoleSessionName` that shows up in CloudTrail as `arn:aws:sts::ACCOUNT:assumed-role/ROLE/SESSION-NAME` — this is what makes assumed-role activity individually auditable even though many different callers might assume the same role.
- Trust policies can require MFA at assumption time via a `Condition` block checking `aws:MultiFactorAuthPresent`, which STS enforces before issuing credentials — a control that's simply not possible with static access keys.

### Instance profiles: how EC2 gets credentials without any keys at all

An **instance profile** is the container object that attaches an IAM role to an EC2 instance (created automatically when you attach a role via the console; via CLI it's a distinct object with `create-instance-profile` + `add-role-to-instance-profile`). Once attached, the **Instance Metadata Service (IMDS)** — a link-local HTTP endpoint at `169.254.169.254`, reachable only from inside the instance — serves temporary credentials for that role automatically:

```bash
# IMDSv2 (token-based — required by default on new instances since 2021+)
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/ci-deploy-role
```

botocore's default credential chain checks IMDS automatically (step 7 above) and **refreshes credentials in the background before they expire** — application code never sees a `sts.assume_role` call at all, it's handled transparently. This is precisely why "just attach the right role to the instance and call `boto3.client(...)` with no arguments" is the correct pattern for anything running on EC2/ECS/EKS.

**IMDSv1 vs. IMDSv2:** IMDSv1 answers plain unauthenticated `GET` requests to the metadata endpoint — which made it exploitable via server-side request forgery (SSRF): a vulnerable web app running on the instance that proxies arbitrary URLs could be tricked into fetching the instance's IAM credentials (this is exactly how the 2019 Capital One breach happened). **IMDSv2 requires a session token obtained via a `PUT` request first**, which most SSRF vulnerabilities can't forge (SSRF typically only allows GET-style requests, and PUT responses aren't reflected back to the attacker). Enforce IMDSv2-only via the instance's metadata options (`HttpTokens: required`) — this is a standard hardening step and a common interview/security-review question.

### Why roles are preferred over keys, concretely

- **No secret to leak.** Temporary credentials rotate automatically; even if captured, they expire within hours.
- **No manual rotation burden.** Static keys require you to remember to rotate them; roles require nothing.
- **Fine-grained, revocable trust.** The trust policy scopes exactly who/what can assume a role; revoking access means editing the trust policy or deleting the role, not chasing down every place a key was copied.
- **Auditable per-session.** Every assumption is a distinct, named CloudTrail event, unlike a shared static key used from many places indistinguishably.
- **MFA and other conditions enforceable at the point of assumption** — impossible with bearer-style static keys.

### `aws-vault`

`aws-vault` (a CLI tool, not a boto3 concept) solves the problem of *where your long-lived keys sit at rest on your laptop*. Plaintext `~/.aws/credentials` is a soft target — malware, a misconfigured backup, or a careless `cat` in a screen-share can expose it. `aws-vault`:

- Stores the actual long-lived key pair **encrypted in the OS-native secret store** (macOS Keychain, Windows Credential Manager, or an encrypted file backend on Linux) — never as plaintext on disk.
- On every invocation, uses those stored keys to call STS (`AssumeRole` or `GetSessionToken`) and injects only **temporary** credentials as environment variables into the child process it launches — the long-lived keys themselves are never exposed to the command being run.
- Integrates MFA prompts directly into that flow and caches the resulting session so you're not re-prompted for every command.

```bash
aws-vault add dev                       # prompts once, stores the long-term key pair encrypted
aws-vault exec dev -- aws s3 ls          # runs `aws s3 ls` with temporary creds injected as env vars
aws-vault exec dev --mfa-token=123456 -- terraform apply
```

It is the practical, laptop-side counterpart to "prefer roles over keys" — when a role genuinely can't be used (e.g., local development where there's no EC2/ECS/EKS context to inherit from), `aws-vault` at least ensures the *unavoidable* long-lived key never touches disk unencrypted and every actual API call still uses short-lived STS credentials.

## Points to Remember

- `client` = 1:1 low-level API mapping, works for every service; `resource` = object-oriented convenience layer, only for a subset of services, receiving no new features going forward. Default to `client` for new automation code.
- The credential chain is ordered and exclusive — the first source found wins entirely; it does not merge partial credentials from multiple sources.
- Assumed-role credentials always include a `SessionToken` and always expire (max 12h, or 1h if role-chained) — there is no way to make them permanent.
- Instance profiles let EC2/ECS/EKS workloads get auto-rotating credentials from IMDS/task metadata with zero credential-handling code — this is the production default, not `aws configure` on the box.
- IMDSv2 (token-based) closes the SSRF-to-credential-theft path that IMDSv1 left open — enforce `HttpTokens: required` on instances.
- `aws-vault` doesn't replace IAM roles; it hardens the one case where a long-lived key is genuinely unavoidable (local dev) by keeping it encrypted at rest and only ever exposing short-lived STS creds to actual commands.

## Common Mistakes

- Hardcoding `aws_access_key_id`/`aws_secret_access_key` directly in a script "just for now" — it's the single most common way credentials end up committed to git history (and git history is forever, even after a later "remove secret" commit).
- Assuming an EC2 instance needs `aws configure` run on it — this creates a static, unrotated key sitting in `~/.aws/credentials` on a machine that should instead have an IAM instance profile attached and use the credential chain's IMDS step.
- Not setting a `RoleSessionName` that identifies the caller (e.g., leaving it as some generic default) — makes CloudTrail investigation of "who did this" far harder when several systems assume the same role.
- Forgetting that role chaining caps session duration at 1 hour, then being surprised when a long-running job assuming role B (via already-assumed role A) gets `ExpiredToken` errors much sooner than expected.
- Treating IMDSv1 as fine because "it still works" — leaving it enabled (rather than enforcing IMDSv2-only) leaves an SSRF-to-credential-exfiltration path open on any web service running on that instance.
- Confusing `aws-vault` with a credential-elimination tool — it still requires an initial long-lived key to bootstrap; the value is encrypting it at rest and forcing every actual AWS call through short-lived STS tokens, not removing the root credential entirely.
