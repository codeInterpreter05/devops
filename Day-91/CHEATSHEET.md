# Day 91 — Cheatsheet: Thanos & Long-term Metrics

## Cardinality-monitoring queries

```promql
prometheus_tsdb_head_series                          # total active series right now
count by (__name__)({__name__=~".+"})                  # series count per metric name
topk(10, count by (__name__)({__name__=~".+"}))         # top 10 highest-cardinality metrics
count(count by (job)(up))                              # sanity check: number of distinct jobs
scrape_samples_scraped                                  # samples returned by a target's last scrape
```

## Prometheus retention flags

```bash
--storage.tsdb.retention.time=15d      # default retention window
--storage.tsdb.retention.size=0        # size-based retention cap (bytes), 0 = disabled
--storage.tsdb.min-block-duration=2h    # local block size before it's immutable
--storage.tsdb.max-block-duration=2h    # keep equal to min to disable local compaction beyond 2h blocks (Thanos does the rest)
```

## Thanos object storage config

```yaml
# objstore.yml
type: S3
config:
  bucket: thanos
  endpoint: s3.amazonaws.com     # or MinIO endpoint for local labs
  access_key: <key>
  secret_key: <secret>
  insecure: false
```

## Thanos components at a glance

```
Sidecar        -> next to live Prometheus; serves recent data via StoreAPI; uploads blocks to object storage
Store Gateway  -> serves historical data FROM object storage via StoreAPI
Compactor      -> runs against bucket only; merges blocks, downsamples, enforces retention. ONE per bucket.
Query          -> Prometheus-API-compatible; fans out to Sidecars + Store Gateways; merges + dedups
```

## Thanos Query external labels + dedup

```yaml
# Prometheus config (per replica)
global:
  external_labels:
    cluster: prod-east
    replica: "0"          # "1" on the second HA replica
```
```bash
# Thanos Query flag to enable dedup on the replica label
--query.replica-label=replica
```

## Downsampling defaults (Compactor)

```
raw data           -> kept as-is
5m resolution       -> after 40h
1h resolution       -> after 10d
```

## VictoriaMetrics remote_write

```yaml
# prometheus.yml
remote_write:
  - url: http://vminsert:8480/insert/0/prometheus/
    queue_config:
      max_samples_per_send: 10000
      max_shards: 30
```

## VictoriaMetrics clustered components

```
vminsert   -> receives remote_write, distributes across storage nodes
vmstorage  -> horizontally scalable storage nodes
vmselect   -> handles queries (PromQL + MetricsQL), merges across vmstorage
```

## Comparison at a glance

```
                Thanos                          VictoriaMetrics
Model           Additive layer on Prometheus     Replacement storage/query backend
Ingest path     Sidecar reads local TSDB          remote_write directly
Storage         Object storage (S3/GCS/...)       Own storage engine (local or replicated)
Components      Sidecar/StoreGW/Compactor/Query    vminsert/vmstorage/vmselect
Best fit        Deep kube-prometheus-stack + object storage maturity   Lower resource footprint, fewer moving parts
```
