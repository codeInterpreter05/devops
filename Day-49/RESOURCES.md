# Day 49 — Resources: AWS Security Services

## Primary (assigned)

- **AWS Security documentation hub** (aws.amazon.com/security) — the assigned starting point; central index into GuardDuty, Security Hub, WAF, Shield, Macie, CloudTrail, and KMS docs, each with concept guides and API references.

## Deepen your understanding

- **AWS GuardDuty User Guide — Finding types** (docs.aws.amazon.com/guardduty) — the full catalog of finding names; skim it once so unfamiliar finding names during an incident don't feel like a foreign language.
- **AWS Security Hub User Guide — CIS AWS Foundations Benchmark controls** — read the actual control list once; many of these (root MFA, no open SSH to 0.0.0.0/0, S3 public access block) are exactly what interviewers probe for as "security basics."
- **AWS Well-Architected Framework — Security Pillar** (whitepaper, free PDF) — ties GuardDuty/Security Hub/WAF/Shield/Macie/CloudTrail/KMS together into a coherent design philosophy rather than a list of disconnected services.
- **"KMS Envelope Encryption" — AWS docs deep dive** — walks through the actual API sequence (`GenerateDataKey`, local encrypt, discard, store encrypted data key) that underlies S3/EBS/RDS encryption-at-rest.

## Reference / lookup

- `aws guardduty help`, `aws securityhub help`, `aws wafv2 help`, `aws kms help`, `aws cloudtrail help` — on-box CLI reference for exact parameter names, no internet needed once the CLI is installed.
- **AWS Security Blog** (aws.amazon.com/blogs/security) — real incident-response patterns and new feature announcements as they ship.

## Practice

- **AWS Free Tier sandbox account** — GuardDuty has a 30-day free trial, Security Hub and Macie have free trials too; run today's lab end-to-end in a disposable sandbox account so you're not guessing at console screenshots.
- **flaws.cloud / flaws2.cloud** — free, gamified AWS security misconfiguration challenges (not GuardDuty-specific, but excellent for building the "what does a real misconfiguration look like" intuition that Macie/Security Hub are built to catch).
