# Day 95 — Cheatsheet: Jaeger & Tempo

## Jaeger components

```
jaeger-collector  -> ingest spans (OTLP/Jaeger/Zipkin), write to storage backend
jaeger-query      -> read path, serves API + UI queries
jaeger-ui         -> waterfall view, service dependency graph
storage backend   -> Cassandra | Elasticsearch/OpenSearch | in-memory (dev only) | Badger
jaeger-agent      -> legacy, UDP-based; prefer OTLP direct-to-collector now
```

## Jaeger quick start (all-in-one, dev only)

```bash
docker run -d --name jaeger \
  -p 16686:16686 -p 4317:4317 -p 4318:4318 \
  -e COLLECTOR_OTLP_ENABLED=true \
  jaegertracing/all-in-one:1.57
# UI: http://localhost:16686   (in-memory storage — data lost on restart)
```

## Tempo components

```
distributor  -> receive spans, hash by trace_id -> route to ingester
ingester     -> buffer in memory, flush blocks to object storage
object store -> S3 / GCS / Azure Blob / local disk / MinIO (the actual "DB")
querier      -> fetch blocks by trace_id for read queries
compactor    -> merge small blocks, enforce retention/TTL
```

## Minimal tempo.yaml (local backend)

```yaml
server:
  http_listen_port: 3200
distributor:
  receivers:
    otlp:
      protocols:
        grpc:
storage:
  trace:
    backend: local
    local:
      path: /tmp/tempo/blocks
```

## TraceQL examples

```
{}                                                  # all traces
{ status = error }                                  # error traces only
{ resource.service.name = "checkout-service" }      # by service
{ duration > 500ms }                                # slow traces
{ span.http.status_code = 500 && duration > 1s }    # combine conditions
```

## Three pillars correlation (Grafana wiring)

```
Metric (exemplar) --click--> Trace (Tempo)
Trace  (trace_id) --click--> Logs  (Loki query: {service="x"} |= "<trace_id>")
Log    (trace_id field) --click--> Trace (Tempo)
Trace  (spans) --auto--> RED Metrics (Tempo metrics-generator / OTel spanmetrics connector)
```

## Spotting N+1 in a trace waterfall

```
Symptom: many sibling spans, SAME operation name, SAME short duration,
         stacked sequentially under one parent, count scales with input size (N)

Fix: batch query (WHERE id IN (...)) | JOIN | ORM eager-load
     (Django: select_related/prefetch_related, SQLAlchemy: joinedload)
```

## RED metrics

```
Rate      -> requests/sec
Errors    -> error rate / fraction of requests failing
Duration  -> latency distribution (p50/p95/p99 — NEVER average of averages)
```

## Diagnosing high p99 latency — the loop

```
1. RED dashboard  -> confirm scope (is it this service? errors too, or pure latency?)
2. Slow traces    -> query backend for traces > p99 threshold, look at SEVERAL
3. Waterfall      -> find the dominant child span (biggest time contributor)
4. Check N+1      -> many small repeated spans vs one big slow span?
5. Trace-to-logs  -> jump to exact error/detail at that span
6. Correlate      -> recent deploy / config / feature flag change?
```

## telemetrygen (synthetic trace generator)

```bash
docker run --rm -e OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:4317 \
  ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  traces --otlp-insecure --traces 20
```
