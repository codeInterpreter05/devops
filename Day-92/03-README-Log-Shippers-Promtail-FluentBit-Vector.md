# Day 92 — Loki & Log Aggregation: Log Shippers — Promtail vs. Fluent Bit vs. Vector

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Logging | **Flag:** ⚡ Interview-critical

## Brief

Loki (or Elasticsearch/OpenSearch) is only half the picture — something has to actually read log files/container stdout off every node, attach the right labels, and ship them to the storage backend. This is the log-shipper/agent layer, and picking one is a real architectural decision with real tradeoffs, similar in spirit to choosing between Thanos and VictoriaMetrics on Day 91: multiple credible tools solve the same problem with different resource footprints and ecosystems.

## Promtail — the Loki-native default

**Promtail** is Grafana Labs' own log shipper, purpose-built as Loki's companion agent (same relationship as Prometheus + node-exporter, or Thanos + Sidecar — a first-party component designed around one specific backend). In Kubernetes, Promtail typically runs as a **DaemonSet** (one pod per node), tailing container log files directly off each node's filesystem (`/var/log/pods/...`) and using **Kubernetes service discovery** to automatically attach pod/namespace/container labels — very similar in spirit to how Prometheus's `kubernetes_sd_configs` auto-discovers scrape targets.

```yaml
# promtail-config.yaml (excerpt)
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
    pipeline_stages:
      - json:
          expressions:
            level: level
      - labels:
          level:
```

Notice the `relabel_configs` syntax is deliberately near-identical to Prometheus's — another deliberate design choice to minimize new concepts for teams already running Prometheus. `pipeline_stages` is Promtail's log-processing chain (parse JSON, extract a field, promote it to a label) — conceptually similar to LogQL's `| json` parser stage, but happening at ingest time if you choose to promote a field to an actual indexed label (a decision that directly affects cardinality — see file 4).

