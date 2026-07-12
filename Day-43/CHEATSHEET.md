# Day 43 — Cheatsheet: AWS Monitoring & Cost

## CloudWatch metrics

```bash
# Put a custom metric
aws cloudwatch put-metric-data --namespace "MyApp/Orders" \
  --metric-name OrdersProcessed --value 42 --unit Count

# List metrics for a namespace
aws cloudwatch list-metrics --namespace AWS/EC2

# Get datapoints
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --start-time $(date -u -v-1H +%FT%TZ) --end-time $(date -u +%FT%TZ) \
  --period 300 --statistics Average Maximum
```

## Retention / resolution reference

```
< 3h    -> 1-second data kept   (high-resolution custom metrics only)
< 15d   -> 1-minute data kept
< 63d   -> 5-minute data kept
< 455d  -> 1-hour data kept
```

## Alarms

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123 \
  --statistic Average --period 300 --evaluation-periods 3 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --treat-missing-data breaching \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:alerts

aws cloudwatch describe-alarms --alarm-names high-cpu
aws cloudwatch set-alarm-state --alarm-name high-cpu --state-value ALARM --state-reason "manual test"
```

`--treat-missing-data`: `missing` (default, no effect) | `notBreaching` | `breaching` | `ignore`

## Logs Insights query syntax

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20

stats count(*) by bin(5m)

fields @message
| parse @message "status=* latency=*" as status, latency
| stats avg(latency), pct(latency,99) as p99 by status
```

## Logs

```bash
aws logs describe-log-groups
aws logs put-retention-policy --log-group-name /aws/eks/my-cluster/app --retention-in-days 30
aws logs put-subscription-filter --log-group-name /aws/lambda/fn \
  --filter-name to-lambda --filter-pattern "ERROR" \
  --destination-arn arn:aws:lambda:...:function:log-processor
```

## AWS Config

```bash
aws configservice describe-configuration-recorders
aws configservice describe-compliance-by-config-rule
aws configservice put-config-rule --config-rule file://rule.json
aws configservice get-compliance-details-by-config-rule --config-rule-name restricted-ssh
```

## Trusted Advisor

```bash
aws support describe-trusted-advisor-checks --language en          # requires Business/Enterprise support
aws support describe-trusted-advisor-check-result --check-id <id>
```

## Cost Explorer / Budgets

```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-04-01,End=2026-07-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

aws ce get-reservation-utilization --time-period Start=2026-06-01,End=2026-07-01
aws ce get-savings-plans-utilization --time-period Start=2026-06-01,End=2026-07-01

aws budgets create-budget --account-id 123456789012 --budget file://budget.json
```

## Savings Plans vs. Reserved Instances — decision cheat

```
Stable instance family, want max discount, 1-3yr commit -> EC2 Instance Savings Plan / Convertible RI
Mixed/changing instance types, multi-service (EC2+Fargate+Lambda) -> Compute Savings Plan
Need to exit early / resell -> Standard RI only (RI Marketplace)
Need guaranteed capacity, not just discount -> On-Demand Capacity Reservation (separate from discount)
```

## Infracost

```bash
infracost auth login
infracost breakdown --path .
infracost diff --path . --compare-to baseline.json
```
