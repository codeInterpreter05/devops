# Day 49 — AWS Security Services: Audit Logging & Encryption

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

CloudTrail and KMS are the two services that almost every other AWS security control depends on. GuardDuty's API-based detections are built on CloudTrail data. Every "who did this and when" incident-response question starts with CloudTrail. Every "is our data encrypted, and who can decrypt it" compliance question starts with KMS. If you only remember two AWS security services cold for an interview, these are strong candidates because they're foundational rather than optional add-ons.

## CloudTrail — the account's audit log

CloudTrail records **every API call** made in your account — through the console, CLI, SDKs, or by other AWS services calling APIs on your behalf — as an immutable event record containing who (IAM identity/role), what (API action), when, source IP, and request/response parameters.

Two event categories matter:
- **Management events** (control-plane) — creating/deleting/modifying resources: `RunInstances`, `CreateBucket`, `PutBucketPolicy`, `AssumeRole`. **Enabled by default** and free for the last 90 days via **Event history** in the console, no trail required.
- **Data events** (data-plane) — high-volume operations on the actual data inside a resource: S3 `GetObject`/`PutObject`, Lambda `Invoke`, DynamoDB item-level operations. **Not enabled by default** (cost/volume reasons) — you must explicitly configure a trail to capture these, and they're what you need if the question is "did someone read this specific S3 object."

**A "trail"** is the durable configuration that ships events to an S3 bucket (and optionally CloudWatch Logs for near-real-time alerting, and EventBridge for automation) for retention beyond 90 days. Key trail concepts:
- **Single-region vs. multi-region trail** — always use multi-region in practice; an attacker who compromises credentials will often pivot to an unusual/unmonitored region specifically because teams forget to enable trails everywhere.
- **Organization trail** — one trail, created in the Organizations management account, that automatically applies to every member account. This is the real-world default for any account with more than a handful of AWS accounts — it removes the "did every account remember to configure this" problem entirely.
- **Log file integrity validation** — CloudTrail can generate a hash-chained digest file alongside your logs so you can cryptographically prove logs weren't tampered with after the fact — this matters for forensics and for compliance frameworks that require tamper-evidence.
- **CloudTrail Lake** — a managed, SQL-queryable event data store, useful when you want to run ad-hoc investigative queries without standing up Athena/S3 querying yourself.

**Why CloudTrail is foundational, not optional**: GuardDuty's `Recon:IAMUser/*` and most `UnauthorizedAccess` findings are literally CloudTrail events being pattern-matched by GuardDuty's ML models. Without CloudTrail data existing, GuardDuty has nothing to analyze for API-level threats — it's the raw material, not a competing/redundant service.

**S3 bucket protection for the trail itself is critical**: if an attacker gains sufficient access to disable or delete your trail (or the S3 bucket storing its logs), they can operate without a paper trail. Best practice: the trail's destination bucket should live in a separate, tightly locked-down account (often the "log archive" account in a multi-account landing zone), with a bucket policy denying delete, and MFA-delete enabled on the bucket.

## KMS — Key Management Service

KMS is how AWS lets you manage encryption keys **without ever exposing the raw key material to you or to AWS operators** — keys live inside FIPS 140-2 validated HSMs (Hardware Security Modules), and all cryptographic operations (encrypt, decrypt, sign) happen inside the HSM boundary; the plaintext key never leaves it.

The interview-critical distinction: **AWS managed keys vs. Customer managed keys (CMKs)**.

| | AWS managed key | Customer managed key (CMK) |
|---|---|---|
| Created by | AWS automatically, first time you use a service needing encryption (e.g., `aws/s3`, `aws/ebs`) | You, explicitly |
| Naming | `aws/<service>` alias, fixed | Any alias you choose |
| Rotation | Automatic, yearly, mandatory, no control | Optional automatic yearly rotation (you can enable/disable), or manual rotation |
| Key policy control | You cannot modify the key policy | You fully control the key policy — who can use/manage it |
| Cross-account sharing | Cannot be shared to another account | Can grant another account's principals permission to use the key |
| Deletion | Cannot be scheduled for deletion by you | You can schedule deletion (7–30 day waiting period, cannot be undone after) |
| Cost | Free | Monthly fee per key + per-API-call charges |

