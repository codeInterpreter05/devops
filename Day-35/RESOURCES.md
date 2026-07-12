# Day 35 — Resources: AWS Storage & Databases

## Primary (assigned)

- **AWS RDS documentation** (docs.aws.amazon.com/AmazonRDS) — the assigned starting point. Read "High Availability (Multi-AZ)" and "Working with Read Replicas" directly back-to-back — the docs are explicit about the sync-vs-async and readable-vs-unreadable distinctions covered in file 2.

## Deepen your understanding

- **AWS — "Managing your storage lifecycle" (S3 documentation)** — full reference on lifecycle rule mechanics, including the versioning interaction covered in file 1.
- **AWS — "Replicating objects" (S3 documentation)** — covers CRR/SRR requirements, the non-retroactive limitation, and S3 Batch Replication for backfilling.
- **AWS — "Amazon RDS Multi-AZ DB cluster deployments" documentation** — covers the newer hybrid Multi-AZ option that blends fast failover with readable standbys, worth knowing exists beyond the classic model.
- **AWS — "Best practices for designing and using partition keys effectively" (DynamoDB documentation)** — the authoritative, official treatment of hot-partition avoidance directly relevant to Lab 4.
- **Redis documentation — "Scale with Redis Cluster"** (redis.io) — official explanation of the hash slot model and hash tags, useful beyond just ElastiCache since the same mechanics apply to any Redis Cluster deployment.

## Reference / lookup

- **AWS RDS CLI reference** (`aws rds help`) and **DynamoDB CLI reference** (`aws dynamodb help`) — exact syntax for every command in today's cheatsheet.
- **AWS — "Aurora Serverless v2" documentation** — ACU scaling mechanics and current min/max capacity limits.

## Practice

- **AWS Skill Builder — "Amazon DynamoDB Deep Dive" (free digital course)** — structured, free practice specifically reinforcing partition key design and GSI tradeoffs from today's material.
