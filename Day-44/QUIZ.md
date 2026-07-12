# Day 44 — Quiz: AWS Reliability & DR

Try to answer without looking at your notes. Answers are at the bottom.

1. Define RTO and RPO precisely, and explain why they're driven by different underlying mechanisms.
2. Why is cross-region replication inherently asynchronous for most stateful services, while cross-AZ replication can be synchronous?
3. Name the four tiers of the DR strategy spectrum, in order of increasing cost/complexity and decreasing RTO/RPO.
4. What is the formula (approximate) for Route 53 failover time, and why might real-world failover still be slower than that formula suggests for some clients?
5. What's the difference between Route 53's Failover routing policy and Latency-based routing policy, in terms of the DR pattern each implements?
6. What two prerequisites are required before you can configure S3 Cross-Region Replication, and what does CRR NOT do automatically that people often assume it does?
7. What does S3 Replication Time Control (RTC) add on top of standard CRR?
8. What is the approximate RPO you get from standard RDS automated backups + PITR, and why is that number what it is?
9. Why does restoring an RDS instance from a snapshot/PITR always create a new instance, and why does that matter for RTO?
10. What is a "stop condition" in an AWS FIS experiment template, and why is it critical?
11. Why is Aurora Global Database able to hit a tighter RPO than standard cross-region RDS read replicas?
12. **Interview question:** What is the difference between RTO and RPO? Design a system with RTO < 5 min and RPO < 1 min.

---

## Answers

1. RTO (Recovery Time Objective) is the maximum acceptable time to restore service after a failure. RPO (Recovery Point Objective) is the maximum acceptable amount of data loss, measured in time. RTO is driven mainly by how automated your failover process is (manual steps = high RTO); RPO is driven mainly by replication frequency/method (synchronous replication = near-zero RPO, periodic snapshots = RPO bounded by snapshot interval).
2. Because forcing synchronous writes across geographically distant regions would add the full round-trip network latency (70-150ms+) to every write, which is usually unacceptable for application performance. AZs within a region are physically close enough (sub-2ms typical) that synchronous replication is practical without meaningfully hurting write latency.
3. Backup & Restore (cheapest, hours RTO) → Pilot Light (data replicated continuously, compute off, tens of minutes RTO) → Warm Standby (scaled-down full stack always running, minutes RTO) → Multi-Site Active/Active (full capacity in both locations, seconds RTO, highest cost).
4. Roughly (health check interval × failure threshold) + DNS TTL. Real-world failover can be slower because some clients/resolvers ignore or override the record's TTL (aggressive caching, some corporate/mobile DNS resolvers), so a subset of clients keep hitting the failed endpoint longer than the TTL alone would suggest.
5. Failover routing policy implements a classic active/passive pattern — Primary serves traffic until its health check fails, then Secondary takes over automatically. Latency-based routing implements active/active — traffic is routed to whichever healthy region has the lowest latency for that user, used for Multi-Site Active/Active DR and global performance optimization simultaneously.
6. Versioning must be enabled on **both** the source and destination buckets. CRR does NOT automatically backfill objects that existed before CRR was enabled — only new writes going forward replicate; you need S3 Batch Replication to backfill pre-existing objects. It also does not replicate delete markers by default.
7. RTC adds an SLA-backed guarantee that 99.9% of new objects replicate to the destination within 15 minutes, plus replication metrics — standard CRR gives no bound on replication lag (usually fast, but not guaranteed).
8. Roughly 5 minutes, because RDS ships transaction logs to S3 on a ~5-minute interval, which is what bounds how far back PITR can guarantee recovery to. For a sub-1-minute RPO you need Multi-AZ synchronous replication or Aurora Global Database instead of relying on automated backups/PITR alone.
9. Because RDS restore operations provision a brand-new DB instance from the backup/log data rather than modifying the existing instance in place — this matters for RTO because instance provisioning and storage attachment time (which can be several minutes or more depending on size) counts against your total recovery time, it isn't instantaneous.
10. A stop condition ties the experiment to a CloudWatch alarm — if the alarm triggers during the experiment (e.g., error rate exceeds a safe threshold), FIS automatically halts the experiment. It's critical because without it, a chaos experiment has no automatic brake and can turn into an uncontrolled real incident if things go worse than expected.
11. Aurora Global Database replicates at the storage layer using purpose-built, low-latency replication (typically under 1 second lag) rather than standard logical/engine-level asynchronous replication used by cross-region RDS read replicas, which typically has higher and less predictable lag — making Aurora Global Database capable of meeting much tighter RPO targets.
12. Strong answer: "RTO is how long it takes to recover; RPO is how much data you can afford to lose. For RTO < 5 min and RPO < 1 min, I'd use Aurora (or RDS Multi-AZ) with synchronous replication for the data tier to get RPO near zero, automated failover completing in 60-120 seconds; Route 53 failover routing with fast health checks (10s interval, threshold 3) and a low TTL for traffic-level failover in under a minute; and a Multi-AZ application tier behind an ALB so pods/instances reschedule automatically across AZs. I'd validate both numbers with an actual chaos engineering test (AWS FIS) rather than trusting the architecture diagram, and document the measured numbers in a runbook." Mention that Multi-Region is not required here since Multi-AZ alone can hit these targets at much lower cost, unless there's a specific full-region-outage or compliance requirement driving it.
