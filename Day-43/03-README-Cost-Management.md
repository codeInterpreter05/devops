# Day 43 — AWS Monitoring & Cost: Cost Explorer, Budgets & Savings Plans vs. RIs

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** –

## Brief

FinOps is no longer a separate team's problem — modern DevOps roles are expected to own cost visibility and optimization for the infrastructure they run, because engineers are the only people who actually understand *why* a resource exists and whether it can be rightsized or shut off. This is also a near-guaranteed interview topic in any org running a shared multi-team AWS account: "how do you allocate cost to teams?" is asked because it reveals whether you've actually operated a real bill, not just spun up free-tier resources.

## Cost Explorer

Cost Explorer visualizes and breaks down AWS spend. The two things that make it *useful* rather than just a chart:

- **Grouping/filtering dimensions**: by Service, Linked Account, Region, Usage Type, and — critically — by **Tag**. Without tags, you can see "EC2 cost $40k" but not "Team A's EC2 cost $12k, Team B's $28k."
- **Forecasting**: Cost Explorer projects future spend based on historical trend — useful for catching a runaway growth curve before it becomes a shocking bill.
- **RI/Savings Plans utilization & coverage reports**: shows what percentage of your eligible usage is actually covered by a commitment discount, and how much of what you bought is being used — the two numbers you need to know if you're overcommitted (buying more than you use) or undercommitted (paying on-demand rates you could have discounted).

**Cost allocation tags** come in two flavors:
- **AWS-generated tags** (e.g., `aws:createdBy`) — automatic, but limited.
- **User-defined tags** — you must explicitly **activate** them in the Billing console before they show up in Cost Explorer, even if the resources already have the tag applied. This is a very common "why isn't my tag showing up in cost reports" gotcha.

For anything beyond Cost Explorer's built-in views, the **Cost and Usage Report (CUR)** — a detailed, line-item CSV/Parquet export to S3 — combined with **Athena** (query) and **QuickSight** (dashboard) is the standard heavy-duty setup for chargeback/showback reporting across many teams.

**Answering "how do you allocate cost to teams in a shared account"** (this day's interview question) in practice means:
1. Enforce a **tagging policy** (via AWS Organizations Tag Policies or a Config rule that flags untagged resources) so every resource carries a `team`/`cost-center` tag from creation.
2. Activate those tags as cost allocation tags.
3. Use Cost Explorer (quick view) or CUR + Athena (detailed, automatable showback/chargeback reports) grouped by that tag.
4. For resources that can't be tagged cleanly (e.g., a shared EKS cluster's underlying EC2 nodes serving multiple teams' pods) — split cost using **Kubecost** or AWS's own **split cost allocation for shared EKS/ECS resources** feature, which apportions node cost across namespaces by resource requests/usage.

## AWS Budgets

Budgets are *forward-looking, threshold-based* — they don't visualize, they **alert (and optionally act)** when spend crosses a line.

- **Cost budgets** — alert when actual or forecasted spend exceeds a dollar threshold.
- **Usage budgets** — alert on usage quantity (e.g., EC2 instance-hours) rather than dollars.
- **RI/Savings Plans utilization & coverage budgets** — alert if you drop below a target utilization (wasting money on unused commitment) or coverage (missing out on available discount) percentage.
- **Budget Actions** — the part people often don't know exists: a budget breach can automatically **attach an IAM/SCP policy** (e.g., deny launching new EC2 instances) or **stop/target an Auto Scaling group**, turning a passive alert into an automated cost guardrail.

## Savings Plans vs. Reserved Instances

Both are **commitment-based discount mechanisms** — you commit to spend/usage for 1 or 3 years in exchange for a discount versus on-demand pricing. Neither is a *capacity reservation* by default (that's a separate, explicit feature — On-Demand Capacity Reservations — which you can optionally combine with a Savings Plan/RI for the discount).

**Reserved Instances (RIs):**
- Tied to a specific **instance family + region** (Standard RI) — inflexible; if you resize or change instance family, the RI doesn't apply unless it's a **Convertible RI** (allows exchanging for a different instance family/OS, at a slightly lower discount than Standard).
- Can be **sold on the RI Marketplace** if no longer needed (Standard RIs only).
- Payment options: All Upfront (biggest discount), Partial Upfront, No Upfront.
- Applies automatically to matching running instances — no action needed once purchased, AWS's billing system matches it.

**Savings Plans:**
- **Compute Savings Plans** — most flexible: automatically applies to any EC2 instance usage regardless of family, size, OS, tenancy, or **even region**, and also covers Fargate and Lambda. Committed as an hourly `$` spend rate, not tied to a specific resource.
- **EC2 Instance Savings Plans** — narrower (locked to an instance family within a region), but a bigger discount than Compute Savings Plans in exchange for that reduced flexibility — conceptually similar to a Standard RI but simpler to manage (no marketplace, no per-resource matching quirks).

**Decision framework:**
- Stable, predictable workload on a fixed instance family for 1-3 years, want the max discount → **EC2 Instance Savings Plan** or Convertible RI.
- Workload composition changes over time (mix of instance types, moving to Graviton, adopting Fargate) → **Compute Savings Plan** — you get a discount without betting on today's instance mix being tomorrow's.
- Need to walk back a bad purchase → only Standard RIs are resellable on the marketplace; Savings Plans commitments cannot be canceled or sold.

**Infracost** (listed as a tool for today) plugs into this at a different point in the lifecycle: it estimates the **monthly cost delta of a Terraform plan** and posts it as a PR comment *before* anything is applied — shifting cost visibility left into code review, rather than discovering the cost impact on next month's bill. This is the FinOps equivalent of a linter: catch the expensive mistake before merge, not after the invoice.

## Points to Remember

- User-defined tags must be explicitly activated as cost allocation tags in Billing settings — tagging the resource alone isn't enough for it to show up in Cost Explorer.
- RIs/Savings Plans are billing discounts, not capacity guarantees — pair with an explicit Capacity Reservation if you need guaranteed capacity, not just a lower price.
- Compute Savings Plans trade a slightly smaller discount for flexibility across instance family/region/service (EC2, Fargate, Lambda); EC2 Instance Savings Plans and Standard RIs trade flexibility for a bigger discount.
- Only Standard RIs can be resold on the RI Marketplace — Savings Plans commitments are locked in for the term.
- Budget Actions can automatically enforce a guardrail (deny new launches, stop an ASG) — budgets aren't just email alerts.
- Infracost gives cost visibility at PR time, before infrastructure is even created — shifting FinOps left instead of discovering costs after the bill arrives.

## Common Mistakes

- Buying Standard RIs for a workload that's actively being resized/migrated (e.g., mid-Graviton-migration), then discovering the RI doesn't apply anymore and money is wasted — Convertible RIs or Compute Savings Plans exist for exactly this situation.
- Forgetting to activate cost allocation tags, then concluding "our tagging strategy doesn't work" when actually the tags were never turned on in Billing preferences.
- Treating an EC2 Instance Savings Plan/RI purchase as a capacity guarantee during a big launch, then being surprised by a capacity error — the discount and the guarantee are two different AWS features.
- Not monitoring RI/Savings Plan utilization — buying a large commitment "to save money" and then running below-capacity, effectively paying for unused reserved discount on top of on-demand for whatever exceeds it.
- Building a whole tagging/chargeback pipeline in a spreadsheet by hand instead of using CUR + Athena/QuickSight, which scales and updates automatically.
