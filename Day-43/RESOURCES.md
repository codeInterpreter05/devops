# Day 43 — Resources: AWS Monitoring & Cost

## Primary (assigned)

- **AWS Cost Optimization documentation** (aws.amazon.com/aws-cost-management) — free, the assigned starting point. Covers Cost Explorer, Budgets, and the commitment-discount landscape directly from the source.

## Deepen your understanding

- **CloudWatch Logs Insights query syntax reference** (AWS docs) — the authoritative reference for `stats`, `parse`, `filter`, and `bin()` functions used in the queries in this day's notes.
- **AWS Well-Architected Framework — Cost Optimization Pillar** — the conceptual framework behind everything in this day (matches nicely with Day 44's Reliability Pillar).
- **"Understanding your AWS Cost and Usage Report" (AWS docs)** — walks through CUR schema and setting up Athena queries against it, the path beyond what Cost Explorer's UI can show you.
- **Infracost documentation** (infracost.io/docs) — practical guide to wiring cost estimation into a Terraform CI pipeline.

## Reference / lookup

- **AWS Config Managed Rules list** (docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html) — searchable list of every built-in compliance rule, so you check before writing a custom Lambda rule.
- **Trusted Advisor check reference** (AWS docs) — full list of checks by category and which support tier unlocks them.
- **AWS Pricing Calculator** (calculator.aws) — for one-off "what would this cost" estimates outside of a Terraform workflow.

## Practice

- **AWS Cost Explorer on your own account** — even a free-tier/sandbox account has enough billing history after a few days to practice grouping/filtering by service, tag, and usage type.
- **Deploy a small Config conformance pack** (e.g., the AWS-provided "Operational Best Practices for CIS AWS Foundations Benchmark" pack) in a sandbox account and review what it flags — much more instructive than reading the rule list.
