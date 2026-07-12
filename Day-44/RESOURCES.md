# Day 44 — Resources: AWS Reliability & DR

## Primary (assigned)

- **AWS Well-Architected Framework — Reliability Pillar** (docs.aws.amazon.com/wellarchitected) — free, the assigned starting point. Defines RTO/RPO framing and the DR strategy spectrum directly from AWS's own architectural guidance.

## Deepen your understanding

- **AWS Disaster Recovery whitepaper** ("Disaster Recovery of Workloads on AWS") — the canonical reference for the Backup & Restore / Pilot Light / Warm Standby / Multi-Site spectrum used in today's notes.
- **Route 53 Developer Guide — DNS Failover** — covers health check types, calculated health checks, and failover routing configuration in full detail.
- **Amazon Aurora Global Database documentation** — worth reading even if you're on standard RDS, to understand why Aurora can hit tighter RPO targets than logical replication.
- **AWS FIS documentation — "What is AWS Fault Injection Simulator"** — walks through experiment templates, actions, and stop conditions with real examples.

## Reference / lookup

- **S3 Replication documentation** — the authoritative reference on CRR/SRR prerequisites, delete-marker replication, and Replication Time Control.
- **RDS User Guide — Backing Up and Restoring** — backup retention, PITR restore mechanics, and snapshot lifecycle behavior.

## Practice

- **AWS FIS console "experiment template" gallery** — AWS ships several pre-built example templates (EC2 stop, network latency injection) you can adapt directly for Lab 3 instead of writing a template from scratch.
- **Principles of Chaos Engineering** (principlesofchaos.org) — short, framework-agnostic reference for designing a hypothesis-driven chaos experiment, applicable beyond just AWS FIS.
