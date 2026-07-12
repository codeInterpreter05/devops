# Day 72 — Load & Chaos Testing: AWS Fault Injection Simulator, Game Days, and Fire Drills

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** Testing | **Flag:** ⚡ Interview-critical

## Brief

LitmusChaos operates at the Kubernetes layer — pods, nodes, network within a cluster. Real production incidents just as often originate one layer down, in the cloud infrastructure itself: an AZ going unhealthy, an EBS volume experiencing I/O degradation, an EC2 instance being stopped outright. **AWS Fault Injection Simulator (FIS)** is AWS's own managed chaos engineering service for exactly this layer. And chaos experiments (whether Kubernetes-level or infrastructure-level) reach their full value only when paired with **game days** — structured, human-in-the-loop incident-response rehearsals — which is the piece that turns "we ran a chaos tool" into genuine organizational readiness.

## AWS Fault Injection Simulator (FIS)

FIS is a managed AWS service for injecting controlled faults directly against real AWS resources — EC2, EKS, ECS, RDS, and more — with the same blast-radius and safety controls that responsible chaos engineering requires, but expressed as an AWS-native construct (an "Experiment Template") rather than Kubernetes CRDs.

```json
{
  "description": "Stop 25% of EC2 instances in the web-tier ASG",
  "targets": {
    "web-tier-instances": {
      "resourceType": "aws:ec2:instance",
      "resourceTags": { "tier": "web" },
      "selectionMode": "PERCENT(25)"
    }
  },
  "actions": {
    "stop-instances": {
      "actionId": "aws:ec2:stop-instances",
      "parameters": { "startInstancesAfterDuration": "PT5M" },
      "targets": { "Instances": "web-tier-instances" }
    }
  },
  "stopConditions": [
    {
      "source": "aws:cloudwatch:alarm",
      "value": "arn:aws:cloudwatch:us-east-1:111111111111:alarm:high-error-rate"
    }
  ]
}
```

```bash
aws fis create-experiment-template --cli-input-json file://experiment-template.json
aws fis start-experiment --experiment-template-id EXT123456789
aws fis get-experiment --id EXP123456789   # check status/results
```

**`stopConditions` is the single most important safety feature to understand.** It wires a real CloudWatch alarm directly into the experiment: if that alarm fires (e.g., error rate crosses a threshold) *at any point during the experiment*, FIS automatically halts the experiment immediately — this is the "automated abort mechanism" from chaos engineering's blast-radius-minimization principle, built directly into the tool rather than something you have to hand-roll yourself.

### What FIS can target that LitmusChaos can't (and vice versa)

FIS operates at the AWS resource layer — stopping/terminating EC2 instances, injecting API throttling/errors (simulating an AWS service itself misbehaving), degrading EBS volume I/O, simulating an AZ failure. LitmusChaos operates inside the Kubernetes layer — killing pods, injecting network chaos between Services, stressing a container's CPU/memory. A rigorous test of "does my EKS-based application survive an AZ failure" genuinely needs FIS (to simulate the AZ event) *and* likely LitmusChaos-style validation of how the application's pods respond once nodes in that AZ become unavailable — the two tools operate at different, complementary layers of the same stack.

## Game days — the human dimension chaos tooling alone doesn't cover

A **game day** is a scheduled, structured exercise where a team deliberately simulates an incident (often using a chaos tool like FIS or LitmusChaos to trigger the actual technical fault) and then runs their **real incident response process** against it — paging, runbooks, communication, decision-making — exactly as they would for a genuine production incident, except everyone involved knows in advance it's a drill (usually with a designated facilitator who knows the details and observes without immediately revealing what's happening to responders).

**Why this matters beyond "did the system technically recover":**
- Chaos tooling proves the *system* is resilient. Game days additionally prove the *people and process* are resilient — whether the right people get paged, whether the runbook is actually accurate and current (a shockingly common finding: runbooks reference dashboards/commands that no longer exist), whether the team's communication under pressure is effective.
- Game days surface **organizational** gaps chaos tooling alone never will: an on-call engineer who doesn't have the right IAM permissions to actually execute the fix at 2am, a runbook link that 404s, a Slack channel nobody's actually monitoring, an escalation path that assumes someone who's since left the company.
- They build **muscle memory and psychological readiness** — a team that has actually practiced responding to a simulated database failover under pressure responds meaningfully faster and calmer to the real one than a team that's only ever read about the procedure.

### A realistic game day structure

1. **Define scope and a specific scenario** in advance (facilitator-only knowledge): e.g., "primary RDS instance fails over unexpectedly during business hours."
2. **Set a blast-radius-safe experiment** to actually trigger a representative fault (FIS targeting the RDS instance, or a controlled manual failover) — not the responders' job to know this is coming.
3. **Run the real incident response process**, unannounced to the responding team beyond "this is a drill happening now" — paging, war room, runbook execution, actual decision-making — while the facilitator observes and takes notes without intervening unless truly necessary.
4. **Timebox and safely conclude** — a predetermined stop condition/time limit, with the facilitator able to immediately halt the underlying fault injection.
5. **Blameless retrospective** — the most important step: what worked, what didn't, what surprised the team, and concrete action items (fix the stale runbook, grant the missing IAM permission, add a missing alert) with owners and deadlines. A game day that doesn't produce concrete follow-up actions has captured only half its value.

## Points to Remember

- AWS FIS operates at the cloud infrastructure layer (EC2, RDS, EBS, AZ-level faults, simulated AWS API errors) — complementary to, not a replacement for, Kubernetes-layer tools like LitmusChaos.
- `stopConditions` tied to a real CloudWatch alarm is FIS's built-in automated-abort mechanism — the practical implementation of chaos engineering's blast-radius-minimization principle, not something you need to hand-build.
- A game day pairs a chaos-tool-triggered technical fault with the team's *real* incident response process (paging, runbooks, communication) — proving organizational/human readiness, not just system resilience.
- Game days routinely surface gaps chaos tooling alone cannot: stale runbooks, missing on-call permissions, dead escalation paths, unmonitored alert channels.
- A game day without a blameless retrospective and concrete follow-up action items has only captured half its value — the drill itself is not the point; the resulting fixes are.

## Common Mistakes

- Treating AWS FIS and LitmusChaos as interchangeable/competing tools rather than complementary layers — missing infrastructure-layer failure modes (AZ outages, EBS degradation) by only ever testing at the Kubernetes pod layer, or vice versa.
- Running an FIS experiment without a `stopConditions` alarm wired in, relying purely on a human watching a dashboard to notice and manually abort if things go wrong — slower and less reliable than an automated stop condition.
- Announcing every detail of a game day scenario to responders in advance ("we're going to fail over RDS at 2pm today") — this tests the technical fault-recovery path but defeats the point of rehearsing *realistic, unannounced* incident response, communication, and decision-making under genuine uncertainty.
- Running a game day and treating "the system recovered" as the only success metric, skipping the blameless retrospective — missing the organizational/process gaps (stale runbooks, missing permissions) that are often the actual valuable findings.
- Scheduling a game day once and considering the resilience question "answered" — like chaos experiments generally, game days lose value if not repeated periodically, especially after significant architecture changes, team turnover, or new services being added to the critical path.
