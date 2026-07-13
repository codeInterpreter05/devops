# Day 114 — Quiz: Cloud Cost Engineering

Try to answer without looking at your notes. Answers are at the bottom.

1. Why does Cost Explorer lag ~24 hours behind real usage, and what's the underlying data pipeline it's built on?
2. What's the difference between **blended** and **unblended** cost, and which one should you use for team-level chargeback in a multi-account AWS Organization?
3. Why do month-over-month cost trend charts look misleading right after a large upfront Reserved Instance purchase, and which Cost Explorer view fixes that?
4. What's the #1 gotcha with cost allocation tags that trips up teams who tag everything correctly in Terraform?
5. What's the structural difference between a Reserved Instance and a Savings Plan (what each one actually commits you to)?
6. Why have Compute Savings Plans mostly superseded Standard RIs for new commitments in modern, fast-changing infrastructure?
7. Should you size a Savings Plan/RI commitment to your peak usage or your sustained minimum usage? Why?
8. Why is a single-AZ, single-instance-type Spot fleet more likely to get interrupted than a diversified one?
9. What's the difference between S3 Lifecycle rules and S3 Intelligent-Tiering, and when would plain Lifecycle rules actually be the *cheaper* choice?
10. Is data transfer between two EC2 instances in different Availability Zones, same region, free? Explain.
11. Why can a NAT Gateway be an expensive default path for traffic to S3, and what's the standard fix?
12. **Interview question:** Our AWS bill is $50k/month and growing 30% monthly. Where do you start to reduce it?

---

## Answers

1. Cost Explorer is built on top of the Cost and Usage Report (CUR) billing pipeline, which itself is generated from usage records that AWS's billing systems process in batches — it is not a real-time metering system, so there's an inherent lag (commonly cited as up to 24 hours) between usage occurring and it appearing in Cost Explorer.
2. **Blended** cost averages Reserved Instance/Savings Plan discounts across all linked accounts under a payer account in an AWS Organization. **Unblended** cost shows what each account actually paid at its own effective rate. For chargeback, use **unblended** — blended cost can misattribute discounts to accounts that didn't pay for the underlying commitment.
3. An upfront-paid commitment shows as one large cost spike on the purchase date and near-zero for the covered usage afterward in raw billing data, which distorts trend analysis. The **amortized cost** view spreads the upfront payment evenly across the commitment term, giving an accurate month-over-month picture.
4. Cost allocation tags must be explicitly **activated** in Billing → Cost Allocation Tags before Cost Explorer/CUR can group by them, and activation is **not retroactive** — usage before the activation date will not be attributed to that tag even if the resource was tagged correctly the whole time.
5. A Reserved Instance commits to a **specific instance configuration** (family, region, optionally AZ). A Savings Plan commits to a **dollar-per-hour spend amount**, which then applies flexibly across whatever eligible usage you actually run.
6. Because Savings Plans (especially Compute Savings Plans) automatically apply to whatever instance family/size/OS/region — or even Fargate/Lambda — you're actually running, so migrating instance generations or shifting workload types doesn't strand the commitment the way it would with a Standard RI locked to one specific instance family.
7. Sustained **minimum** usage — commitments should cover your floor, with On-Demand or Spot absorbing variable usage above it. Sizing to peak means paying for idle commitment during every period you're below peak.
8. Spot capacity pools are independent per instance-type-per-AZ. A fleet pinned to one type in one AZ draws from a single pool, so any reclaim event in that pool interrupts the whole fleet at once. A fleet diversified across multiple instance types and AZs draws from many independent pools, so an interruption in one pool only affects a fraction of the fleet.
9. Lifecycle rules move objects to a target storage class after a fixed number of days, based on an access pattern **you** must already know. Intelligent-Tiering monitors actual per-object access and moves objects automatically, at the cost of a small per-object monitoring fee. Plain Lifecycle rules are cheaper when the access pattern is already known and uniform (e.g., logs that are never read after 7 days) — no reason to pay the monitoring fee for a pattern you can already hardcode.
10. No — cross-AZ transfer within the same region is metered, and billed on **both** the sending and receiving side (roughly $0.01/GB each way as of typical AWS pricing), even though it never leaves the region. This is one of the most commonly overlooked cost sources in "we're all in one region, transfer should be basically free" assumptions.
11. NAT Gateway charges a per-GB **data processing** fee on top of any underlying data transfer charge for everything routed through it. For high-volume traffic to AWS-native services like S3 or DynamoDB, a free **Gateway VPC Endpoint** routes that traffic privately, eliminating both the NAT processing fee and the need to traverse the NAT at all.
12. Strong answer: "First, get visibility — group Cost Explorer by service, then usage type, then tag/account, to find what's actually driving the 30% monthly growth (a real launch? a leak — unattached volumes, no S3 lifecycle policy, an autoscaling misconfiguration causing runaway instance counts? Or unmanaged Spot/On-Demand mix?). Second, check for quick, risk-free wins: unattached EBS volumes, idle Elastic IPs, missing S3 lifecycle/Intelligent-Tiering, oversized instances vs. actual CPU/memory utilization. Third, look at purchasing strategy — if there's a stable steady-state floor with no commitment coverage, that's usually the single biggest lever (up to ~70% off that floor with a Savings Plan), sized to sustained minimum, not peak. Fourth, check for architectural cost leaks like unnecessary cross-AZ chatter or NAT Gateway processing fees that a VPC endpoint would eliminate. Finally, put guardrails in place — budgets with alerts, tag enforcement via SCPs/Config rules — so growth is visible and attributable going forward instead of discovered after the fact next month." Then, if you have one, mention a specific real example you've done this for.
