# Day 96 — Resources: SLOs, SLAs & Error Budgets

## Primary (assigned)

- **Google SRE Book — Chapter 4: Service Level Objectives** (sre.google/sre-book/service-level-objectives, free online) — the assigned starting point for this day, and the canonical source for SLI/SLO/error budget vocabulary used industry-wide.

## Deepen your understanding

- **Google SRE Workbook — Chapter 5: Alerting on SLOs** (sre.google/workbook/alerting-on-slos, free online) — the direct source for the multi-window, multi-burn-rate alerting pattern covered in file 2; includes the full burn-rate/window/budget-consumption table.
- **Google SRE Book — Chapter 3: Embracing Risk** (sre.google/sre-book/embracing-risk) — the conceptual foundation for *why* 100% reliability is the wrong target and how error budgets formalize risk tolerance as policy.
- **Sloth documentation** (sloth.dev) — practical docs for the SLO-generator tool used in today's lab; good source for seeing real-world SLO spec examples beyond the basic one used here.
- **Nobl9 blog / docs** (nobl9.com/resources) — useful for understanding the managed-SLO-platform category and how organizations operationalize SLOs across heterogeneous telemetry backends at scale.

## Reference

- **sre.google** — the umbrella site hosting all three free SRE books (SRE Book, SRE Workbook, Building Secure & Reliable Systems) — worth bookmarking as the canonical reference for this entire domain.
- **Prometheus histogram documentation** (prometheus.io/docs/practices/histograms) — essential background for correctly designing histogram bucket boundaries so latency SLIs are actually computable (the trap covered in file 1).

## Practice

- **Today's lab** (Sloth-generated burn-rate alerts against a live Prometheus + Alertmanager) — the most direct hands-on path to internalizing the burn-rate math.
- **Sloth's example repository** (github.com/slok/sloth/tree/main/examples) — a set of ready-made SLO specs for common service patterns; useful for comparing your own spec against a range of real examples.
