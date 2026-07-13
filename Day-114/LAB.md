# Day 114 — Lab: Cloud Cost Engineering

**Goal:** Analyze a real (or realistic sandbox) AWS account's spend, find the top 3 savings opportunities, and calculate the ROI of a Reserved Instance / Savings Plan purchase — the actual assigned hands-on activity for today, broken into concrete steps.

**Prerequisites:** An AWS account with at least a few weeks of billing history (a personal/free-tier account works, though findings will be small-scale; if you have access to a work sandbox account with real workloads, use that instead). AWS CLI configured (`aws configure`). IAM permissions for `ce:*` (Cost Explorer) and `budgets:*`. Optional: [Infracost](https://www.infracost.io/) installed (`brew install infracost` or see their docs) for Lab 4.

---

### Lab 1 — Turn on Cost Explorer and get oriented

1. In the AWS Console, go to **Billing and Cost Management → Cost Explorer**. If it's your first time, enabling it can take up to 24 hours to backfill data — do this step first, before anything else, so data is ready by the time you reach later labs.
2. Set the date range to the last 3 months, **Granularity: Monthly**, **Group by: Service**.
3. Identify your top 3 services by cost. Switch **Group by** to **Usage Type** for the #1 service and identify the specific SKU driving spend.
4. Switch the cost metric toggle to **Amortized costs** and compare the chart shape to **Unblended costs** — note any months where they diverge (a sign of an upfront RI/Savings Plan purchase).

**Success criteria:** You can name your account's top 3 cost-driving services and the specific usage type within the #1 service, and you can explain in one sentence why amortized and unblended views diverged (or state that they didn't, and why that's also meaningful — no big upfront commitments yet).

---

### Lab 2 — Activate and use cost allocation tags

1. Go to **Billing → Cost Allocation Tags**. Activate at least one user-defined tag (e.g., `Environment` or `Team`) if you have tagged resources; if you have none tagged yet, tag 2-3 EC2/S3 resources first via the console or CLI:
   ```bash
   aws ec2 create-tags --resources i-0123456789abcdef0 --tags Key=Team,Value=platform
   ```
2. Wait ~24 hours (tag activation isn't instant for cost data), then return to Cost Explorer and set **Group by: Tag → Team**.
3. Note that any usage from *before* activation shows as untagged, even though the resource now has the tag — confirm this in your own account and write one sentence explaining why (activation is forward-looking, not retroactive).

**Success criteria:** You've activated a real tag, grouped a Cost Explorer report by it, and can explain the forward-only activation gotcha from firsthand observation, not just from reading about it.

---

### Lab 3 — The core hands-on activity: find 3 savings opportunities + RI/Savings Plan ROI

This is the assigned activity — do it against your real usage data.

1. Go to **Billing → Savings Plans → Recommendations** (or **Reserved Instances → Recommendations**). Note the recommended commitment size, based on your trailing usage.
2. For your top instance family by On-Demand spend (from Lab 1), calculate the ROI of a 1-year, no-upfront Compute Savings Plan by hand:
   ```text
   Current monthly On-Demand cost for that usage:      $X
   Savings Plan discount rate (Console shows this):     Y%
   Projected monthly cost under the Savings Plan:       $X * (1 - Y)
   Monthly savings:                                     $X - ($X * (1-Y))
   Annual savings:                                       Monthly savings * 12
   Break-even point (no-upfront plans have no break-even risk — 1-year commitment,
   but note the "stranded commitment" risk if usage drops below the committed $/hr)
   ```
3. Identify 2 more savings opportunities that are NOT purchasing commitments — e.g.:
   - An S3 bucket with no lifecycle policy and objects older than 90 days (`aws s3api list-objects-v2 --bucket <name> --query 'Contents[?LastModified<=`2026-04-01`]'`)
   - An EBS volume unattached to any running instance (`aws ec2 describe-volumes --filters Name=status,Values=available`)
   - An idle/oversized RDS or EC2 instance (compare CloudWatch `CPUUtilization` average over 2 weeks against instance size)
4. Write a one-paragraph summary as if presenting to a manager: the 3 opportunities, estimated $/month impact of each, and which one you'd do first and why (usually: whichever has zero risk and highest $ impact — deleting unattached EBS volumes almost always wins that ranking).

**Success criteria:** You have 3 real, numbered savings opportunities with a dollar estimate for each, and a hand-calculated ROI for at least one commitment-based option.

---

### Lab 4 — Estimate cost *before* it hits the bill with Infracost

1. Install Infracost and get a free API key (`infracost auth login`).
2. In any Terraform directory you have (or a small sample `main.tf` provisioning an EC2 instance + EBS volume), run:
   ```bash
   infracost breakdown --path .
   ```
3. Note that this estimates cost **before `terraform apply`** — compare this workflow to Cost Explorer, which only tells you about cost *after* the fact.
4. Add `infracost diff --path . --compare-to infracost-base.json` to your mental model of a PR-time cost gate (this is literally what Infracost's GitHub Action does — comments estimated cost delta on every PR).

**Success criteria:** You can explain the difference between *reactive* cost visibility (Cost Explorer, after spend happens) and *proactive* cost visibility (Infracost, before `apply`), and why a mature FinOps practice needs both.

---

### Cleanup

```bash
# Remove any test tags/resources you created purely for this lab
aws ec2 delete-tags --resources i-0123456789abcdef0 --tags Key=Team,Value=platform
# Do NOT delete real production resources you tagged for genuine chargeback purposes.
# If you created a sandbox EC2 instance purely for this lab, terminate it:
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0
```

### Stretch challenge

Set up an **AWS Budget** (Billing → Budgets) with a monthly cost threshold and an SNS-notified alert at 80% and 100% of budget. Then simulate the alert path: subscribe your email to the SNS topic and confirm you actually receive the notification. This is the proactive-alerting half of cost visibility that Cost Explorer alone doesn't give you — Cost Explorer requires you to go look; Budgets come to you.
