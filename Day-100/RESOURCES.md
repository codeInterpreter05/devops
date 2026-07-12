# Day 100 — Resources: Datadog / New Relic (Commercial Tools)

## Primary (assigned)

- **Datadog documentation** (docs.datadoghq.com) — the assigned starting point for this day; covers Agent architecture, Kubernetes deployment (DaemonSet + Cluster Agent), APM, log management, and monitors directly from the vendor's own reference docs.

## Deepen your understanding

- **Datadog's "Kubernetes Monitoring" and "Cluster Agent" architecture docs** — the specific pages explaining why the Cluster Agent exists and how it fans in API server calls; worth reading before you deploy this at real cluster scale.
- **New Relic's NRQL reference and "Understand data ingest and pricing" docs** — the clearest place to understand NRDB's unified-datastore model and how New Relic's ingest+seats pricing actually works, so the Datadog-vs-New-Relic comparison isn't guesswork.
- **"OpenTelemetry and Datadog" integration docs** — how Datadog ingests OTLP data directly, relevant if you're carrying forward OTel instrumentation from Day 99's ADOT work instead of using `dd-trace`.
- **Honeycomb's and Datadog's own public blog posts on observability pricing/cardinality** — useful, vendor-adjacent but candid discussions of why high-cardinality tagging is the actual cost driver across every commercial platform, not a Datadog-specific quirk.

## Reference / lookup

- **Datadog Helm chart repo** (github.com/DataDog/helm-charts) — the source of truth for every configurable value when installing the Agent/Cluster Agent via Helm.
- **Datadog Monitor types reference** and **New Relic NRQL syntax reference** — bookmark both for writing real alert rules and queries without guessing syntax.

## Practice

- Run the same instrumented workload through both Datadog APM and your existing OTel-based tracing setup from Day 99 (as in Lab 4) — a first-hand, side-by-side comparison using identical traffic is far more convincing (to yourself and to an interviewer) than reciting vendor marketing claims about either approach.
