# Day 43 — Quiz: AWS Monitoring & Cost

Try to answer without looking at your notes. Answers are at the bottom.

1. Why might a CloudWatch query over 6-month-old data return hourly-resolution points even though you asked for 1-minute granularity?
2. What does "3 out of 3 evaluation periods" mean for a CloudWatch alarm, and why does this exist?
3. What does the `Treat Missing Data` alarm setting control, and which setting would you use for a service heartbeat metric — and why is the default wrong for that case?
4. What's the difference between a CloudWatch dashboard and a CloudWatch alarm, in terms of what each is actually for?
5. Why does CloudWatch Logs retention default to "Never expire," and why is that a cost risk?
6. What's the difference in *purpose* between AWS Config and Trusted Advisor?
7. What support plan tier is required to see the full set of Trusted Advisor checks?
8. Why won't a resource's tags show up in Cost Explorer even if the resource is correctly tagged?
9. Are Reserved Instances or Savings Plans a capacity guarantee? If not, what AWS feature actually guarantees capacity?
10. What's the practical tradeoff between a Compute Savings Plan and an EC2 Instance Savings Plan?
11. What does Infracost do, and at what point in the workflow does it add value compared to Cost Explorer?
12. **Interview question:** How do you allocate AWS costs to individual teams in a shared account?

---

## Answers

1. Because CloudWatch's retention/resolution tiers degrade over time: full 1-minute resolution is only kept for 15 days, then data is aggregated to 5-minute (up to 63 days) and eventually 1-hour (up to 455 days). A query on old data silently returns the coarser available granularity rather than erroring.
2. It means the alarm condition (e.g., CPU > 80%) must hold true continuously across 3 consecutive measurement periods before the alarm transitions to `ALARM` state. This exists to debounce transient spikes — a single 1-minute blip shouldn't page anyone.
3. It controls how the alarm evaluator treats missing data points. For a heartbeat metric, you want `breaching` — if the metric stops arriving, that itself means the service is likely dead, and the alarm should fire. The default (`missing`) has no effect on alarm state, so a fully crashed process that stops emitting its heartbeat would leave the alarm silently stuck at `INSUFFICIENT_DATA` forever, never paging anyone.
4. A dashboard is a visualization for humans to look at — it does not notify anyone on its own. An alarm is an automated state machine that evaluates a metric and can trigger actions (SNS notification, Auto Scaling, EC2 action) without a human watching. Relying on a dashboard as your alerting mechanism means nothing happens unless someone is actively staring at the screen.
5. Because AWS's default behavior is to keep logs indefinitely unless you set an explicit retention policy. Since most log groups accumulate continuously, "never expire" left unset across many log groups silently grows storage cost over months — a very common invisible AWS cost leak.
6. AWS Config continuously tracks configuration changes and evaluates them against rules **you define** (managed or custom), with full change history and optional auto-remediation. Trusted Advisor runs periodic checks against **AWS-curated generic best practices** across cost, security, performance, and fault tolerance, with no auto-remediation — it's a recommendation engine, not an enforcement engine.
7. Business or Enterprise Support. On Basic/Developer support you only get a limited subset of checks (mostly Security and Service Limits) — most Cost Optimization, Performance, and Fault Tolerance checks are gated behind the higher tiers.
8. Because cost allocation tags must be explicitly **activated** in the Billing console (Cost Allocation Tags page) before Cost Explorer will use them for grouping — simply tagging the resource in EC2/other consoles isn't sufficient on its own.
9. No — RIs and Savings Plans are billing discount mechanisms, not capacity guarantees; AWS can still fail to have capacity available for your instance type even if you've reserved/committed to it, unless you separately purchase an **On-Demand Capacity Reservation**, which can optionally be combined with a Savings Plan/RI to get the discount plus the guarantee.
10. A Compute Savings Plan applies flexibly across instance family, size, OS, region, and even service (EC2, Fargate, Lambda) at a somewhat lower discount rate. An EC2 Instance Savings Plan is locked to a specific instance family within a region but gives a bigger discount in exchange for that reduced flexibility — pick based on how stable your compute footprint is expected to be over the commitment term.
11. Infracost estimates the dollar cost impact of a Terraform plan/diff and can post it as a comment on a pull request before anything is applied. Unlike Cost Explorer (which reports on cost that has *already* been incurred), Infracost gives cost visibility at code-review time — shifting FinOps left so expensive mistakes are caught before merge, not after the next bill arrives.
12. Strong answer: "Enforce a tagging policy (via Organizations Tag Policies or a Config rule flagging untagged resources) so every resource is created with a `team`/`cost-center` tag. Activate those as cost allocation tags in Billing, then use Cost Explorer for quick visibility or the Cost and Usage Report loaded into Athena/QuickSight for detailed, automatable chargeback reports grouped by that tag. For genuinely shared resources that can't be tagged to one team — like the EC2 nodes under a shared EKS cluster — use split cost allocation (EKS/ECS) or a tool like Kubecost to apportion cost by namespace based on actual resource requests/usage." Mention Budgets with per-team thresholds as a complementary proactive guardrail if you have a real example.
