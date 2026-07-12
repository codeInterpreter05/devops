# Day 91 — Resources: Thanos & Long-term Metrics

## Primary (assigned)

- **Thanos official documentation** (thanos.io/tip/thanos/quick-tutorial.md) — the assigned free starting point; the "Getting Started" and component-by-component docs (Sidecar, Store Gateway, Compactor, Query) are the clearest source for how the pieces fit together.

## Deepen your understanding

- **"Prometheus: Up & Running" (O'Reilly, Brian Brazil)** — the chapter on federation, remote storage, and operational limits gives the "why" behind retention/cardinality constraints from one of Prometheus's original authors.
- **VictoriaMetrics official documentation — "Cluster architecture"** (docs.victoriametrics.com/cluster-victoriametrics) — the clearest explanation of vminsert/vmstorage/vmselect directly from the source, plus their published benchmark comparisons against Thanos/Cortex/Mimir.
- **"Monitoring cardinality" — Grafana Labs blog** — practical, example-driven walkthrough of finding and fixing cardinality explosions using exactly the queries covered in this day's notes.
- **CNCF Thanos project page + KubeCon talks (YouTube, free)** — search "Thanos global view Prometheus" for conference talks that walk through real production deployments and failure scenarios.

## Reference / lookup

- **Thanos "Storage" docs** (thanos.io/tip/thanos/storage.md) — the authoritative reference for supported object storage backends and their config formats (S3, GCS, Azure, MinIO).
- **Prometheus "Metric and label naming" official guide** (prometheus.io/docs/practices/naming) — the canonical naming-convention reference this day's first file summarizes.

## Practice

- **thanos.io Quick Tutorial (local Docker Compose setup)** — the official tutorial spins up Sidecar + Query + a local MinIO-backed bucket entirely via Docker Compose, a faster on-ramp than the full Kubernetes lab if you want to see the wiring first.
- **killercoda.com Prometheus/Thanos scenarios** — free browser-based sandboxes if you'd rather not provision your own cluster + MinIO for this lab.
