# Day 114 — Resources: Cloud Cost Engineering

## Primary (assigned)

- **AWS Cost Optimization Hub** (docs.aws.amazon.com/cost-management/latest/userguide/cost-optimization-hub.html) — the assigned starting point; AWS's own consolidated view of cost-saving recommendations (rightsizing, RI/Savings Plan purchases, idle resource cleanup) across your entire account/organization in one place.

## Deepen your understanding

- **FinOps Foundation — FinOps Framework** (finops.org/framework) — the industry-standard vocabulary and lifecycle (Inform → Optimize → Operate) for cost management practices; useful for speaking the same language as a FinOps team in an interview or on the job.
- **AWS Well-Architected Framework — Cost Optimization Pillar** (docs.aws.amazon.com/wellarchitected) — AWS's own design principles for cost-aware architecture, goes beyond "buy an RI" into architectural decisions.
- **AWS Pricing Calculator** (calculator.aws) — model out the cost of a proposed architecture before building it; good habit to build for any design/interview whiteboard exercise.
- **"Cloud FinOps" by J.R. Storment & Mike Fuller (O'Reilly)** — the standard reference book for FinOps practice, covers allocation, optimization, and organizational operating models in depth.

## Reference / lookup

- **AWS Cost Explorer API Reference** (docs.aws.amazon.com/aws-cost-management/latest/APIReference) — for scripting cost reports (`get-cost-and-usage`, `get-rightsizing-recommendation`, `get-savings-plans-purchase-recommendation`).
- **Infracost documentation** (infracost.io/docs) — pre-`apply` Terraform cost estimation and PR-comment cost diffs.

## Practice

- **Infracost's public GitHub Action examples** — set up `infracost diff` as a check on a personal Terraform repo's PRs to see real cost-delta comments in action, the closest free hands-on equivalent to how a FinOps-mature org gates infrastructure changes.
