# Day 114 — Cheatsheet: Cloud Cost Engineering

## Cost Explorer CLI (`ce`)

```bash
# Total cost by service, last 6 months, monthly granularity
aws ce get-cost-and-usage \
  --time-period Start=2026-01-01,End=2026-07-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE

# Group by a cost allocation tag (must be activated first!)
aws ce get-cost-and-usage \
  --time-period Start=2026-06-01,End=2026-07-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=TAG,Key=Team

# Savings Plans purchase recommendation
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS

# Reserved Instance purchase recommendation
aws ce get-reservation-purchase-recommendation \
  --service "Amazon Elastic Compute Cloud - Compute"

# Rightsizing recommendations (find oversized instances)
aws ce get-rightsizing-recommendation --service "AmazonEC2"
```

## Cost allocation tags

```bash
# Tag a resource
aws ec2 create-tags --resources i-0123456789abcdef0 --tags Key=Team,Value=platform

# Activate a user-defined cost allocation tag (billing-side, console or CLI)
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Team,Status=Active

# List which tags are activated
aws ce list-cost-allocation-tags --status Active
```

## Finding waste

```bash
# Unattached EBS volumes (pure waste — no instance using them)
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType}'

# Idle Elastic IPs (charged when NOT attached to a running instance)
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null]'

# Old, unused EBS snapshots
aws ec2 describe-snapshots --owner-ids self \
  --query 'Snapshots[*].{ID:SnapshotId,Start:StartTime,Size:VolumeSize}'

# S3 buckets with no lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket my-bucket   # errors if none set
```

## S3 storage class / lifecycle

```bash
# Set default Intelligent-Tiering for all new objects
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket --lifecycle-configuration '{
  "Rules": [{
    "ID": "auto-it",
    "Filter": {"Prefix": ""},
    "Status": "Enabled",
    "Transitions": [{"Days": 0, "StorageClass": "INTELLIGENT_TIERING"}]
  }]
}'

# Transition to Glacier Deep Archive after 180 days, expire after 2 years
aws s3api put-bucket-lifecycle-configuration --bucket my-archive --lifecycle-configuration '{
  "Rules": [{
    "ID": "archive-then-expire",
    "Filter": {"Prefix": "logs/"},
    "Status": "Enabled",
    "Transitions": [{"Days": 180, "StorageClass": "DEEP_ARCHIVE"}],
    "Expiration": {"Days": 730}
  }]
}'

# Check current storage class distribution (via S3 Storage Lens, console-only summary)
```

## Savings Plans / RI purchasing decision cheat-sheet

```
Predictable, stable steady-state spend      -> Compute Savings Plan (most flexible)
Need a specific instance family guaranteed   -> EC2 Instance Savings Plan or Standard RI
Need guaranteed capacity availability        -> Capacity Reservation (+ optional SP for discount)
Stateless / interruption-tolerant workload   -> Spot (diversify instance types + AZs)
Size any commitment to: sustained MINIMUM usage, never peak
```

## Data transfer cost mental model

```
Internet  -> AWS                : free (inbound)
AWS       -> Internet             : metered, tiered rate, decreases at volume
Same AZ   <-> Same AZ             : free
Cross-AZ, same region             : charged BOTH directions (~$0.01/GB each way)
Cross-region                      : charged BOTH directions, higher rate
NAT Gateway                       : + per-GB PROCESSING fee on top of transfer cost
S3/DynamoDB Gateway VPC Endpoint  : free — replaces NAT for traffic to those services
```

## Budgets & alerting

```bash
aws budgets create-budget --account-id 123456789012 --budget '{
  "BudgetName": "monthly-total",
  "BudgetLimit": {"Amount": "5000", "Unit": "USD"},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}' --notifications-with-subscribers '[{
  "Notification": {"NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN", "Threshold": 80},
  "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "you@example.com"}]
}]'
```

## Infracost (pre-apply cost estimation)

```bash
infracost auth login                        # get a free API key
infracost breakdown --path .                # estimate cost of a Terraform plan/dir
infracost diff --path . --compare-to base.json   # cost delta vs a baseline, PR-friendly
```
