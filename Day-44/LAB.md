# Day 44 — Lab: AWS Reliability & DR

**Goal:** Simulate a real AZ failure against your EKS cluster, measure the actual RTO, and produce a runbook — not just read about DR theory.

**Prerequisites:** An EKS cluster spanning at least 2 AZs (from earlier days), `kubectl` access, AWS CLI with permissions for EC2/EKS/FIS/Route 53/RDS (a sandbox account is strongly preferred — this lab intentionally breaks things), and ideally a small RDS instance for Labs 2-3 (a free-tier `db.t3.micro` is fine).

---

### Lab 1 — Define RTO/RPO for a real system before touching anything

1. Pick a real (or your practice) application running on your EKS cluster. Write down, concretely:
   - Current RTO if a full AZ died right now — your honest best guess.
   - Current RPO — how much data could you lose given your actual backup/replication setup today.
2. State a *target* RTO and RPO (e.g., RTO < 5 min, RPO < 1 min) and identify, on paper, which of today's four DR-spectrum tiers (Backup & Restore / Pilot Light / Warm Standby / Multi-Site Active/Active) your current setup actually matches.

**Success criteria:** You have a written "as-is" and "target" RTO/RPO for one real system, plus which DR tier you're actually operating at today (often it's more honest — and lower — than assumed).

---

### Lab 2 — RDS PITR restore, timed

1. Create a small RDS instance if you don't have one, insert a few rows, note the exact time.
2. Wait 10+ minutes, insert more rows (a second, distinguishable batch), note that time too.
3. Start a stopwatch and restore to the point in time right after the *first* batch:
   ```bash
   aws rds restore-db-instance-to-point-in-time \
     --source-db-instance-identifier <source> \
     --target-db-instance-identifier <source>-pitr-test \
     --restore-time <timestamp-after-first-batch>
   ```
4. Stop the stopwatch once the restored instance is `available` and you've confirmed (by querying) that only the first batch of rows is present.

**Success criteria:** You have a measured, real wall-clock time for a PITR restore-to-available, and you've confirmed the restored data cuts off exactly where expected — giving you real numbers instead of assumed ones for RTO/RPO on this data tier.

---

### Lab 3 — Simulate an AZ failure for your EKS cluster (the assigned hands-on activity)

1. Identify which AZ(s) your EKS worker nodes live in: `kubectl get nodes -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels.topology\\.kubernetes\\.io/zone`.
2. Pick one AZ to "fail." Using AWS FIS, create an experiment template targeting EC2 instances tagged with that AZ, using the `aws:ec2:stop-instances` action, with a stop condition tied to a CloudWatch alarm on your ALB's 5xx error rate.
   ```bash
   aws fis create-experiment-template --cli-input-json file://az-failure-template.json
   ```
3. Before starting the experiment, start a timer and begin watching your application's error rate / availability (CloudWatch dashboard from Day 43, or `kubectl get pods -o wide --watch`).
4. Start the experiment:
   ```bash
   aws fis start-experiment --experiment-template-id <id>
   ```
5. Observe: do pods get rescheduled onto healthy-AZ nodes? Does the ALB stop routing to the failed AZ's targets? How long until traffic is fully healthy? Stop your timer at that point — that's your measured RTO for this failure mode.
6. If you don't have FIS access or want a simpler substitute: manually cordon and drain all nodes in one AZ (`kubectl cordon <node>` then `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`) and time recovery the same way.

**Success criteria:** A measured, real RTO number for an AZ failure against your actual cluster — not a guess — plus observed evidence of whether pod anti-affinity/topology spread (if configured) actually redistributed replicas across the surviving AZs.

---

### Lab 4 — Document the runbook

Write a runbook (a markdown file is fine) covering: the failure symptom/trigger, exact remediation steps (automated and manual), the measured RTO from Lab 3, and at least one concrete follow-up action for anything that was slower or more manual than it should have been.

**Success criteria:** A runbook that a teammate unfamiliar with this exercise could follow to diagnose and respond to the same failure mode.

---

### Cleanup

```bash
aws rds delete-db-instance --db-instance-identifier <source>-pitr-test --skip-final-snapshot
kubectl uncordon <node>   # if you cordoned/drained manually instead of using FIS
aws fis delete-experiment-template --id <id>
```

### Stretch challenge

Configure Route 53 failover routing with health checks pointing at two ALBs (or two target groups in different AZs), then re-run the AZ-failure simulation and measure end-to-end DNS-level failover time — compare it against the Kubernetes-level recovery time measured in Lab 3, and explain in your runbook which layer (DNS failover vs. pod rescheduling) actually dominated your total RTO.
