# Day 35 — AWS Storage & Databases: S3 Deep Dive

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** —

## Brief

Everyone can `aws s3 cp` a file. Fewer people can explain how S3's versioning interacts with lifecycle policies to actually save money, how cross-region replication works under the hood, or when access points solve a real problem that bucket policies can't. This file goes past "S3 is object storage" into the specific mechanics that come up when S3 is holding production data at real scale — this is foundational for the rest of the day's database topics too, since RDS snapshots, DynamoDB backups, and application data all frequently land in S3 eventually.

This day is split into three files:

1. **This file** — S3 lifecycle policies, versioning, replication, and access points.
2. **[02-README-RDS-And-Aurora.md](02-README-RDS-And-Aurora.md)** — RDS Multi-AZ vs. read replicas, and Aurora Serverless v2.
3. **[03-README-ElastiCache-And-DynamoDB.md](03-README-ElastiCache-And-DynamoDB.md)** — ElastiCache Redis cluster mode, and DynamoDB partition keys/GSIs/streams.

## Versioning — the foundation everything else builds on

```bash
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Enabled
```

Once enabled, S3 **never overwrites or deletes an object in place** — every `PUT` to the same key creates a new version, and every `DELETE` actually just adds a **delete marker** (a zero-byte placeholder that makes the object appear gone from normal listings) rather than removing prior versions. This is why versioning is the real backup mechanism for accidental overwrites/deletes: `aws s3api list-object-versions` shows every prior version, and you can restore by copying an old version back to the "current" slot, or simply delete the delete-marker to "undelete" the object.

**Important nuance**: versioning cannot be disabled once enabled — only **suspended** (`Status=Suspended`), which stops creating new versions going forward but does not remove any versions already created. This is a one-way door many engineers don't realize until they're looking for an "off" switch that isn't there.

## Lifecycle policies — automating the cost curve of data over time

```json
{
  "Rules": [{
    "ID": "archive-old-logs",
    "Status": "Enabled",
    "Filter": { "Prefix": "logs/" },
    "Transitions": [
      { "Days": 30, "StorageClass": "STANDARD_IA" },
      { "Days": 90, "StorageClass": "GLACIER" }
    ],
    "Expiration": { "Days": 365 },
    "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
  }]
}
```

A **lifecycle policy** automatically transitions objects between storage classes (Standard → Infrequent Access → Glacier → Deep Archive) based on object age, and can expire (delete) objects entirely after a retention period. This matters financially at real scale: Standard storage is the most expensive per-GB; most data becomes far less frequently accessed as it ages (old logs, old backups), and lifecycle rules automate the "move it to cheaper storage" decision instead of relying on someone manually auditing and moving data — a task that never actually happens consistently by hand.

**The versioning + lifecycle interaction that trips people up**: once versioning is enabled, *every* version of an object consumes storage and (unless explicitly addressed) never expires on its own. `NoncurrentVersionExpiration` specifically targets old, non-current versions for cleanup — without this rule, a versioned bucket's storage cost silently grows forever as every overwrite/delete just accumulates another billed version, even though the bucket "looks" the same size in a normal listing.

## Replication — CRR and SRR

```json
{
  "Role": "arn:aws:iam::123456789012:role/replication-role",
  "Rules": [{
    "Status": "Enabled",
    "Priority": 1,
    "Filter": {},
    "Destination": { "Bucket": "arn:aws:s3:::my-bucket-dr-region", "StorageClass": "STANDARD" }
  }]
}
```

**Cross-Region Replication (CRR)** and **Same-Region Replication (SRR)** asynchronously copy new objects from a source bucket to a destination bucket — CRR for disaster recovery/geographic redundancy or serving data closer to users in another region, SRR for aggregating logs from multiple buckets or maintaining a separate compliance/audit copy in the same region. Both require versioning enabled on **both** source and destination buckets — replication is built directly on top of the versioning mechanism, tracking which version has been replicated.

**Key limitation to know**: replication is **not retroactive** by default — enabling it only replicates objects created *after* the rule is enabled; existing objects need S3 Batch Replication (a separate, explicit one-time job) to backfill. It's also **not synchronous** — there's typically a replication lag (usually seconds, but not guaranteed instant), so treat cross-region copies as eventually consistent, not a live mirror for read-your-writes-style expectations.

## Access Points — scoped, named access to a shared bucket

```bash
aws s3control create-access-point --account-id 123456789012 --name finance-team-ap \
  --bucket my-shared-bucket --vpc-configuration VpcId=vpc-0abc123
```

An **Access Point** is a named, dedicated hostname/ARN with its own access policy, pointing at a specific bucket (or prefix within it) — instead of one enormous, hard-to-audit bucket policy trying to express every team's/application's specific access pattern, each consumer gets its own access point with a scoped policy. This matters at scale: a single shared bucket used by ten different applications/teams historically meant one bucket policy juggling ten sets of conditions; access points let each consumer be granted access through its own point, each independently auditable, and each optionally restricted to a specific VPC (so an access point can only be reached from within a designated VPC, adding a network-level restriction bucket policies alone can't express as cleanly).

## Points to Remember

- Once enabled, S3 versioning can only be suspended, never fully disabled — and every version (plus every delete marker) is retained and billed until explicitly expired.
- Lifecycle policies automate moving aging data to cheaper storage classes and expiring it — without `NoncurrentVersionExpiration`, a versioned bucket's cost grows unbounded as old versions pile up invisibly.
- CRR/SRR require versioning on both source and destination, are asynchronous (eventual, not instant), and are not retroactive — existing objects need a separate Batch Replication job to backfill.
- Access Points give each consumer of a shared bucket its own scoped, independently auditable access policy (and optionally a VPC restriction) instead of one sprawling bucket policy.
- A "0-byte" delete marker in a versioned bucket is not an empty file — it's S3's mechanism for making an object appear deleted from default listings while all prior real versions remain intact underneath it.

## Common Mistakes

- Enabling versioning without ever adding a `NoncurrentVersionExpiration` lifecycle rule, then being surprised months later that storage costs for a bucket have grown far beyond what the "current" object listing would suggest.
- Assuming replication is retroactive and enabling CRR expecting all existing objects to show up in the destination bucket immediately — only new objects (post-enablement) replicate automatically.
- Treating cross-region replication as a synchronous, always-consistent mirror and building logic that assumes a just-written object is immediately readable in the destination region.
- Trying to "turn off" versioning entirely on a bucket and being confused when only "Suspended" is available — existing versions are never removed by suspending.
- Managing access for many different consumer teams through one increasingly complex bucket policy instead of adopting Access Points, making it hard to audit or safely modify any single team's access without risking every other team's.
