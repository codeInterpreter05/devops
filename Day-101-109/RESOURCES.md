# Day 101-109 — Resources: Observability Capstone + AWS SAA-C03

## Primary (assigned)

- **AWS SAA-C03 study guide (TutorialsDojo)** — the assigned best resource for this block's parallel exam track. Known for scenario-style practice questions that closely mirror the real exam's phrasing and difficulty, with detailed per-question explanations that are more useful for closing gaps than a raw right/wrong score. Use it as your final-week calibration tool (see LAB.md Part 5 and 03-README-AWS-SAA-C03-Exam-Day.md).

## Prometheus, Grafana, Loki, Tempo — official docs

- **Prometheus documentation** (prometheus.io/docs) — especially the "Querying" section for PromQL semantics and the "Operator" concepts if using `kube-prometheus-stack`; the querying basics and `histogram_quantile` docs are worth reading end to end at least once, not just skimmed via cheatsheets.
- **kube-prometheus-stack Helm chart README** (github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) — the actual values.yaml reference for every flag used in this block's LAB.md; the chart bundles the Prometheus Operator, so this is also the fastest way to understand `ServiceMonitor`/`PodMonitor`/`PrometheusRule` CRDs in practice.
- **Grafana documentation** (grafana.com/docs/grafana) — the "Data sources" and "Explore" sections cover exemplars, derived fields, and trace-to-logs/metrics configuration used throughout this block's correlation setup.
- **Grafana Loki documentation** (grafana.com/docs/loki) — read "Best practices" for label design specifically; it's the clearest explanation of why cardinality matters here and what goes wrong when it's ignored.
- **Grafana Tempo documentation** (grafana.com/docs/tempo) — the "TraceQL" section for query syntax, and "Trace discovery" for how Tempo intentionally leans on exemplars/metrics-generated indexes rather than full-text search.
- **OpenTelemetry documentation** (opentelemetry.io/docs) — the "Collector" section (architecture, receivers/processors/exporters) and language-specific auto-instrumentation guides for whichever sample app you build in LAB.md Part 2.

## SLOs and error budgets

- **Google SRE Book — Chapter 4: Service Level Objectives** (sre.google/sre-book/service-level-objectives) — the original source for SLI/SLO/SLA definitions used throughout this block; free online.
- **Google SRE Workbook — Chapter 5: Alerting on SLOs** (sre.google/workbook/alerting-on-slos) — the source of the multi-window, multi-burn-rate alerting technique in 02-README and LAB.md Part 3, including the derivation of the 14.4x/6x/3x/1x multipliers used in the cheatsheet's threshold table. Read this chapter directly at least once — most third-party summaries (including this repo's notes) simplify the derivation.
- **Google SRE Book — Chapter 3: Embracing Risk** (sre.google/sre-book/embracing-risk) — the error-budget-as-policy framing (why 100% reliability is the wrong target).

## PagerDuty and incident response

- **PagerDuty Incident Response documentation** (response.pagerduty.com) — PagerDuty's own open incident-response process guide; covers escalation policy design, severity/urgency mapping, and on-call scheduling patterns referenced in LAB.md Part 4.
- **PagerDuty + Prometheus Alertmanager integration guide** (PagerDuty's official docs site, search "Prometheus integration guide") — the exact `pagerduty_configs` receiver syntax and routing-key setup used in this block's lab.
- **"Effective DevOps" / runbook-writing guides** — search for PagerDuty's own "How to write a good runbook" post; it covers the mitigation-first, exact-commands structure this block's runbook template follows.

## AWS SAA-C03 practice exams and study

- **AWS Certified Solutions Architect – Associate exam guide** (aws.amazon.com/certification) — the official domain breakdown and task statements; re-check this against your practice-exam weak spots before final-week review.
- **Tutorials Dojo SAA-C03 practice exams** — paired with the primary resource above; use for the two full-length timed practice exams called out in LAB.md Part 5 and 03-README-AWS-SAA-C03-Exam-Day.md.
- **AWS Skill Builder** (skillbuilder.aws) — free official practice questions and exam-prep content directly aligned with the exam guide.
- **AWS Well-Architected Framework** (docs.aws.amazon.com/wellarchitected) — the fast pillar pass recommended in the final-week checklist; a large share of scenario questions implicitly test which pillar a given tradeoff favors.

## Reference / lookup

- **PromQL cheat sheet by PromLabs** (promlabs.com/promql-cheat-sheet) — quick-reference companion to this block's CHEATSHEET.md for less common PromQL functions not covered here.
- **LogQL and TraceQL official reference pages** (within the Loki/Tempo docs above) — the authoritative syntax reference if this block's cheatsheet examples don't cover a specific query you need.
