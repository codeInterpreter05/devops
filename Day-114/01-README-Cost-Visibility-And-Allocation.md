# Day 114 — Cloud Cost Engineering: Cost Visibility & Allocation

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** FinOps | **Flag:** ⚡ Interview-critical

## Brief

FinOps starts with visibility — you cannot optimize what you cannot attribute. Most orgs' first cost crisis isn't "the bill is high," it's "nobody can tell which team, service, or feature is causing it." This note covers AWS Cost Explorer (the primary lens into *where* money goes) and cost allocation tagging (the mechanism that makes "where" answerable at all). Interviewers probe this constantly because it separates people who've only ever *looked* at a bill from people who've actually had to explain one to a VP of Engineering.

This day is split into three focused files:

1. **This file** — Cost Explorer mechanics and cost allocation tag strategy.
2. **[02-README-Compute-Purchasing-Options.md](02-README-Compute-Purchasing-Options.md)** — Savings Plans vs. Reserved Instances vs. Spot.
3. **[03-README-Storage-And-Data-Transfer.md](03-README-Storage-And-Data-Transfer.md)** — S3 Intelligent-Tiering and the hidden data transfer bill.

## AWS Cost Explorer: how it actually works

Cost Explorer isn't real-time — it's built on top of the **Cost and Usage Report (CUR)** pipeline, and data typically lags 8-24 hours. Understanding its data model matters more than clicking through the UI:

- **Granularity**: daily or monthly. Hourly granularity exists but only for the last 14 days and costs extra to enable (`GetCostAndUsage` API with `Granularity=HOURLY`).
- **Dimensions vs. Tags**: you can group/filter by built-in **dimensions** (Service, Linked Account, Region, Usage Type, Instance Type, Purchase Option) or by **cost allocation tags** you've defined yourself. Dimensions are always available; tags are only queryable *after* you activate them (see below) — and only for usage *after* activation, never retroactively.
- **Unblended vs. Blended costs**: in a multi-account AWS Organization, **blended cost** averages Reserved Instance/Savings Plan discounts across all linked accounts sharing the payer account, while **unblended cost** shows what each account actually paid at its own effective rate. For chargeback/showback to individual teams, you almost always want **unblended** — blended cost can make a team that bought no commitments look like it's getting a discount it didn't pay for, and vice versa.
- **Amortized costs**: an upfront-paid Reserved Instance or Savings Plan looks like a huge one-time spike in raw billing data on the purchase date and $0 for the rest of the term. Cost Explorer's "amortized" cost view spreads that upfront payment evenly across the commitment term so trend charts aren't misleading. Always use amortized view when analyzing month-over-month trends if your org uses upfront commitments.

### The real workflow for a cost investigation

```text
1. Cost Explorer -> Group by: Service -> spot the top 3-5 services by spend
2. Drill into the worst offender -> Group by: Usage Type -> find the specific SKU
   (e.g. "BoxUsage:m5.4xlarge" vs "DataTransfer-Out-Bytes")
3. Group by: Linked Account / Tag:team -> find WHO is generating it
4. Filter by date range around any spend spike -> correlate with a deploy/launch
5. Cross-reference with CloudTrail if the spike is unexplained (someone spun up
   something manually outside IaC)
```

Interview signal: candidates who say "I'd open Cost Explorer and look at the graph" sound junior. Candidates who say "I'd group by service, then usage type, then tag, and cross-reference the spike date with deploy history" sound like they've actually done it.

## Cost allocation tags: the strategy, not just the mechanic

Tagging is the single highest-leverage FinOps practice, and it's also the one that rots fastest if not enforced.

**Two tag types in AWS:**
- **User-defined tags**: applied by you/your IaC (`Team: platform`, `Environment: prod`, `CostCenter: 4410`).
- **AWS-generated tags**: `aws:createdBy`, automatically populated — useful for finding *who* (IAM principal) created a resource when a tag is missing.

**A tag must be *activated* in the Billing console** (Billing → Cost Allocation Tags) before Cost Explorer or CUR will let you group by it. This is the #1 gotcha: teams tag everything correctly in Terraform, then wonder why Cost Explorer shows nothing when grouped by that tag — because nobody activated it, and even after activating, it only applies going forward, not retroactively.

**A minimal tagging schema that actually survives contact with reality:**

| Tag key | Purpose | Example values |
|---|---|---|
| `Environment` | prod isolation for cost + blast radius | `prod`, `staging`, `dev` |
| `Team` / `Owner` | chargeback target | `platform`, `search`, `payments` |
| `CostCenter` | maps to finance's GL codes | `4410` |
| `Service` | which microservice/app | `checkout-api` |
| `ManagedBy` | prevents accidental manual deletion of IaC-owned resources | `terraform`, `manual` |

**Enforcement, not just convention:**
- **AWS Config rules** (`required-tags` managed rule) or **Service Control Policies (SCPs)** that deny `RunInstances`/`CreateBucket` etc. without required tags present — enforcement at the API call, not after the fact.
- **Terraform**: enforce via a wrapper module or a `default_tags` block at the provider level so every resource inherits baseline tags without every engineer remembering to add them by hand:
  ```hcl
  provider "aws" {
    default_tags {
      tags = {
        ManagedBy = "terraform"
        Team      = var.team
      }
    }
  }
  ```
- **Tag Policies** (AWS Organizations) can enforce allowed *values* for a tag key across the whole org (e.g., `Environment` must be one of `prod|staging|dev`, rejecting typos like `Prod ` or `production`).

Untagged/mistagged resources don't just create a reporting gap — they get bucketed into an "Untagged" or "No tag key" line in Cost Explorer, which in a real org routinely represents 15-30% of spend if enforcement is weak. That bucket is unattributable by definition, which means during a cost review it becomes "everyone's problem," which in practice means nobody owns it.

## Points to Remember

- Cost Explorer is CUR-backed and lags ~24h — it is an analysis tool, not a real-time dashboard; for real-time, you'd stream CUR to Athena/QuickSight or use budgets/alerts instead.
- **Unblended** cost for chargeback (what each account actually paid); **amortized** view for trend analysis when upfront RIs/Savings Plans are in play; **blended** mostly matters at the payer-account/consolidated-billing level.
- Tags must be explicitly activated in Billing → Cost Allocation Tags before they're queryable, and only apply from the activation date forward — never retroactively.
- Enforce tags at creation time (SCPs, AWS Config `required-tags`, Terraform `default_tags`) — a tagging *convention* without enforcement decays within a quarter.
- A large "Untagged" bucket in Cost Explorer is a governance failure, not a reporting nuance — it means real spend has no owner.

## Common Mistakes

- Looking at **blended** costs in a multi-account org and drawing chargeback conclusions from them — this silently redistributes RI/Savings Plan discounts across accounts that didn't buy the commitment.
- Tagging resources correctly in Terraform, then being confused when Cost Explorer shows no data for that tag — because the tag was never activated in the Billing console, or was activated after the usage already occurred.
- Treating tagging as a "nice to have" cleanup task instead of an enforced gate — leads to a permanently growing "Untagged" bucket that nobody is incentivized to fix because fixing it doesn't reduce the bill, it just reveals who's responsible for it.
- Comparing month-over-month cost trends using raw (non-amortized) billing data right after a large upfront Reserved Instance/Savings Plan purchase, and panicking at the apparent spike (or later, the apparent "free" months).
- Not knowing about `aws:createdBy` — spending hours trying to find who launched an untagged resource manually when AWS already recorded the creating principal.
