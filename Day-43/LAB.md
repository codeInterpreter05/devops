# Day 43 — Lab: AWS Monitoring & Cost

**Goal:** Set up real billing/observability guardrails on your AWS account and practice reading your own bill like an engineer, not a spectator.

**Prerequisites:** An AWS account with billing access (root or an IAM user/role with `Billing` + `CloudWatch` + `ce:*` permissions), AWS CLI configured (`aws configure` or SSO), and — for Labs 3-4 — an existing EKS cluster (from earlier days) or any running EC2/ECS workload as a substitute if you haven't built EKS yet.

---

### Lab 1 — Billing alarm (the assigned hands-on activity, part 1)

Billing alerts are billed off the `AWS/Billing` namespace, which **only publishes metrics in `us-east-1`**, regardless of where your resources live — a very common trap.

1. Enable billing alerts once per account (Billing console → Billing Preferences → "Receive Billing Alerts"), or via CLI if using an Organization payer account feature.
2. Create an SNS topic to notify yourself:
   ```bash
   aws sns create-topic --name billing-alerts --region us-east-1
   aws sns subscribe --topic-arn <topic-arn> --protocol email --notification-endpoint you@example.com --region us-east-1
   ```
   Confirm the subscription from your inbox.
3. Create the alarm on estimated charges:
   ```bash
   aws cloudwatch put-metric-alarm \
     --alarm-name "billing-over-50-usd" \
     --namespace "AWS/Billing" \
     --metric-name EstimatedCharges \
     --dimensions Name=Currency,Value=USD \
     --statistic Maximum \
     --period 21600 \
     --evaluation-periods 1 \
     --threshold 50 \
     --comparison-operator GreaterThanThreshold \
     --alarm-actions <topic-arn> \
     --region us-east-1
   ```
4. Verify: `aws cloudwatch describe-alarms --alarm-names billing-over-50-usd --region us-east-1` and confirm state is `OK` or `INSUFFICIENT_DATA` (billing metrics can take hours to first populate).

**Success criteria:** An alarm exists in `us-east-1` on `AWS/Billing EstimatedCharges` with a working SNS email subscription, and you can explain why this alarm *must* live in `us-east-1` regardless of your primary working region.

---

### Lab 2 — CloudWatch dashboard for your EKS cluster (the assigned hands-on activity, part 2)

1. If not already installed, deploy CloudWatch Container Insights on your EKS cluster (quick-start via the observability add-on):
   ```bash
   aws eks update-cluster-config --name <cluster-name> --region <region> \
     --logging '{"clusterLogging":[{"types":["api","audit"],"enabled":true}]}'
   aws eks create-addon --cluster-name <cluster-name> --addon-name amazon-cloudwatch-observability --region <region>
   ```
2. Wait ~5 minutes for the CloudWatch agent + Fluent Bit DaemonSets to come up (`kubectl get pods -n amazon-cloudwatch`), then open CloudWatch console → Container Insights → Resources, and confirm cluster/node/pod metrics are populating.
3. Build a custom dashboard combining: cluster-level CPU/memory utilization, node count, and a Logs Insights widget showing recent `ERROR` lines from your application log group.
4. Export the dashboard as JSON (`aws cloudwatch get-dashboard --dashboard-name <name>`) and save it to your notes — this is how you'd version-control a dashboard as code (e.g., via Terraform's `aws_cloudwatch_dashboard` resource).

**Success criteria:** A saved CloudWatch dashboard showing live EKS cluster metrics plus at least one Logs Insights widget, and you have the dashboard definition exported as JSON.

---

### Lab 3 — Identify your top 3 cost drivers (the assigned hands-on activity, part 3)

1. Open Cost Explorer → set date range to "Last 3 months" → Group by "Service."
2. Note the top 3 services by cost. For each, drill in by grouping by "Usage Type" to see *what specifically* is driving the cost (e.g., is EC2 cost mostly compute-hours, or EBS storage, or data transfer?).
3. Check whether cost allocation tags are activated (Billing → Cost Allocation Tags). If any of your resources are tagged (e.g., `team`, `environment`), re-run the same report grouped by that tag instead of Service.
4. Run a CLI equivalent for scripting/automation purposes:
   ```bash
   aws ce get-cost-and-usage \
     --time-period Start=$(date -v-90d +%Y-%m-%d),End=$(date +%Y-%m-%d) \
     --granularity MONTHLY \
     --metrics "UnblendedCost" \
     --group-by Type=DIMENSION,Key=SERVICE
   ```
   (On Linux, replace `date -v-90d` with `date -d "90 days ago"`.)

**Success criteria:** You can name your account's top 3 cost drivers by service and explain, in one sentence each, why they're likely that high (e.g., "NAT Gateway data processing charges from cross-AZ traffic").

---

### Lab 4 — Infracost on a Terraform plan

1. Install Infracost: `brew install infracost` (or see infracost.io/docs for other platforms), then `infracost auth login` (free tier is sufficient).
2. In any Terraform directory you have (or a small test one with an `aws_instance` resource), run:
   ```bash
   infracost breakdown --path .
   ```
3. Change an instance type or add a resource, re-run, and observe the cost delta.

**Success criteria:** You've seen a monthly cost estimate generated directly from a Terraform plan, without anything being applied to AWS.

---

### Cleanup

```bash
aws cloudwatch delete-alarms --alarm-names billing-over-50-usd --region us-east-1
aws sns delete-topic --topic-arn <topic-arn> --region us-east-1
aws eks delete-addon --cluster-name <cluster-name> --addon-name amazon-cloudwatch-observability --region <region>
```

### Stretch challenge

Set up an **AWS Budget** with a Budget Action that automatically attaches a deny-new-EC2-launches IAM policy to a target IAM group/role once a $100 monthly forecasted spend threshold is crossed — then intentionally test it in a sandbox account (never in a shared/production account) to confirm the guardrail actually blocks a new `RunInstances` call.
