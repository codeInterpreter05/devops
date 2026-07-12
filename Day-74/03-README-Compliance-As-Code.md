# Day 74 — Compliance & Audit: Compliance as Code (Prowler & AWS Security Hub)

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** DevSecOps

## Brief

Manual compliance — a human clicking through the AWS console with a spreadsheet checklist once a quarter — doesn't scale and produces evidence that's stale the moment it's collected. "Compliance as code" means encoding CIS/SOC2/GDPR-relevant checks into tools that run continuously and produce machine-readable, timestamped, re-runnable evidence. This file covers the practical center of that: designing an audit logging strategy, running Prowler against a real AWS account, and understanding how AWS Security Hub aggregates multiple compliance standards into one dashboard. This is today's actual hands-on deliverable — running Prowler for real and fixing findings — so understanding the mechanics here directly maps to the lab.

## Audit logging strategy

Before any compliance tool matters, you need logs to audit — and "turn on logging everywhere" isn't a strategy, it's noise. A real audit logging strategy answers four questions per log source:

1. **What must be logged?** At minimum: authentication events (success and failure), authorization decisions (especially denials), administrative/configuration changes, and data access to sensitive resources.
2. **Where does it go?** Centralized, write-once storage separate from the systems being audited — logs living only on the host they describe can be tampered with or deleted by whoever compromised that host. In AWS: CloudTrail to a dedicated logging account's S3 bucket with **MFA delete** and versioning enabled; in Kubernetes: audit logs shipped off-cluster to a SIEM or log aggregator, not just sitting in a pod's stdout.
3. **How long is it retained, and is that documented?** SOC2/GDPR auditors ask for a *written* retention policy, not just "we keep logs." Common baseline: 1 year for CloudTrail/audit logs, sometimes longer for specific regulated data.
4. **Who/what alerts on it?** A log nobody looks at until after an incident isn't a detective control, it's an archive. Tie critical events (root account usage, IAM policy changes, security group changes allowing 0.0.0.0/0) to real-time alerting (EventBridge → SNS/Slack, or a SIEM rule).

```bash
# Minimum viable AWS audit logging baseline
aws cloudtrail create-trail --name org-trail --s3-bucket-name org-cloudtrail-logs \
  --is-multi-region-trail --enable-log-file-validation
aws cloudtrail put-event-selectors --trail-name org-trail \
  --event-selectors '[{"ReadWriteType": "All", "IncludeManagementEvents": true}]'
```

`--enable-log-file-validation` is what makes CloudTrail logs **tamper-evident** — it generates a cryptographic digest file hourly that lets you later prove a given log file wasn't altered, which is exactly the kind of integrity guarantee an auditor (or an incident responder) needs to trust the log at all.

## Prowler: automated compliance scanning for AWS

Prowler is an open-source CLI that runs hundreds of checks against a live AWS account and maps each one directly to compliance frameworks — CIS AWS Benchmark, SOC2, GDPR, ISO 27001, PCI-DSS, NIST 800-53 — so a single scan produces a report you can hand to multiple auditors.

```bash
# Install and run against the current AWS CLI profile
pip install prowler
prowler aws --compliance cis_2.0_aws

# Run only checks mapped to SOC2
prowler aws --compliance soc2_aws

# Full account scan, output as HTML + CSV for a compliance report
prowler aws -M csv,html,json-ocsf -o ./prowler-report/
```

Sample finding structure (conceptually):

```
Check ID: iam_root_hardware_mfa_enabled
Severity: HIGH
Status: FAIL
Region: us-east-1
Resource: <root account>
Remediation: Enable a hardware MFA device for the root account.
Compliance: CIS 1.5, SOC2 CC6.1
```

Each finding lists **which framework(s) it maps to** — this is the "compliance as code" payoff: a single `iam_root_hardware_mfa_enabled` check satisfies evidence requirements for CIS *and* SOC2 simultaneously, and re-running the scan monthly (or on a schedule via CI) produces a running evidence trail for free.

