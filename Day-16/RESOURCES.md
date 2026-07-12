# Day 16 — Resources: Python for Automation II — AWS

## Primary (assigned)

- **Boto3 official documentation** (boto3.amazonaws.com/v1/documentation/api/latest/index.html) — the authoritative reference for every client/resource method, parameter, and response shape. You will come back to this constantly; get comfortable navigating straight to a service's client reference (e.g., search "boto3 EC2 client describe_instances") instead of guessing parameter names.
- **Real Python — "Python, Boto3, and AWS S3: Demystified"** (realpython.com/python-boto3-aws-s3) — the assigned tutorial; walks through client vs. resource, uploading/downloading, and permissions with runnable examples. Good companion to file 2 of today's notes.

## Deepen your understanding

- **AWS docs — "IAM roles"** (docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) — the concept-level explanation of trust policies vs. permission policies that today's notes build on.
- **AWS docs — STS `AssumeRole` API reference** (docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) — exact request/response shape, parameter limits (session duration bounds, policy size limits for session policies).
- **AWS docs — "Instance metadata and user data"** (docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) — the full IMDSv1/IMDSv2 mechanics, including how to enforce token-required mode account-wide.
- **`aws-vault` README** (github.com/99designs/aws-vault) — written by the tool's maintainers; explains exactly what problem it solves and how the encrypted-storage + short-lived-session model works under the hood.

## Reference / lookup

- **Boto3 — "Credentials" guide** (boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) — the canonical statement of the credential resolution chain order, referenced directly in file 1 of today's notes.
- **AWS CLI — "Configuration and credential file settings"** (docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) — every field `~/.aws/config`/`~/.aws/credentials` support, including `role_arn`, `source_profile`, `mfa_serial`, and `credential_process`.
- **AWS docs — "IAM best practices"** (docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) — AWS's own current guidance on preferring roles/temporary credentials over long-lived access keys, and rotating any keys you can't avoid.

## Practice

- **AWS Free Tier** (aws.amazon.com/free) — sign up if you haven't; everything in today's lab (IAM, STS, S3, and modest EC2 usage) fits comfortably within it.
- **LocalStack** (github.com/localstack/localstack) — runs a local emulation of S3, EC2, IAM, STS, and dozens of other AWS services in Docker; lets you run and break boto3 scripts (including AssumeRole flows) with zero AWS cost or account risk while you're still building confidence.
- **AWS Skill Builder — free digital courses** (explore.skillbuilder.aws) — official AWS training platform; has free, self-paced modules covering IAM fundamentals and the Boto3/CLI basics at a hands-on level.
