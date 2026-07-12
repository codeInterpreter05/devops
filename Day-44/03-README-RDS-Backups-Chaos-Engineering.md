# Day 44 — AWS Reliability & DR: RDS Backups/PITR & Chaos Engineering

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

A DR architecture is a theory until you've (a) actually tested that you can restore your database to any point in time, and (b) actually broken something on purpose to see if your failover works. This file covers RDS's backup/restore mechanics (the RPO half of the equation for your data tier) and Chaos Engineering on AWS (how you validate the RTO/RPO numbers you've claimed instead of just trusting the architecture diagram).

## RDS automated backups & Point-in-Time Recovery (PITR)

RDS takes a **daily automated snapshot** during your configured backup window, plus continuously ships **transaction logs** to S3 every 5 minutes. Together, these let you restore to **any point within your retention window**, not just to a snapshot boundary:

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db \
  --target-db-instance-identifier prod-db-restored \
  --restore-time 2026-07-10T14:32:00Z
```

**Key mechanics:**
- **Retention period**: 0-35 days for automated backups (0 disables them — never do this in production; it also disables PITR entirely). Restoring always creates a **new** DB instance — you cannot restore in place, which matters for RTO calculations (spin-up time for the new instance/data volume counts against your RTO).
- **RPO from automated backups**: effectively ~5 minutes, bounded by the transaction log shipping interval — this is why "RPO < 1 min" (this day's interview scenario) cannot be met by automated backups/PITR alone; it requires Multi-AZ synchronous replication or a continuously-replicating read replica instead.
- **Manual snapshots** persist beyond the retention window and beyond instance deletion (unlike automated backups, which are deleted when you delete the instance unless you explicitly keep a final snapshot) — take a manual snapshot before any risky schema migration or major version upgrade.
- **Cross-region automated backup replication** (a distinct feature from a full cross-region read replica) copies your automated backups/snapshots to a second region continuously, so you can PITR-restore into a DR region even if the primary region is unavailable — a lighter-weight option than running a full standing cross-region replica.
- **Storage-level snapshots are incremental** — after the first full snapshot, each subsequent automated snapshot only stores changed blocks, which is why backup windows have minimal performance impact on modern RDS (this used to cause I/O freezes on Magnum/Magnetic storage; gp2/gp3/io1/io2 with the newer snapshot mechanism largely eliminated the pause).

**Aurora is a different mechanism worth distinguishing in an interview**: Aurora's storage layer is already distributed across 3 AZs with 6 copies of data, and backups are continuous and incremental at the storage layer — Aurora PITR restore granularity is much finer (down to the second) and Aurora Global Database gives typical replication lag under 1 second cross-region, which *can* meet a sub-1-minute RPO target where standard RDS cross-region replication (which is logical/async at the engine level) cannot.

## Chaos Engineering on AWS — AWS Fault Injection Simulator (FIS)

Chaos Engineering is the discipline of **deliberately injecting failure** into a system to verify it behaves the way your architecture diagram claims — the only way to convert an assumed RTO into a measured, proven one.

**AWS FIS** is the managed service for this:
- An **experiment template** defines: **actions** (what to break — e.g., `aws:ec2:stop-instances`, `aws:ec2:terminate-instances`, `aws:rds:reboot-db-instances`, `aws:eks:pod-delete`, CPU/memory/network stress via the SSM agent, network latency/packet-loss injection), **targets** (which resources, selected by tag/ARN/filter — never "all resources" without narrowing), and **stop conditions** (a CloudWatch alarm that, if triggered, automatically halts the experiment — critical safety mechanism so a chaos test doesn't turn into a real incident).
- Common experiment for this day's hands-on activity — **simulate an AZ failure**: FIS doesn't have a literal "kill an AZ" button (AWS doesn't expose that), so the realistic approach is to target **all EC2 instances / EKS nodes in one specific AZ** with a stop or network-disruption action, and separately block that AZ's subnet route (or use security groups/NACLs to sever the AZ) to simulate the effect of losing it.
- Best practice: run first in a non-production environment, define a clear **hypothesis** ("if we lose AZ-2, the ALB should stop routing to it within N seconds and error rate should stay under X%"), have a stop condition wired to an error-rate alarm, and have a specific person owning "abort the experiment" authority during the run.

**Why this matters for the RTO/RPO conversation**: a chaos experiment is literally how you measure your actual RTO — start a timer when you inject the failure, stop it when traffic is fully healthy again, and compare against your target. If reality doesn't match the target, that's a finding, not a failure of the exercise — it's exactly what the exercise is for.

**Runbook discipline**: this day's hands-on activity explicitly asks you to "document a runbook" after the AZ failure simulation. A good runbook captures: the trigger/symptom that indicates this failure mode, the exact steps taken (including any manual steps that couldn't be automated), the measured time-to-recovery, and a follow-up action item for anything that took longer than expected or required manual intervention that should be automated next.

## Points to Remember

- RDS automated backups + PITR give roughly a 5-minute RPO (bounded by transaction log shipping interval) — not sufficient for sub-minute RPO targets; that requires Multi-AZ sync replication or Aurora Global Database.
- Restoring from a snapshot/PITR always creates a new DB instance — factor instance spin-up time into your RTO estimate, it is not instantaneous.
- Manual snapshots persist independent of the source instance's lifecycle and retention settings; automated backups do not survive instance deletion unless you keep a final snapshot.
- AWS FIS experiments need targets scoped precisely (never "all resources") and a CloudWatch-alarm-based stop condition as a safety net — chaos engineering without an abort mechanism is just an outage you triggered on purpose.
- A DR architecture's RTO/RPO numbers are unverified assumptions until a real chaos/failover test has measured them and produced a runbook.

## Common Mistakes

- Believing "we have automated backups" satisfies an aggressive sub-minute RPO requirement without checking the actual transaction-log-shipping interval (~5 minutes) that bounds standard RDS PITR.
- Deleting an RDS instance without checking the "keep a final snapshot" option, only to discover the automated backup history is gone along with the instance.
- Running a chaos experiment against production with a target selector broad enough to hit resources outside the intended blast radius (e.g., forgetting to scope by AZ tag and accidentally affecting multiple AZs).
- Skipping the stop-condition/alarm setup on an FIS experiment "because it's just a quick test," turning a controlled experiment into an actual incident with no automatic brake.
- Treating a DR runbook as a one-time deliverable instead of something re-validated periodically — infrastructure and dependencies change, and a runbook tested once a year ago may no longer reflect the real failover path.
