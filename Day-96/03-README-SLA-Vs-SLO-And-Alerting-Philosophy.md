# Day 96 — SLOs, SLAs & Error Budgets: SLA vs. SLO & SLO-Based Alerting

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** SRE | **Flag:** ⚡ Interview-critical

## Brief

Two things trip people up even after they understand SLIs/SLOs/error budgets: (1) confusing an SLO with an SLA — they look similar but serve completely different purposes and audiences, and (2) not understanding *why* SLO-based alerting is considered a strictly better default than plain threshold alerting. This file closes both gaps, and connects error budgets (file 2) to the concrete tooling landscape (Sloth, Nobl9) used to operationalize them.

## SLA — Service Level Agreement: the contractual layer

An **SLA** is a contractual, often legally binding commitment made to *customers* (external, paying customers, typically), with **consequences for violation** — usually a service credit, refund, or penalty clause. Where an SLO is an internal engineering target used to guide operational decisions, an SLA is a business/legal document, and it's usually **looser** than the internal SLO.

The relationship in practice:

```
SLA (external, contractual, e.g. 99.5% — what you PROMISE customers, with financial penalty if missed)
  <
SLO (internal, operational target, e.g. 99.9% — what you ACTUALLY aim for day to day)
  <
SLI actual performance (what you're really achieving, ideally comfortably above the SLO)
```

**Why the SLO is deliberately set tighter than the SLA:** you want the internal alarm (SLO burn-rate alert) to trigger and give your team time to react *before* you're anywhere near breaching the contractual, financially-penalized SLA. If your SLO and SLA were the same number, you'd have zero margin — by the time an SLO-based alert fires, you might already be in SLA-violation territory with customers owed refunds. The gap between SLO and SLA is your **operational safety margin**.

Practically: not every service needs an SLA at all — SLAs are typically only formalized for external-facing, revenue-relevant, or enterprise-contract-bound services (a cloud provider's API, a B2B SaaS platform's uptime commitment). Internal tools and most microservices have SLOs without any corresponding SLA — there's no external contract to violate, but internal reliability targets still matter for prioritization and alerting.

## SLO-based alerting vs. threshold alerting

**Threshold alerting** (the older, naive default) alerts on a static condition being crossed, evaluated independent of any budget or trend: "CPU > 80%," "error rate > 1%," "latency > 500ms." It answers "is this specific metric currently bad," full stop.

**SLO-based alerting** (error-budget burn-rate alerting, file 2) instead asks "**at the current rate of badness, will we run out of error budget before the window resets** — and if so, how urgently do we need to act?" This is a fundamentally different question, and it produces better alerts for several concrete reasons:

1. **Threshold alerts don't distinguish "brief, harmless blip" from "sustained, budget-threatening degradation."** A single 30-second spike to 5% errors that then recovers is often operationally meaningless (users barely notice, retries absorb it) — but a naive `error_rate > 1%` threshold alert fires anyway, training on-call engineers to ignore or snooze alerts (alert fatigue), which is dangerous because it desensitizes them to the alerts that *do* matter.
2. **Threshold alerts don't account for the SLO's own tolerance.** A service with a 99% SLO (much error budget available) firing an identical alert to a service with a 99.99% SLO (almost no budget) at the same raw threshold makes no sense — the same 1% error rate is a non-event for the first service and a serious budget-burning event for the second. Burn-rate alerting is inherently relative to *that specific service's* agreed budget, so the alert severity automatically reflects actual risk to that service's promise.
3. **Threshold alerts don't tell you urgency.** "Error rate is above 1%" doesn't say whether you have 5 minutes or 5 days before this becomes a real problem. Multi-window burn-rate alerts (file 2) explicitly encode urgency into severity: a 14.4x burn rate pages immediately (budget gone in ~2 days if sustained), a 3x burn rate opens a ticket (budget gone in ~a week) — the alert itself tells the responder how fast they need to move.
4. **SLO-based alerting is symptom-based, not cause-based**, which aligns with the broader SRE alerting philosophy: page on user-facing symptoms (the SLI is degrading, users are actually affected) rather than on every possible internal cause (a single replica's CPU spiked, one node's disk is at 85%) — many internal anomalies self-heal or don't affect users at all, and paging on every one of them is a leading cause of on-call burnout (connects directly to Day 97's incident management and Day 98's toil topics).