**As of the Grafana Labs 2024 announcement, Promtail is in maintenance mode** and being deprecated in favor of **Grafana Alloy** (Grafana's unified OpenTelemetry-based collector) — worth knowing for anyone starting a *new* deployment today, though Promtail remains extremely widely deployed in existing production systems and still fully functional.

## Fluent Bit — the lightweight, broadly-adopted CNCF standard

**Fluent Bit** is a CNCF graduated project, written in C, designed to be extremely lightweight (small memory/CPU footprint — routinely cited in the low tens of MB of RAM per instance) and is the default/most common log shipper baked into managed Kubernetes offerings (EKS's recommended logging add-on, GKE, etc.). It's backend-agnostic: the same Fluent Bit deployment can ship to Loki, Elasticsearch/OpenSearch, S3, Kafka, CloudWatch, Datadog, or several destinations simultaneously via its plugin-based **input → filter → output** pipeline model.

```ini
# fluent-bit.conf (excerpt)
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            docker
    Tag               kube.*

[FILTER]
    Name              kubernetes
    Match             kube.*
    Merge_Log         On
    Keep_Log          Off

[OUTPUT]
    Name              loki
    Match             *
    Host              loki.observability.svc.cluster.local
    Labels            job=fluentbit
```

Fluent Bit's `kubernetes` filter is what enriches raw container log lines with pod metadata (namespace, pod name, labels) by querying the Kubernetes API — functionally similar to what Promtail's `kubernetes_sd_configs` does, but implemented as an explicit pipeline filter stage rather than a discovery+relabel model.

## Vector — the newer, high-performance, config-flexible option

**Vector** (originally by Timber.io, now part of Datadog) is written in Rust, and is generally positioned as higher-throughput and more memory-efficient than Fluent Bit under heavy load, with a more expressive configuration/transform language (**VRL — Vector Remap Language**) for reshaping log data mid-pipeline without needing external scripting.

```toml
# vector.toml (excerpt)
[sources.k8s_logs]
type = "kubernetes_logs"

[transforms.parse_json]
type = "remap"
inputs = ["k8s_logs"]
source = '''
  . = parse_json!(.message)
'''

[sinks.loki]
type = "loki"
inputs = ["parse_json"]
endpoint = "http://loki.observability.svc.cluster.local:3100"
labels.namespace = "{{ kubernetes.pod_namespace }}"
```

VRL is a real differentiator: instead of chaining many small filter plugins with limited expressiveness (as in Fluent Bit's config), Vector lets you write a small, purpose-built transformation script inline — parsing, reshaping, dropping, and enriching fields with actual conditional logic, which becomes valuable once your log pipeline needs anything beyond "tail, tag, ship."

## Choosing between them — the honest tradeoff

| | Promtail | Fluent Bit | Vector |
|---|---|---|---|
| **Best paired with** | Loki specifically (first-party) | Any backend, especially when already the platform default (EKS/GKE) | Any backend, when you need complex in-flight transforms or very high throughput |
| **Resource footprint** | Moderate | Very low (C, minimal) | Low-moderate (Rust, efficient, but generally does more work) |
| **Config expressiveness** | Prometheus-relabel-style, adequate but limited | Plugin-chain based, adequate for common cases | VRL — most expressive, real scripting for complex reshaping |
| **Ecosystem status** | Deprecated in favor of Grafana Alloy | Extremely widely adopted, CNCF graduated, cloud-provider default | Newer, growing fast, backed by Datadog |

A defensible default for a team newly adopting Loki today: **Fluent Bit** if you want the most broadly-supported, lowest-footprint, "boring and proven" choice (and it's often already installed as your cloud provider's default logging agent anyway); **Vector** if your pipeline needs real transformation logic beyond simple field extraction/labeling; **Grafana Alloy** (Promtail's successor) if you're all-in on the Grafana stack and starting fresh.

## Points to Remember

- Promtail is Loki's first-party agent (Prometheus-relabel-style config) but is now in maintenance mode, superseded by Grafana Alloy for new deployments.
- Fluent Bit is the lightweight, CNCF-graduated, backend-agnostic standard — often the pre-installed default on managed Kubernetes, using an input → filter → output plugin pipeline.
- Vector (Rust, Datadog-owned) differentiates on performance and VRL, a real inline scripting language for complex log transformation that plugin-chain configs struggle to express.
- All three solve the same core problem (tail logs, enrich with Kubernetes metadata, ship to one or more backends) — the decision axis is resource footprint, transformation complexity needs, and how invested you already are in a specific ecosystem (Grafana vs. cloud-provider-default vs. Datadog).
- Whichever shipper you choose, the labels it attaches at ingest time directly become Loki's index — this decision connects directly to the cardinality discussion in the next file.

## Common Mistakes

- Deploying Promtail for a brand-new project today without knowing it's in maintenance mode — not wrong per se (it still works), but a team should make that choice knowingly, not by default inertia, given Grafana Alloy is the forward path.
- Assuming "lightweight" (Fluent Bit) automatically means "less capable" — Fluent Bit handles the overwhelming majority of real-world Kubernetes logging needs perfectly well; reach for Vector's VRL only when you actually hit a transformation Fluent Bit's filter plugins can't express cleanly.
- Running multiple log shippers side-by-side "just in case" (e.g., both the cloud provider's default Fluent Bit and a separately-installed Promtail) without a clear reason — doubles resource consumption and log volume/cost for no benefit if they're shipping the same data to the same or overlapping destinations.
- Not tuning buffer/batch/retry settings under log-volume spikes — all three shippers can fall behind or drop data during a logging burst (e.g., a crash-looping pod flooding stdout) if buffers aren't sized for worst-case burst volume, silently creating gaps in exactly the logs you need most during an incident.
- Forgetting that whatever the shipper promotes into a Loki *label* (not just log content) is subject to the same cardinality rules as Prometheus labels — a shipper config that labels by a high-cardinality field is a self-inflicted Loki cardinality problem, not a Loki design flaw.
