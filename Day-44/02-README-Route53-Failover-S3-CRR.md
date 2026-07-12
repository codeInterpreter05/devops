# Day 44 — AWS Reliability & DR: Route 53 Failover & S3 Cross-Region Replication

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

RTO and RPO targets (file 01) are abstract until you wire up the actual mechanisms that hit them. Route 53 health checks + failover routing is how DNS-level traffic redirection actually happens during a regional or AZ failure, and S3 Cross-Region Replication is the standard way to get durable, geographically separated copies of object data. Both show up in real DR architectures and in interview system-design questions about building resilient, globally available services.

## Route 53 health checks

A **health check** is Route 53 (from multiple global "checker" locations, by default) polling an endpoint — HTTP, HTTPS, or TCP — on an interval (default 30s, can be set to 10s "fast" checks) and marking it healthy/unhealthy based on a **failure threshold** (default: 3 consecutive failures before marking unhealthy).

Three health check flavors:
1. **Endpoint health check** — directly polls an IP/domain + path (e.g., `GET /health` expecting a 200).
2. **Calculated health check** — combines the status of up to 256 other health checks with AND/OR/NOT logic (e.g., "healthy only if at least 2 of 3 regional endpoints are healthy") — same composite idea as CloudWatch composite alarms.
3. **CloudWatch alarm-based health check** — ties a health check's state directly to a CloudWatch alarm, useful when "healthy" needs to reflect an internal metric (e.g., queue depth) rather than just "endpoint responds to HTTP."

**The RTO math that matters**: total detection-plus-failover time ≈ (health check interval × failure threshold) + DNS TTL + client-side DNS cache behavior. With a 10-second fast health check and a threshold of 3, detection alone takes ~30 seconds; add a low TTL (e.g., 60s or less) on the record, and total observed failover time for well-behaved clients lands in roughly 1-2 minutes — but note that **some clients and resolvers ignore TTL** (aggressive caching, some corporate DNS resolvers, some mobile carriers), which is a real-world caveat you should mention if asked "why didn't failover happen instantly."

## Route 53 routing policies for failover

- **Failover routing policy** — the simplest DR pattern: designate a Primary and Secondary record, each tied to a health check. Route 53 serves the Primary's answer while healthy; the instant the Primary's health check fails, it serves the Secondary automatically. No application logic needed.
- **Weighted routing** — split traffic by percentage across multiple endpoints; commonly used for canary/blue-green traffic shifting, not primarily DR, but combinable with health checks to auto-remove an unhealthy weighted target.
- **Latency-based routing** — route each user to whichever region has the lowest latency for them; used for Multi-Region *active/active* setups, and if combined with health checks, unhealthy regions are automatically excluded — this is how Multi-Site Active/Active DR (from file 01's spectrum) actually gets implemented at the DNS layer.
- **Geolocation / geoproximity routing** — route based on the user's geographic location rather than measured latency — used more for compliance/data-residency requirements than pure DR.

For the pure Primary/Secondary DR pattern this day is aimed at, **Failover routing policy with health checks on both records** is the textbook answer.

## S3 Cross-Region Replication (CRR)

CRR asynchronously copies objects from a source bucket to a destination bucket in a **different region** as they're written (also available same-region as **SRR**, mainly for compliance/latency reasons rather than DR).

**Mechanics worth knowing:**
- Requires **versioning enabled on both source and destination buckets** — CRR literally cannot be configured otherwise.
- Replicates new object writes going forward only — **existing objects at the time you enable CRR are not backfilled automatically**; use S3 Batch Replication for that.
- Replication is asynchronous — most objects replicate within seconds to minutes, but S3 offers **Replication Time Control (RTC)**, an SLA-backed option guaranteeing 99.9% of objects replicate within 15 minutes, for a small additional cost — relevant when your RPO target requires a bounded, guaranteed replication lag rather than "usually fast."
- You can replicate to multiple destination buckets, replicate only objects matching a prefix/tag filter, change storage class or ownership on the destination, and optionally replicate delete markers (off by default — meaning a deletion in the source does **not** propagate to the destination unless explicitly configured, which is often the *safer* default for accidental-deletion protection).
- IAM: CRR needs an IAM role with `s3:GetReplicationConfiguration`, `s3:GetObjectVersionForReplication`, etc. on the source and `s3:ReplicateObject`/`ReplicateDelete`/`ReplicateTags` on the destination — a common setup mistake is granting the role permissions on only one side.

**Why delete-marker replication defaults to off matters operationally**: if someone accidentally (or maliciously) deletes objects in the source bucket, the destination copy — with delete-marker replication off — remains intact, effectively giving you an accidental extra layer of protection against destructive mistakes, at the cost of the two buckets being able to drift out of sync on deletions. Decide deliberately rather than leaving it as an accidental default.

## Points to Remember

- Route 53 failover time is roughly (health check interval × failure threshold) + DNS TTL — tune both down for tighter RTO, but expect some clients/resolvers to ignore low TTLs anyway.
- Failover routing policy + health checks on Primary and Secondary is the standard, simplest Route 53 pattern for active/passive DR; latency-based routing is for active/active.
- CRR requires versioning on both buckets and does not backfill pre-existing objects — use S3 Batch Replication to backfill.
- CRR is asynchronous by default (seconds to minutes); Replication Time Control gives an SLA-backed 15-minute guarantee if your RPO target needs a bound rather than "usually fast."
- Delete-marker replication is off by default in CRR — deletions in the source do not propagate to the destination unless you explicitly enable it.

## Common Mistakes

- Setting a low DNS TTL and health check interval and assuming failover is instant for every client — some resolvers and cached clients ignore TTL, so real-world failover time has a long tail even with aggressive settings.
- Enabling CRR and assuming all existing objects are now replicated — they aren't; only new writes replicate going forward unless you run a Batch Replication job.
- Forgetting to enable versioning on the destination bucket (only doing the source), which silently prevents CRR from being configurable at all until fixed.
- Assuming CRR is synchronous/instant and relying on it for a sub-minute RPO without enabling Replication Time Control — plain CRR gives no SLA on replication lag.
- Not testing the Route 53 failover in practice — configuring Primary/Secondary records and health checks but never actually taking the primary down to confirm failover behaves as expected (ties directly into file 03's Chaos Engineering practice).