**This doesn't mean threshold alerts disappear entirely** — they're still useful for capacity/infrastructure-level signals that aren't directly SLO-mapped (disk filling up, certificate expiring soon) where "will this become a problem eventually" is the right question, just on a different timescale than burn-rate math suits. The shift is in what you page a human for at 3am: symptom/budget-based alerts for that, threshold-based tickets/dashboards for everything else.

## Tooling: Sloth and Nobl9

- **Sloth** (an open-source SLO generator for Prometheus) takes a simple, declarative SLO spec (service name, SLI query, target percentage, alerting windows) and generates the full set of Prometheus recording rules and multi-window burn-rate alerting rules for you — codifying the file 2 math so teams don't hand-write error-prone PromQL for every service's SLO.
  ```yaml
  # Sloth SLO spec (simplified)
  apiVersion: sloth.slok.dev/v1
  kind: PrometheusServiceLevel
  metadata:
    name: checkout-api
  spec:
    service: "checkout-api"
    slos:
      - name: "requests-availability"
        objective: 99.9
        sli:
          events:
            errorQuery: sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[{{.window}}]))
            totalQuery: sum(rate(http_requests_total{job="checkout-api"}[{{.window}}]))
        alerting:
          name: CheckoutAPIHighErrorRate
          pageAlert: {labels: {severity: page}}
          ticketAlert: {labels: {severity: ticket}}
  ```
  Running `sloth generate` against this spec produces ready-to-apply Prometheus rule YAML implementing the multi-window burn-rate pattern from file 2 automatically.
- **Nobl9** — a commercial, vendor-agnostic SaaS SLO management platform: define SLOs once against many backend data sources (Prometheus, Datadog, CloudWatch, BigQuery, etc.), get unified error-budget dashboards, burn-rate alerting, and reporting across an entire organization's services, without hand-building Prometheus rules per-service. It's the "SLOs as a managed product" answer for orgs with heterogeneous telemetry stacks where a Prometheus-only tool like Sloth isn't sufficient on its own.

## Points to Remember

- SLA = external, contractual, financially penalized, usually looser than the SLO. SLO = internal, operational, no direct financial penalty, usually tighter — the gap between them is your safety margin to react before a contract is breached.
- Not every service needs an SLA — only external/revenue/contract-relevant services typically do; internal tools have SLOs without SLAs.
- SLO-based (burn-rate) alerting alerts on "are we on pace to exhaust budget," which naturally scales severity to each service's own tolerance and encodes urgency — threshold alerting does neither, and tends to either miss slow burns or fire on harmless blips.
- The broader philosophy: page humans on user-facing symptom/budget signals; use lower-urgency channels (tickets, dashboards) for cause-level infra signals that don't yet threaten the SLO.
- Sloth auto-generates Prometheus SLO/burn-rate rules from a declarative spec; Nobl9 is the cross-backend, managed-SaaS equivalent for heterogeneous telemetry stacks.

## Common Mistakes

- Using "SLA" and "SLO" interchangeably in conversation or documentation — an interviewer (or a legal/business stakeholder) will immediately notice, since they have materially different audiences and consequences.
- Setting the internal SLO equal to (or, worse, looser than) the external SLA — leaves zero margin to react before a contractual, financially-penalized breach.
- Keeping only old-style static threshold alerts after adopting SLOs, rather than replacing paging alerts with burn-rate alerts — leads to continued alert fatigue from the same noisy thresholds SLOs were meant to fix.
- Paging on-call for every internal cause-level anomaly (single-node CPU, one pod restarting) instead of reserving pages for actual user-facing/budget-threatening symptoms — a major driver of on-call burnout and alert desensitization.
- Adopting an SLO tool (Sloth/Nobl9) without first doing the SLI-definition work from file 1 — the tool automates the *math*, but if the underlying SLI query doesn't reflect real user experience, you get precisely-calculated alerts on the wrong thing.
