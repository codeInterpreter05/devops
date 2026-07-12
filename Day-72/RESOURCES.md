# Day 72 — Resources: Load & Chaos Testing

## Primary (assigned)

- **k6 documentation** (k6.io/docs) **+ "Principles of Chaos Engineering"** (principlesofchaos.org) — free, the assigned starting points for this day. The k6 docs cover scripting and thresholds in depth; the Principles site is the short, canonical statement of chaos engineering's core tenets (steady state, hypothesis, blast radius, run in production).

## Deepen your understanding

- **"Chaos Engineering" (O'Reilly, by Casey Rosenthal & Nora Jones)** — the foundational book from Netflix's chaos engineering team, covering both the technical practice and the organizational/cultural side (including game days) in depth.
- **LitmusChaos documentation** (litmuschaos.io/docs) — full experiment library (pod, node, network, application-level faults) and `ChaosEngine`/`ChaosResult` reference.
- **AWS Fault Injection Simulator documentation** — official reference for supported actions per AWS service, experiment templates, and `stopConditions` configuration.
- **Locust documentation** (locust.io) — official docs on distributed load generation, custom clients beyond HTTP, and the `on_start`/task-weighting model.

## Reference / lookup

- **k6 `thresholds` and `checks` reference** (k6.io/docs/using-k6/thresholds) — exact syntax for percentile/rate-based pass/fail criteria.
- **Netflix Tech Blog — Chaos Monkey / Simian Army** — the original production accounts of chaos engineering's origins at Netflix, useful for real-world grounding of the principles.

## Practice

- Run the combined k6-load-plus-LitmusChaos-pod-delete exercise from this day's lab against a real HPA-backed Deployment — watching an HPA scale out in real time while a pod is simultaneously killed is the single most convincing way to internalize why "prove it, don't assume it" matters.
- Design (on paper, even without running it) a full game day scenario for a system you know well — pick a specific realistic fault, define the steady-state metrics you'd watch, and write the retrospective questions you'd ask afterward. This is close to how the actual interview question is often explored in follow-up depth.