**Why CMKs matter in practice**: the moment you need cross-account access to encrypted data (e.g., a shared S3 bucket for a data pipeline spanning two accounts), fine-grained control over who can decrypt (separate from who can merely use the S3 bucket via IAM), or an audit requirement to control/rotate your own keys, AWS managed keys can't do it — you need a CMK.

**Envelope encryption** is the mechanism underneath almost every KMS integration and is a common "explain how this actually works" interview probe:
1. KMS generates a **data key**: a plaintext data key + that same key encrypted under your CMK (the "encrypted data key").
2. The calling service (S3, EBS, RDS, your own app via the SDK) uses the **plaintext data key** to encrypt your actual data locally, then **discards the plaintext data key from memory** and stores only the **encrypted data key** alongside the ciphertext.
3. To decrypt later, the service sends the encrypted data key back to KMS, KMS decrypts it (inside the HSM, using the CMK) and returns the plaintext data key, which is then used to decrypt the actual data.

This design means your (potentially huge) data never has to leave the calling service to be encrypted, KMS never has to touch your actual data (only tiny data keys), and each object/volume can have its own unique data key even though they all trace back to one CMK — limiting blast radius if a single data key were ever compromised.

**Grants vs. key policies vs. IAM policies**: KMS access control is the classic "three ways to grant permission" AWS gotcha. A principal needs *both* an IAM policy allowing `kms:Decrypt` *and* the CMK's key policy allowing that principal (key policies act as the resource-based policy and are authoritative — a permissive IAM policy alone is not sufficient if the key policy doesn't also allow it, unless the key policy delegates trust to IAM via the common `"Enable IAM User Permissions"` statement). **Grants** are a separate, more granular, often temporary mechanism used for programmatic/service-to-service delegation (e.g., an ASG letting instances decrypt an EBS volume) without editing the key policy each time.

## Points to Remember

- CloudTrail management events are on by default (90-day Event history, free); data events (S3 object-level, Lambda invokes) require an explicit trail and cost more — know which one answers "who read this specific object."
- Use a multi-region, organization-level trail with a locked-down destination bucket in a separate account — single-region/single-account trails are how attackers evade detection by pivoting regions or destroying evidence.
- GuardDuty's API-based findings are built on top of CloudTrail data — CloudTrail is foundational infrastructure, not a competing service.
- AWS managed keys are free, auto-rotated yearly, and you cannot control their policy or share them cross-account; CMKs cost money but give you full policy control, cross-account sharing, and deletion control.
- Envelope encryption means KMS only ever handles small data keys, not your actual data — the CMK never directly encrypts a multi-GB object; it encrypts the (small) data key that does.
- KMS access requires both an IAM policy *and* the key policy to allow the action — a common source of "access denied" confusion.

## Common Mistakes

- Believing CloudTrail is "always capturing everything" and being surprised no data event exists for an S3 `GetObject` because data events were never enabled on a trail.
- Leaving a trail as single-region and single-account, missing activity in a region nobody normally checks, or losing all trail history because the log bucket lived in the same (compromised) account.
- Using AWS managed keys for a use case that later needs cross-account sharing or custom key policies, then having to migrate encrypted data to a new CMK after the fact — decide CMK vs. managed key based on future sharing/rotation needs up front, not just "what's free today."
- Granting `kms:Decrypt` via IAM policy only and forgetting the key policy also needs to allow it (or delegate to IAM), leading to confusing `AccessDeniedException` errors that look like an IAM problem but are actually a key-policy problem.
- Scheduling CMK deletion without realizing the mandatory waiting period (7–30 days) is the *only* safety net — after it elapses, any data encrypted under that key becomes permanently unrecoverable, including old snapshots/backups nobody remembered depended on it.
- Enabling automatic annual key rotation on a CMK and assuming old ciphertext needs re-encrypting — it doesn't; KMS transparently keeps prior key material versions available to decrypt data encrypted before rotation.
