# Day 35 — AWS Storage & Databases: RDS Multi-AZ, Read Replicas & Aurora Serverless v2

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** —

## Brief

"What is the difference between RDS Multi-AZ and a Read Replica? Can you promote a read replica?" is today's assigned interview question, and it's asked constantly because the two features sound similar (both involve a second copy of your database) but solve completely different problems — one is for **availability**, the other is for **read scaling**. Confusing them (or not knowing you can combine them) is a fast way to signal you haven't actually operated a production RDS database. Aurora Serverless v2 is the newer twist worth knowing on top of the classic RDS model.

## RDS Multi-AZ — availability, not scaling

```bash
aws rds create-db-instance --db-instance-identifier my-db --multi-az \
  --engine postgres --db-instance-class db.r5.large --allocated-storage 100 \
  --master-username admin --master-user-password ****
```

Multi-AZ provisions a **synchronous, physical-replication standby** in a different AZ from the primary. The standby is **not accessible for reads or writes under normal operation** — its entire purpose is failover: if the primary fails (AZ outage, instance failure, even a routine OS patching maintenance window), RDS automatically promotes the standby and repoints the DB's endpoint DNS record at it, typically within 60-120 seconds, with **no data loss** because replication to the standby is synchronous (a write isn't acknowledged as committed until it's confirmed on the standby too).

- **Purpose**: availability/durability — survive an AZ failure with automatic failover and zero data loss.
- **Read traffic**: cannot be served from the standby at all (in classic RDS Multi-AZ; note the newer "Multi-AZ DB cluster" deployment option, covered below, changes this).
- **Cost**: you pay for the full standby instance running continuously, purely as insurance — it does no useful work under normal conditions.

## Read Replicas — read scaling, not availability

```bash
aws rds create-db-instance-read-replica --db-instance-identifier my-db-replica \
  --source-db-instance-identifier my-db
```

A read replica is an **asynchronous** copy of your database, fully queryable, meant to offload read traffic from the primary (reporting queries, analytics, read-heavy application paths) so the primary isn't burdened with them. You can create **multiple** read replicas (up to 5 for standard RDS engines, more for Aurora), and — a detail worth knowing — a read replica can even live in a **different region** from its source, which Multi-AZ standbys cannot do.

- **Purpose**: horizontally scale read capacity; replicas are actively queryable at all times, unlike a Multi-AZ standby.
- **Replication**: asynchronous — there is **replica lag**, meaning a query against a replica can return slightly stale data compared to the primary; this lag is visible/monitorable (`ReplicaLag` CloudWatch metric) but not something you control precisely.
- **Failover**: NOT automatic. A read replica does not automatically take over if the primary fails.

## Can you promote a read replica? Yes — but understand what that actually means

```bash
aws rds promote-read-replica --db-instance-identifier my-db-replica
```

Promoting a read replica **detaches it from its source and turns it into a fully independent, standalone, writable RDS instance** — it stops receiving replication from the old primary and becomes its own primary going forward. This is a **manual, deliberate operation**, not an automatic failover mechanism — if your production primary dies and you only have read replicas (no Multi-AZ), you must manually promote one, which takes time (typically minutes) and, because replication was asynchronous, may involve **losing whatever writes hadn't yet replicated** to that replica at the moment of failure. This is precisely why "read replica" and "Multi-AZ" are not substitutes for each other for availability purposes — a manually-promoted replica is a disaster-recovery action with potential data loss and downtime; Multi-AZ failover is automatic with zero data loss by design.

## Combining both — the actual production pattern

Real production RDS setups typically run **both**: Multi-AZ for automatic, zero-data-loss failover against AZ failure, **plus** one or more read replicas (which can themselves be Multi-AZ) for read scaling. These are complementary, not either/or — Multi-AZ answers "what happens if my primary dies," read replicas answer "how do I serve more read traffic without overloading my primary."

## Multi-AZ DB clusters — the newer hybrid option

AWS's newer **"Multi-AZ DB cluster"** deployment option (distinct from the classic "Multi-AZ instance" described above) provisions **two readable standbys** across different AZs, using a faster, semi-synchronous replication protocol — giving you both faster failover (typically under 35 seconds) **and** readable standbys for read scaling, blurring the classic Multi-AZ/read-replica line. Worth knowing this exists so you're not caught off guard if an interviewer mentions it, but the *foundational* stateful-vs-scaling distinction between classic Multi-AZ and read replicas is still the core concept being tested.

## Aurora Serverless v2 — capacity that scales with load, not a fixed instance size

```bash
aws rds create-db-cluster --db-cluster-identifier my-aurora-cluster \
  --engine aurora-postgresql --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16
```

Aurora (AWS's own MySQL/PostgreSQL-compatible engine, with storage automatically replicated 6 ways across 3 AZs at the storage layer, independent of Multi-AZ instance concepts) offers a **Serverless v2** mode where compute capacity is expressed in **ACUs (Aurora Capacity Units)** and scales automatically, in fine-grained increments, based on actual load — no manually choosing/resizing a fixed instance class. This fits workloads with **unpredictable or highly variable** traffic (dev/test environments, infrequently-used internal tools, multi-tenant SaaS with spiky per-tenant load) where provisioning for peak capacity 24/7 would waste money, and manually resizing instances for every traffic pattern change isn't practical.

**Important nuance**: Serverless v2 scales *capacity within a running instance* quickly and without downtime — it is not the same as "scales to zero and disappears," and it's a different mechanism from read replicas/Multi-AZ (which are about topology, not capacity sizing). It can still be combined with Aurora's own read replica and Multi-AZ-equivalent (Aurora Replicas, which are always readable and can be promoted with much faster failover than standard RDS read replicas, since Aurora's storage layer is already shared).

## Points to Remember

- Multi-AZ = availability (automatic failover, zero data loss, standby unreadable); Read Replica = read scaling (manual promotion only, asynchronous, always readable, possible data loss on manual failover).
- Promoting a read replica is a manual, deliberate action that detaches it into an independent instance — it is not an automatic failover mechanism, and can lose unreplicated writes.
- Production systems typically run both Multi-AZ and read replicas together — they solve different problems and are not substitutes for each other.
- Multi-AZ DB clusters (the newer option) provide both fast automatic failover and readable standbys, blending the two classic concepts.
- Aurora Serverless v2 scales compute capacity (ACUs) automatically with load within a running instance — a capacity-sizing solution, distinct from the topology-focused Multi-AZ/read-replica decision.

## Common Mistakes

- Believing a read replica provides automatic failover the same way Multi-AZ does — it requires a manual `promote-read-replica` call, with real downtime and potential data loss from async replication lag.
- Assuming Multi-AZ standbys can serve read traffic to reduce primary load — under the classic Multi-AZ instance model, they cannot; that requires a read replica (or the newer Multi-AZ DB cluster option) instead.
- Running only read replicas with no Multi-AZ configuration on a production database, mistaking "I have a copy of the data somewhere" for "I have an availability strategy" — without Multi-AZ, a primary failure means manual, lossy recovery.
- Ignoring the `ReplicaLag` metric and routing time-sensitive reads (e.g., "read my own recent write") to a replica, then being confused when the just-written data doesn't appear yet.
- Conflating Aurora Serverless v2's capacity autoscaling with a topology/availability feature — it solves "right-size compute for variable load," not "survive an AZ failure" or "scale read throughput across replicas," which remain separate decisions even on Aurora.
