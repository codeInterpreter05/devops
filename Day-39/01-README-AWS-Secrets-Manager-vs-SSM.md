# Day 39 — Secrets in AWS & K8s: Secrets Manager vs. SSM Parameter Store

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Secrets

## Brief

Yesterday was HashiCorp Vault — a full, self-hosted secrets platform. Today is the AWS-native equivalent, and the reality most teams live in: not everyone runs Vault, and AWS gives you two overlapping-but-different built-in services for storing secrets — **Secrets Manager** and **SSM Parameter Store** — that get confused constantly, including in interviews. Knowing precisely when to reach for which (and why "just use Secrets Manager for everything" is not free) is a practical, everyday AWS decision, not academic trivia.

This day is split into three files:

1. **This file** — AWS Secrets Manager vs. SSM Parameter Store.
2. **[02-README-External-Secrets-Operator-and-Sealed-Secrets.md](02-README-External-Secrets-Operator-and-Sealed-Secrets.md)** — syncing cloud secrets into Kubernetes with ESO, and the GitOps-friendly alternative, Sealed Secrets.
3. **[03-README-SOPS-and-Secrets-Hygiene.md](03-README-SOPS-and-Secrets-Hygiene.md)** — SOPS with age encryption, and the discipline of never committing secrets to git or baking them into images.

## AWS Secrets Manager

Purpose-built for secrets: automatic **rotation**, native integrations with RDS/Redshift/DocumentDB for password rotation, resource policies, and cross-account/cross-region replication.

```bash
aws secretsmanager create-secret --name prod/myapp/db-password --secret-string '{"username":"admin","password":"S3cr3t!"}'
aws secretsmanager get-secret-value --secret-id prod/myapp/db-password --query SecretString --output text
aws secretsmanager rotate-secret --secret-id prod/myapp/db-password --rotation-lambda-arn arn:aws:lambda:...:function:rotate-rds
aws secretsmanager put-secret-value --secret-id prod/myapp/db-password --secret-string '{"password":"NewPass!"}'
```

Key capabilities that justify its (real, per-secret) cost:
- **Built-in rotation** — for RDS/Aurora/Redshift/DocumentDB, AWS provides ready-made Lambda rotation functions; point Secrets Manager at the DB and a rotation schedule, and it handles generating a new password, updating the database, and updating the secret value atomically, without your application needing to do anything beyond re-fetching the current value.
- **Versioning with staging labels** (`AWSCURRENT`, `AWSPENDING`, `AWSPREVIOUS`) — during rotation, both old and new credentials are briefly valid, avoiding a hard cutover that could break in-flight connections.
- **Resource-based policies** — fine-grained, per-secret IAM policies controlling exactly which principals can read which secret, auditable via CloudTrail.
- **Cost**: charged per secret per month (roughly $0.40/secret) plus API call charges — this matters at scale (thousands of secrets) and is the main reason teams don't reflexively put *everything* here.

## SSM Parameter Store

Part of AWS Systems Manager — originally built for general configuration data (feature flags, connection strings, non-sensitive settings), with a **SecureString** parameter type added for secrets.

```bash
aws ssm put-parameter --name /myapp/prod/db-password --value 'S3cr3t!' --type SecureString --key-id alias/myapp-key
aws ssm get-parameter --name /myapp/prod/db-password --with-decryption --query Parameter.Value --output text
aws ssm get-parameters-by-path --path /myapp/prod/ --recursive --with-decryption
```

- **Standard tier is free** for up to 10,000 parameters (throughput-limited); **Advanced tier** costs a small per-parameter fee and supports higher throughput, larger values, and parameter policies (e.g., auto-expiration).
- **SecureString** encrypts the value using KMS — same underlying encryption strength as Secrets Manager, but **no built-in automatic rotation** — you'd have to build that yourself (e.g., a scheduled Lambda that calls `put-parameter`).
- **Hierarchical naming** (`/myapp/prod/db-password`, `/myapp/staging/db-password`) plus `get-parameters-by-path` makes it natural to fetch an entire environment's config tree in one call — a pattern Secrets Manager doesn't offer as elegantly.
- Widely used for **non-secret configuration** too (`/myapp/prod/feature-flags/new-checkout`), which is really its original design center — using it purely as a secrets store is a secondary, later-added use case.

## The decision in practice

| Need | Choose |
|---|---|
| Automatic rotation integration with RDS/Aurora/Redshift | **Secrets Manager** |
| Cross-account or cross-region secret replication | **Secrets Manager** |
| Thousands of simple config values plus a handful of secrets, cost-sensitive | **SSM Parameter Store** |
| Hierarchical fetch of an entire app's config tree in one call | **SSM Parameter Store** (`get-parameters-by-path`) |
| Fine-grained resource policies per individual secret | **Secrets Manager** |
| You already use Parameter Store heavily for config and don't want two systems | **SSM Parameter Store** (`SecureString`), accept manual/custom rotation |

Many real orgs use **both**: Parameter Store for the bulk of configuration (including some low-sensitivity secrets), Secrets Manager specifically for anything that benefits from automatic DB credential rotation. Neither is objectively "better" — they trade off cost, built-in rotation, and integration depth differently.

## Fetching secrets into workloads

Both integrate with ECS task definitions and Kubernetes via ESO (file 2) without an application ever needing the AWS SDK bundled in just to fetch its own secrets:

```json
// ECS task definition snippet
"secrets": [
  { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:prod/myapp/db-password" }
]
```

ECS itself resolves the secret at task launch time and injects it as a plain environment variable inside the container — the container never calls the Secrets Manager API itself, and the plaintext secret never appears in the task definition, CloudFormation template, or Terraform state as a literal value (only the ARN reference does).

## Points to Remember

- Secrets Manager: built for secrets specifically — automatic rotation (especially RDS-family databases), resource policies, cross-account/region replication, per-secret cost.
- SSM Parameter Store: originally general configuration storage, `SecureString` type adds KMS encryption for secrets, free at Standard tier, no built-in rotation, strong at hierarchical config trees.
- Neither service requires baking the AWS SDK into application code just to read a secret at deploy time — ECS/EKS-level integrations (task definitions, ESO) can resolve and inject secrets before the application ever starts.
- "Automatic rotation" in Secrets Manager specifically means AWS-provided Lambda functions that coordinate updating both the database and the stored secret value together, using staging labels to avoid a hard cutover.
- Cost and built-in rotation are the two axes that should actually drive the decision — not "which one sounds more secure" (the encryption strength, via KMS, is comparable for both).

## Common Mistakes

- Assuming SSM Parameter Store's `SecureString` is somehow less secure than Secrets Manager — both use KMS encryption under the hood; the real difference is rotation tooling and pricing, not cryptographic strength.
- Using Secrets Manager for thousands of simple, non-rotating config values purely out of habit, and getting hit with a surprising monthly bill at scale, when Parameter Store's Standard tier would have been free.
- Forgetting SSM Parameter Store has no built-in automatic rotation — teams that store DB passwords there and assume "it's encrypted so it's handled" often go years without any rotation at all.
- Putting a secret's plaintext value directly into a Terraform resource argument or CloudFormation template parameter (even if the *storage* is Secrets Manager/Parameter Store) — it still ends up readable in Terraform state or CFN change sets/history unless handled carefully (state encryption, `sensitive = true`, or writing the initial value out-of-band).
- Not setting resource policies/IAM conditions tightly on Secrets Manager secrets, relying only on broad IAM user/role permissions — any principal with generic `secretsmanager:GetSecretValue` on `*` can read every secret in the account, not just the one it needs.
