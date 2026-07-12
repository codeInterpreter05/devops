# Day 44 — Cheatsheet: AWS Reliability & DR

## RTO / RPO / DR spectrum

```
RTO = time to recover (downtime)
RPO = data lost (measured in time)

DR spectrum (cost/complexity increases left to right):
Backup & Restore -> Pilot Light -> Warm Standby -> Multi-Site Active/Active
(hours RTO)         (tens of min)   (minutes)       (seconds)
```

## Route 53 health checks & failover

```bash
aws route53 create-health-check --caller-reference $(uuidgen) \
  --health-check-config Type=HTTPS,ResourcePath=/health,FullyQualifiedDomainName=app.example.com,Port=443,RequestInterval=10,FailureThreshold=3

aws route53 change-resource-record-sets --hosted-zone-id Z123 --change-batch file://failover-primary.json
```

```json
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "primary",
      "Failover": "PRIMARY",
      "TTL": 60,
      "ResourceRecords": [{"Value": "1.2.3.4"}],
      "HealthCheckId": "abcd-1234"
    }
  }]
}
```

Failover time ≈ (health check interval × failure threshold) + TTL. Fast check: 10s interval, threshold 3 = ~30s detection.

## S3 Cross-Region Replication

```bash
aws s3api put-bucket-versioning --bucket src-bucket --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket dst-bucket --versioning-configuration Status=Enabled

aws s3api put-bucket-replication --bucket src-bucket --replication-configuration file://repl.json

# Backfill existing objects (CRR only replicates new writes going forward)
aws s3control create-job --account-id 123456789012 --operation '{"S3ReplicateObject":{}}' \
  --manifest file://manifest.json --priority 1 --role-arn arn:aws:iam::...:role/batch-repl
```

Replication Time Control (RTC): add `"Metrics":{"Status":"Enabled"}` and `"ReplicationTime":{"Status":"Enabled","Time":{"Minutes":15}}` to the destination config — SLA-backed 15-min bound.

## RDS backups & PITR

```bash
aws rds describe-db-instances --db-instance-identifier prod-db \
  --query 'DBInstances[0].[BackupRetentionPeriod,LatestRestorableTime]'

aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db \
  --target-db-instance-identifier prod-db-restored \
  --restore-time 2026-07-10T14:32:00Z

aws rds create-db-snapshot --db-instance-identifier prod-db --db-snapshot-identifier pre-migration-snap

aws rds modify-db-instance --db-instance-identifier prod-db \
  --backup-retention-period 7 --apply-immediately
```

## AWS Fault Injection Simulator (FIS)

```bash
aws fis create-experiment-template --cli-input-json file://template.json
aws fis start-experiment --experiment-template-id EXT123
aws fis list-experiments
aws fis get-experiment --id EXP123
aws fis stop-experiment --id EXP123
```

Common FIS actions:
```
aws:ec2:stop-instances
aws:ec2:terminate-instances
aws:ecs:stop-task
aws:eks:pod-delete / aws:eks:pod-cpu-stress
aws:ssm:send-command  (network latency, packet loss, disk stress via SSM agent)
aws:rds:reboot-db-instances
aws:rds:failover-db-cluster
```

Always set a `stopCondition` referencing a CloudWatch alarm ARN — automatic abort if things go wrong.

## Multi-AZ vs Multi-Region quick decision

```
Need protection from: single AZ/hardware failure -> Multi-AZ (cheap, default)
Need protection from: full regional outage / compliance residency -> Multi-Region
Need low latency for global users -> Multi-Region (performance reason, not reliability)
```
