# Day 94 — Cheatsheet: OpenTelemetry

## The three signals

```
Traces   -> request path across services (spans, tree, trace_id)
Metrics  -> aggregated numeric trends (Counter, Gauge, Histogram)
Logs     -> discrete timestamped events (correlated via trace_id/span_id)
```

## Python SDK — install & auto-instrument

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install     # installs instrumentors for detected libs

OTEL_SERVICE_NAME=my-service \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
OTEL_TRACES_EXPORTER=otlp \
opentelemetry-instrument uvicorn main:app --port 8000
```

## Manual instrumentation (Python)

```python
from opentelemetry import trace

tracer = trace.get_tracer("my-service")

with tracer.start_as_current_span("step_name") as span:
    span.set_attribute("key", "value")
    try:
        do_work()
    except Exception as e:
        span.record_exception(e)
        span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
        raise
```

## SDK wiring (manual, no launcher)

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

provider = TracerProvider(resource=Resource.create({"service.name": "my-service"}))
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="localhost:4317", insecure=True)))
trace.set_tracer_provider(provider)
```

## Metric instruments

```
Counter    -> monotonic, only increases   (http_requests_total)
Gauge      -> up or down                  (queue_depth)
Histogram  -> bucketed distribution       (http_request_duration_seconds) -> percentiles
```

## W3C Trace Context header

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^ ^^
             |  trace_id (32 hex, shared         parent span_id  flags
             |  across whole distributed trace)  (8 bytes)       (01=sampled)
             version

tracestate: vendor1=value1,vendor2=value2   # optional, multi-vendor extension
```

- Injected by the caller's HTTP client instrumentation; extracted by the callee's server instrumentation.
- Message queues (Kafka/SQS) do NOT propagate this automatically — must inject on publish, extract on consume.

## Collector pipeline shape

```yaml
receivers: [otlp, jaeger, zipkin, prometheus, ...]   # data IN
processors: [memory_limiter, batch, tail_sampling, attributes, resource]  # transform
exporters: [otlp, prometheus, logging, ...]           # data OUT

service:
  pipelines:
    traces:  {receivers: [otlp], processors: [batch], exporters: [otlp/jaeger]}
    metrics: {receivers: [otlp], processors: [batch], exporters: [prometheus]}
```

Deployment: **agent** (per-host/sidecar, cheap local work) vs. **gateway** (centralized pool, needed for tail sampling — route by `trace_id` consistent hash so one trace's spans land on one instance).

## Sampling

```
Head sampling  -> decision at trace start, cheap, can't use outcome
                  OTEL_TRACES_SAMPLER=traceidratio
                  OTEL_TRACES_SAMPLER_ARG=0.1        # keep 10%

Tail sampling  -> decision after full trace collected (in Collector)
                  policies: always keep errors, always keep slow (>Xms),
                  probabilistic sample of the rest
```

## Useful env vars

```bash
OTEL_SERVICE_NAME=my-service
OTEL_RESOURCE_ATTRIBUTES=service.version=1.2.0,deployment.environment=prod
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc            # or http/protobuf
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
OTEL_LOGS_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
```

## Jaeger quick start

```bash
docker run -d --name jaeger \
  -p 16686:16686 -p 4317:4317 -p 4318:4318 \
  -e COLLECTOR_OTLP_ENABLED=true \
  jaegertracing/all-in-one:1.57
# UI: http://localhost:16686
```
