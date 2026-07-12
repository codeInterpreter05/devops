# Day 44 — AWS Reliability & DR: RTO/RPO and Multi-AZ vs. Multi-Region

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

Every reliability decision in AWS — Multi-AZ vs. Multi-Region, synchronous vs. asynchronous replication, backup frequency — is really a decision about how much data loss and downtime the business can tolerate, expressed as two numbers: RTO and RPO. Interviewers ask about these constantly because they force you to reason about *tradeoffs and cost* rather than reciting "just use Multi-Region, it's more reliable" — which is almost never the right answer once you account for cost and complexity. This file covers the vocabulary and the Multi-AZ/Multi-Region architecture decision; the next two files cover the specific AWS mechanisms (Route 53 failover, S3 CRR, RDS backups, Chaos Engineering) that implement these targets.

## RTO and RPO — define them precisely

- **RTO (Recovery Time Objective)**: the maximum acceptable **time** between a failure occurring and the system being back up and serving traffic. "RTO = 15 minutes" means: however you get there, restoring service must take 15 minutes or less.
- **RPO (Recovery Point Objective)**: the maximum acceptable amount of **data loss**, measured in time. "RPO = 5 minutes" means: after recovery, you can have lost at most the last 5 minutes of writes — not that recovery takes 5 minutes.

These are independent and commonly confused in interviews — get asked to design a system with a specific RTO/RPO and immediately state both numbers back before touching the whiteboard, because conflating them leads to designing the wrong thing (e.g., designing fast failover when the real requirement was zero data loss).

**Why they cost money in opposite ways to what people assume:** RPO is driven by **replication frequency/method** (synchronous replication → RPO near zero, but costs latency and often cross-AZ bandwidth; async replication/periodic snapshots → cheaper, but RPO = however stale the last replicated copy is). RTO is driven by **how much manual work is needed to fail over** (automated health-check-triggered failover → low RTO; "someone gets paged, SSHs in, restores a snapshot" → high RTO, often 30+ minutes even for a well-drilled team).

**Example decision from this day's interview question — "design a system with RTO < 5 min and RPO < 1 min":**
- RPO < 1 min rules out anything relying on hourly snapshots; you need continuous/near-continuous replication — e.g., RDS Multi-AZ synchronous replication (RPO effectively 0 for the standby) or DynamoDB Global Tables (typically sub-second replication lag).
- RTO < 5 min rules out any manual intervention step; you need **automated failover** — RDS Multi-AZ automatic failover typically completes in 60-120 seconds; Route 53 health-check-based DNS failover with a low TTL can redirect traffic in well under a minute once the health check itself detects the failure (health check interval + failure threshold matters here — see file 02).
- Putting it together: RDS Multi-AZ (or Aurora with a reader promoted) for the data tier, Route 53 failover routing with aggressive health checks for the traffic tier, and Multi-AZ (not necessarily Multi-Region) application tier behind an ALB, since Multi-AZ alone already satisfies both targets at a fraction of Multi-Region's cost and complexity.

## Multi-AZ vs. Multi-Region

**Availability Zones (AZs)** are physically separate data centers within a region, connected by high-bandwidth, low-latency private links — close enough for **synchronous replication** to be practical (sub-2ms typical inter-AZ latency), far enough apart to not share power/cooling/flooding risk. This is why almost every managed AWS reliability feature (RDS Multi-AZ, EKS across AZs, ALB across AZs) defaults to AZ-level redundancy — it's the highest-value, lowest-cost place to add resilience.

**Regions** are geographically distant, fully independent AWS deployments — they do not share underlying infrastructure at all. Cross-region replication is **inherently asynchronous** for anything with real transactional consistency requirements, because forcing synchronous writes across, say, us-east-1 and eu-west-1 would add 70-150ms+ of latency to every write — usually unacceptable.

