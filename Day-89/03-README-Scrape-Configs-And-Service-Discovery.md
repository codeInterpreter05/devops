# Day 89 — Prometheus Fundamentals: Scrape Configs & Service Discovery

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** ⚡ Interview-critical

## Brief

Prometheus's biggest operational advantage over push-based systems is that it decides *what* to scrape and *when*, using a `scrape_configs` block plus a service discovery mechanism — so you never have to hand-maintain a list of IPs as pods churn. Understanding how targets get discovered and how `relabel_configs` reshapes them before scraping is what separates "I installed kube-prometheus-stack and it mostly worked" from actually being able to debug why a target shows up as `down` or why a metric is missing an expected label.

## Anatomy of a scrape config

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /metrics
    static_configs:
      - targets: ['10.0.1.10:9100', '10.0.1.11:9100']
        labels:
          env: production
```

- `job_name` becomes the value of the `job` label on every series scraped by this block — it's how you distinguish "these metrics came from node-exporter" from "these came from the app itself" in queries.
- `scrape_interval` / `scrape_timeout` can be set per-job, overriding the global default — useful when one exporter is slow to respond and needs a longer timeout, or when a low-priority job can tolerate a longer interval to reduce load.
- Every scrape target automatically gets an `instance` label (`host:port` by default) in addition to `job` — together `job` + `instance` uniquely identify "which process, on which machine, exposed this metric."

Static configs work fine for a handful of stable IPs, but in Kubernetes, pods are ephemeral and IPs churn constantly — this is where **service discovery (SD)** takes over.

## Service discovery: `kubernetes_sd_configs`

Instead of a static list, Prometheus can be told to continuously watch the Kubernetes API for a resource type and auto-generate targets:

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

`role: pod` tells Prometheus to enumerate every pod cluster-wide via the Kubernetes API (other roles: `service`, `endpoints`, `node`, `ingress`). For each discovered pod, Prometheus generates a set of **meta labels** prefixed `__meta_kubernetes_*` — namespace, pod name, labels, annotations, container ports, node name, and more. These meta labels exist only during the discovery/relabeling phase; by default they are **dropped before scraping** unless you explicitly keep or rename them via `relabel_configs`. This is exactly why the `prometheus.io/scrape: "true"` annotation convention exists: the first `relabel_config` above uses `action: keep` to filter the target list down to only pods carrying that annotation, so Prometheus doesn't attempt (and fail, and clutter `up{}` with noise) to scrape every single pod in the cluster — most of which don't expose `/metrics` at all.

## `relabel_configs` vs `metric_relabel_configs` — the distinction that trips everyone up

- **`relabel_configs`** run **before the scrape happens**, against the discovered target's meta labels. They decide *whether* to scrape a target at all (`action: keep`/`drop`) and *how* to construct the final `__address__` and `__metrics_path__`.
- **`metric_relabel_configs`** run **after the scrape**, against the actual samples returned by the target. They're used to drop unwanted, high-cardinality metrics or rewrite label values *after* you already paid the cost of scraping them over the network.

```yaml
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'go_gc_duration_seconds.*'
        action: drop
```

If your goal is "don't scrape this target at all," use `relabel_configs`. If your goal is "I scraped it, but I don't want to store this particular metric," use `metric_relabel_configs` — but understand you still paid the network/CPU cost of the scrape itself; for truly unwanted targets, filtering earlier with `relabel_configs` is cheaper.

## Points to Remember

- `job` + `instance` labels uniquely identify "which job, on which host/pod" a series came from — both are added automatically.
- `kubernetes_sd_configs` continuously watches the K8s API for a given `role` (pod/service/endpoints/node) and turns matching resources into scrape targets automatically — no manual IP maintenance.
- Discovery produces `__meta_kubernetes_*` labels that exist only transiently; you must promote the ones you want (namespace, pod name, etc.) into real labels via `relabel_configs`, or they vanish before storage.
- `relabel_configs` run pre-scrape (decide targets/address/path); `metric_relabel_configs` run post-scrape (decide which returned samples to keep/drop/rewrite).
- The `prometheus.io/scrape`, `prometheus.io/port`, `prometheus.io/path` pod-annotation convention is implemented entirely via `relabel_configs` — it's not a Kubernetes-native feature, just a widely-adopted pattern.

## Common Mistakes

- Forgetting that meta labels (`__meta_kubernetes_*`) are dropped by default and never appear on stored series — then being confused why a query can't filter by namespace even though "the discovery clearly saw it."
- Using `metric_relabel_configs` to drop a noisy/high-cardinality metric from a target you're scraping thousands of times a minute, instead of using `relabel_configs` to avoid scraping the target's irrelevant metrics endpoint in the first place — wastes network and CPU even after the "fix."
- Not setting a longer `scrape_timeout` for slow exporters (e.g., a database exporter that runs an expensive query per scrape), causing intermittent `context deadline exceeded` scrape failures that look like flaky monitoring rather than a config issue.
- Relying on `static_configs` in a Kubernetes environment for anything beyond a quick demo — pod IPs get recycled and stale targets silently stop reporting with no obvious error beyond `up == 0`.
- Not double-checking that `__address__` was rewritten correctly in a custom relabel chain — a typo'd regex can silently produce an unreachable address, and the failure just looks like "target down" with no clue why.
