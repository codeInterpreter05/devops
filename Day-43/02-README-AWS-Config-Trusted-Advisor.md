# Day 43 — AWS Monitoring & Cost: AWS Config & Trusted Advisor

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** –

## Brief

CloudWatch (previous file) tells you what your infrastructure is *doing*. AWS Config and Trusted Advisor tell you whether your infrastructure is *configured correctly* — one continuously, against rules you define; the other as a periodic best-practices scan across cost, security, performance, and resilience. Both come up in interviews around governance ("how do you stop config drift across 50 accounts?") and are the tools you reach for when an auditor or a security review asks "prove nobody has an open security group."

## AWS Config — continuous configuration compliance

AWS Config records **configuration items (CIs)** — point-in-time snapshots of a resource's configuration and relationships — every time a tracked resource changes. It answers two distinct questions:
1. **"What did this resource look like at time T, and what changed?"** — the configuration timeline/history.
2. **"Is this resource compliant with a rule?"** — evaluated continuously as resources change, or on a schedule.

**How it works mechanically:**
- The **Configuration Recorder** watches supported resource types (you choose "record all resources" or a specific list) and generates CIs on change.
- The **Delivery Channel** ships CIs to an S3 bucket (for history/audit) and optionally an SNS topic (for real-time notification).
- **Config Rules** evaluate CIs against a condition. **Managed rules** are AWS-authored (`s3-bucket-public-read-prohibited`, `restricted-ssh`, `rds-storage-encrypted`) — pick and configure, no code. **Custom rules** are your own Lambda function (or a **Guard** policy-as-code document) that receives a CI and returns `COMPLIANT`/`NON_COMPLIANT`/`NOT_APPLICABLE`.
- **Conformance Packs** bundle many rules + remediations into one deployable template — e.g., a CIS AWS Foundations Benchmark pack you deploy once per account instead of wiring up 40 rules by hand.
- **Remediation Actions** attach an SSM Automation document to a rule, so a non-compliant resource can be **auto-remediated** (e.g., automatically attach an S3 bucket policy to block public access) — turning Config from "detect and alert a human" into "detect and self-heal."
- **Aggregators** roll up compliance data from many accounts/regions into one dashboard — essential once you're past a single-account setup, which by Day 43 in a real org you almost certainly are.

**Why this matters over just "writing a script that checks things":** Config's CIs give you relationship graphs (this EC2 instance is attached to this security group, which is referenced by this ENI) and a full change history for free — reconstructing "who changed this and when" from CloudTrail alone is much harder than querying Config's timeline directly.

## Trusted Advisor — periodic best-practice recommendations

Trusted Advisor scans your account against AWS best practices across **five categories**:
1. **Cost Optimization** — idle load balancers, underutilized EC2/RDS instances, unassociated Elastic IPs, RI/Savings Plan purchase recommendations.
2. **Performance** — service limits approaching, overutilized instances.
3. **Security** — open security groups (e.g., unrestricted SSH/RDP to `0.0.0.0/0`), IAM use (root account MFA, access key rotation), S3 bucket permissions, exposed access keys.
4. **Fault Tolerance** — Multi-AZ configuration checks, EBS snapshot age, Auto Scaling group health checks.
5. **Service Limits (Quotas)** — usage approaching account service quotas.

**The support-tier gotcha:** the **full set** of checks (especially most Security and Fault Tolerance checks) requires a **Business or Enterprise Support plan**. On Basic/Developer support, you only get a handful of core checks (mainly a subset of Security and Service Limits). This is a common interview/practical trap — "why isn't Trusted Advisor showing me anything for cost optimization?" is often just "you're on Basic support."

**Config vs. Trusted Advisor — the distinction that actually matters:**

| | AWS Config | Trusted Advisor |
|---|---|---|
| Trigger | Continuous, on every resource change | Periodic refresh (varies by check, often ~daily or on-demand) |
| Scope | Rules **you** define (or managed rules you enable) | Fixed AWS-curated best-practice checks |
| Output | Compliant/Non-compliant + full history | Green/Yellow/Red status + recommendation |
| Remediation | Can auto-remediate via SSM Automation | Recommendation only — no auto-fix |
| Use case | "Enforce and audit our own policies continuously" | "Tell us where we're deviating from generic AWS best practice" |

In practice, mature teams use Trusted Advisor as a periodic sanity check / cost-saving discovery tool, and Config (with conformance packs + auto-remediation) as the enforced, continuous compliance layer — they're complementary, not redundant.

## Points to Remember

- Config records configuration *changes* continuously; Trusted Advisor runs periodic best-practice *checks* — different cadence, different purpose.
- Config Rules can be managed (AWS-provided) or custom (Lambda/Guard) — and can trigger automatic remediation via SSM Automation documents.
- Conformance Packs bundle rules + remediations for one-shot deployment of a standard (e.g., CIS Benchmark) across accounts.
- Trusted Advisor's full check set requires Business/Enterprise Support — Basic support only shows a limited subset, mostly security and service-limit checks.
- Config Aggregators and Trusted Advisor Organizational View both exist to give a multi-account rollup — don't build a custom cross-account reporting pipeline before checking if these cover it.

## Common Mistakes

- Enabling AWS Config's "record all resource types" in every region without considering cost — Config charges per configuration item recorded, which adds up fast in accounts with high resource churn (e.g., ephemeral Lambda/ECS task configs).
- Assuming Trusted Advisor is "empty" because there are no problems, when actually most checks are simply gated behind a support plan you're not on.
- Writing a custom Lambda to check for public S3 buckets when a managed Config rule (`s3-bucket-public-read-prohibited`) already exists and is battle-tested.
- Turning on remediation actions without a dry-run/notify-only phase first — auto-remediating in production without validating the blast radius (e.g., auto-modifying security groups) can break legitimate traffic.
- Treating Config's compliance dashboard as real-time — depending on rule evaluation type (periodic vs. configuration-change-triggered), there can be a lag before a change is reflected as compliant/non-compliant.