**When Multi-AZ is enough (the common case):**
- Protects against: hardware failure, a single data center outage, most power/network issues within a region.
- Does NOT protect against: a full regional outage (rare, but AWS regions have had multi-hour/multi-service outages), a region-wide network partition, or a region-specific compliance/legal requirement (data residency).
- Cost: relatively cheap — roughly 2x the compute (standby) plus some cross-AZ data transfer, no cross-region data transfer, no application logic for handling geographically split traffic.

**When you need Multi-Region:**
- Regulatory/compliance requirements for geographic data separation or true disaster recovery independent of a single region's control plane.
- Business requirement that genuinely cannot tolerate a regional outage (rare — most companies say they need this and don't actually justify the cost/complexity when pressed).
- Global latency requirements — users on multiple continents need low-latency access, which is a *performance* argument for multi-region, separate from the *reliability* argument.

**The honest interview answer**: "Most systems should be Multi-AZ by default because it's cheap and covers the overwhelming majority of real failure modes. Multi-Region should be justified by a specific compliance requirement, a demonstrated business cost of regional downtime that exceeds the (nontrivial) cost and operational complexity of running active infrastructure in two regions, or a genuine global-latency need — not adopted reflexively because it sounds more resilient." Being able to push back on over-engineering a Multi-Region setup is itself a strong signal in an interview.

**DR strategy spectrum** (increasing RTO/RPO improvement, increasing cost) — a standard framework worth knowing by name:
1. **Backup & Restore** — highest RTO/RPO (hours), cheapest. Just snapshots/backups in another region, restored on demand.
2. **Pilot Light** — core data replicated continuously (e.g., RDS cross-region read replica) but compute is not running, spun up on failover. RTO in tens of minutes.
3. **Warm Standby** — a scaled-down but fully functional copy of the full stack running in the DR region at all times, scaled up on failover. RTO in minutes.
4. **Multi-Site Active/Active** — full production capacity running in both regions simultaneously, traffic split/failed-over via DNS. Lowest RTO/RPO (seconds), highest cost and operational complexity.

## Points to Remember

- RTO = time to recover; RPO = data lost. They are independent and driven by different mechanisms (RTO by failover automation, RPO by replication frequency/method).
- Multi-AZ is synchronous-replication-friendly (low latency between AZs) and covers the vast majority of real-world failure scenarios at a fraction of Multi-Region's cost.
- Multi-Region replication is inherently asynchronous for most stateful services because of speed-of-light latency — meaning Multi-Region almost always increases your RPO for the region-to-region hop, even while improving your protection against a regional outage.
- The four-tier DR strategy spectrum (Backup & Restore → Pilot Light → Warm Standby → Multi-Site Active/Active) is the standard vocabulary for framing a DR cost/RTO tradeoff conversation.
- Neither RIs/Savings Plans nor Multi-AZ automatically means "we have a DR plan" — a DR plan requires a defined RTO/RPO target, a tested failover procedure, and a runbook, not just redundant infrastructure sitting there.

## Common Mistakes

- Confusing RTO and RPO in conversation (or on a whiteboard) — stating "our RPO is 15 minutes" when actually describing how long a failover takes (that's RTO).
- Reaching for Multi-Region as a default "we want to be reliable" architecture choice without a concrete justification, taking on 2x+ the cost and real operational complexity (cross-region data consistency, DNS failover logic, testing overhead) for a failure mode (full regional outage) that's far rarer than AZ-level failures.
- Assuming Multi-AZ automatically gives near-zero RPO for all AWS services — it does for RDS/Aurora (synchronous), but not for services or self-managed databases where you've configured asynchronous cross-AZ replication.
- Having a documented DR architecture but never actually testing failover (see file 03's Chaos Engineering section) — an untested RTO number is a guess, not a guarantee.
- Treating backups as a DR strategy on their own without accounting for restore time at scale — restoring a multi-terabyte database from a snapshot can itself take far longer than the stated RTO target if nobody has timed it.