### Fixing top findings (typical highest-impact items)

Common Prowler findings and their fixes, roughly in order of how often they show up as top offenders on a first scan:

1. **Root account has no hardware/virtual MFA** → enable MFA on root immediately; root should never be used for daily operations at all (IAM users/roles should do everything).
2. **S3 buckets publicly accessible** → enable S3 Block Public Access at the account level, then per-bucket as needed.
3. **CloudTrail not enabled in all regions** → create a multi-region trail (see above); a single-region trail misses activity in regions you don't normally use, which is exactly where an attacker who compromised credentials might operate to avoid detection.
4. **Security groups allow unrestricted ingress (0.0.0.0/0) on SSH/RDP** → restrict to known IP ranges or remove direct SSH/RDP exposure entirely in favor of SSM Session Manager / a bastion.
5. **IAM password policy too weak / no rotation requirement** → enforce via `aws iam update-account-password-policy` (minimum length, complexity, age).
6. **EBS/RDS not encrypted** → enable encryption by default at the account level (`aws ec2 enable-ebs-encryption-by-default`) so all *new* volumes are encrypted; existing unencrypted volumes require a snapshot-copy-with-encryption migration.

## AWS Security Hub: aggregating standards

Security Hub is AWS's native service for continuously running compliance checks and aggregating findings from multiple sources (its own checks, GuardDuty, Inspector, Macie, and third-party tools like Prowler via the ASFF format) into one dashboard with a normalized **security score**.

```bash
aws securityhub enable-security-hub
aws securityhub batch-enable-standards --standards-subscription-requests \
  '[{"StandardsArn": "arn:aws:securityhub:us-east-1::standards/cis-aws-foundations-benchmark/v/1.4.0"}]'
aws securityhub get-findings --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}'
```

Security Hub's advantage over a point-in-time Prowler run is that it's **continuous and native** — checks re-run automatically on a schedule and findings update in near-real-time as configuration changes, and it can auto-remediate certain findings via EventBridge rules triggering Lambda/SSM Automation documents. Its disadvantage is cost (per-check, per-region pricing at scale) and being AWS-only, whereas Prowler is free and increasingly supports Azure/GCP/Kubernetes too — many teams run both: Security Hub for continuous native monitoring, Prowler in CI for point-in-time gate checks and multi-cloud/multi-framework reporting.

## Points to Remember

- A logging strategy isn't "turn everything on" — define what's logged, where it's centralized (off the audited system), retention duration (documented), and what alerts on it in real time.
- `--enable-log-file-validation` on CloudTrail is what makes logs tamper-evident (cryptographic digest files) — without it, you can't prove logs weren't altered after the fact.
- Prowler maps every check to multiple frameworks simultaneously (CIS, SOC2, GDPR, PCI-DSS, etc.) — one scan produces evidence reusable across audits.
- Root account should have hardware MFA and essentially never be used operationally — it's consistently the highest-severity, most-common first finding in any account.
- Security Hub aggregates continuous, native AWS compliance checking across multiple standards and other services (GuardDuty, Inspector, Macie); Prowler is broader (multi-cloud, free, CI-friendly) but point-in-time unless scheduled.

## Common Mistakes

- Enabling CloudTrail in only the primary/default region, missing activity (including malicious activity deliberately routed through an unused region) elsewhere in the account.
- Treating a Prowler or Security Hub scan as a one-time compliance checkbox instead of scheduling it (via CI cron or Security Hub's continuous checks) to produce ongoing evidence for Type II-style audits.
- Fixing S3 public-access findings bucket-by-bucket while leaving the account-level Block Public Access setting off, so any newly created bucket reintroduces the same risk.
- Never rotating or hardware-MFA-protecting the root account because "we never use it" — root credentials being unused daily doesn't reduce their blast radius if compromised; it's still the single highest-privilege identity in the account.
- Logging extensively but never wiring alerts to any of it — discovering a root-account login or a security-group change from a log query during incident response instead of from a real-time alert the moment it happened.
