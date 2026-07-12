# Day 35 — Cheatsheet: AWS Storage & Databases

## S3 versioning & lifecycle

```bash
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Enabled
aws s3api list-object-versions --bucket my-bucket
aws s3api delete-object --bucket my-bucket --key file.txt --version-id <DELETE_MARKER_ID>   # undelete
```
```json
{
  "Rules": [{
    "ID": "archive-old", "Status": "Enabled", "Filter": {"Prefix": "logs/"},
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER"}
    ],
    "Expiration": {"Days": 365},
    "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
  }]
}
```
```bash
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket --lifecycle-configuration file://lifecycle.json
```

## S3 replication (CRR/SRR)

```bash
aws s3api put-bucket-replication --bucket my-bucket --replication-configuration file://replication.json
```
Requires versioning on BOTH source and destination. Not retroactive (use S3 Batch Replication to backfill). Async — eventual, not instant.

## S3 access points

```bash
aws s3control create-access-point --account-id ACCOUNT --name finance-ap --bucket my-shared-bucket
```

## RDS Multi-AZ

```bash
aws rds create-db-instance --db-instance-identifier my-db --multi-az \
  --engine postgres --db-instance-class db.r5.large --allocated-storage 100 \
  --master-username admin --master-user-password ****
aws rds reboot-db-instance --db-instance-identifier my-db --force-failover   # test failover
```
Synchronous standby, unreadable, automatic failover (~60-120s), zero data loss.

## RDS Read Replicas

```bash
aws rds create-db-instance-read-replica --db-instance-identifier my-db-replica --source-db-instance-identifier my-db
aws rds promote-read-replica --db-instance-identifier my-db-replica   # manual, one-way, possible data loss
```
Asynchronous, always readable, NOT automatic failover, can be cross-region.

## Aurora Serverless v2

```bash
aws rds create-db-cluster --db-cluster-identifier my-aurora \
  --engine aurora-postgresql --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16
```

## Multi-AZ vs Read Replica — one-line answer

```
Multi-AZ:      availability, sync, standby unreadable, auto failover, zero data loss
Read Replica:  read scaling, async, always readable, MANUAL promotion, possible data loss
```

## ElastiCache Redis

```bash
# cluster mode disabled: 1 shard
aws elasticache create-replication-group --replication-group-id my-redis \
  --engine redis --cache-node-type cache.r6g.large --num-node-groups 1 --replicas-per-node-group 2

# cluster mode enabled: N shards, hash-slot partitioned (16384 slots, CRC16(key) mod 16384)
aws elasticache create-replication-group --replication-group-id my-redis-cluster \
  --engine redis --cache-node-type cache.r6g.large --num-node-groups 3 --replicas-per-node-group 1
```
```bash
redis-cli -c CLUSTER KEYSLOT mykey
# hash tags force co-location: {user:123}:profile and {user:123}:sessions -> same slot
```

## DynamoDB

```bash
aws dynamodb create-table --table-name Orders \
  --attribute-definitions AttributeName=CustomerId,AttributeType=S AttributeName=OrderId,AttributeType=S \
  --key-schema AttributeName=CustomerId,KeyType=HASH AttributeName=OrderId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

aws dynamodb update-table --table-name Orders --attribute-definitions AttributeName=Status,AttributeType=S \
  --global-secondary-index-updates '[{"Create":{"IndexName":"StatusIndex","KeySchema":[{"AttributeName":"Status","KeyType":"HASH"}],"Projection":{"ProjectionType":"ALL"}}}]'

aws dynamodb update-table --table-name Orders --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```
Partition key hash determines physical placement — low-cardinality/hot key = throttled hot partition regardless of capacity. GSIs are eventually consistent, own separate write capacity. Streams retain 24h of change events.
