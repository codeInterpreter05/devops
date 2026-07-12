# Day 89 — Resources: Prometheus Fundamentals

## Primary (assigned)

- **Prometheus official documentation** (prometheus.io/docs) — start with "Concepts" (data model, metric types) then "Querying" (PromQL basics, functions). The assigned free starting point for today.
- **PromQL for Humans** (timber.io/blog/promql-for-humans, mirrored on several blogs) — a much gentler, example-first walkthrough of `rate()`, `increase()`, and aggregation than the official reference docs.

## Deepen your understanding

- **"Histograms and summaries" — Prometheus docs** (prometheus.io/docs/practices/histograms) — the canonical explanation of why histograms aggregate and summaries don't, straight from the source.
- **robustperception.io blog** (archived but still widely referenced) — deep, practitioner-written posts on cardinality, relabeling, and recording rule design from the original Prometheus team.
- **"Alerting and Recording rules" — Prometheus docs** (prometheus.io/docs/prometheus/latest/configuration/alerting_rules) — the authoritative reference for `for:`, labels/annotations templating, and rule group semantics.

## Reference / lookup

- **Alertmanager configuration docs** (prometheus.io/docs/alerting/latest/configuration) — full reference for `route`, `receivers`, `inhibit_rules` syntax.
- **kube-prometheus-stack Helm chart README** (github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) — every configurable value, including how to add custom `PrometheusRule`/`ServiceMonitor` CRDs.

## Practice

- **PromLabs PromQL exercises** (promlabs.com/promql-exercises) — free, interactive PromQL katas that check your query results against expected output.
- **killercoda.com Prometheus scenarios** — free, browser-based Kubernetes + Prometheus sandboxes if you don't want to spin up your own cluster for the lab.
