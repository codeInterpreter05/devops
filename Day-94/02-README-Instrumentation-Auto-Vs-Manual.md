# Day 94 — OpenTelemetry: Instrumentation (Auto vs. Manual) & the Python SDK

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** Tracing | **Flag:** ⚡ Interview-critical

## Brief

Instrumentation is the act of making your code emit telemetry. OTel gives you two ways to do it — auto-instrumentation (near-zero code changes, instrument libraries you didn't write) and manual instrumentation (explicit spans/attributes you add yourself for business-specific detail). Almost every real service uses **both**: auto-instrumentation for the boring, high-value boilerplate (HTTP frameworks, DB drivers, message queues), manual for anything domain-specific ("did the discount code get applied," "which shard served this request"). Knowing which to reach for — and how they compose — is the practical skill this file builds; it's also the mechanism behind today's hands-on lab (instrumenting a FastAPI service).

## Auto-instrumentation

Auto-instrumentation works via **monkey-patching / bytecode injection**: the OTel language distro ships instrumentation packages for popular libraries (Flask, FastAPI, Django, requests, psycopg2, redis-py, boto3, SQLAlchemy, etc.). When enabled, it wraps the relevant library functions at import/runtime so that calling them automatically starts/ends spans, without you touching the library's call sites in your code.

For Python:

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install   # auto-detects installed libraries, installs matching instrumentors
```

Then run your app through the auto-instrumentation launcher instead of directly:

```bash
opentelemetry-instrument \
  --traces_exporter otlp \
  --metrics_exporter otlp \
  --service_name checkout-service \
  --exporter_otlp_endpoint http://localhost:4317 \
  uvicorn main:app --host 0.0.0.0 --port 8000
```

`opentelemetry-instrument` is a wrapper that sets up the SDK, discovers every installed instrumentor via Python's entry-points mechanism, and patches those libraries **before** your application code runs its own imports. This is why a FastAPI app instrumented this way will already show a span per incoming HTTP request, and a nested span per outgoing `requests`/`httpx` call or SQL query, with zero lines added to `main.py`.

**Why this matters:** auto-instrumentation captures the "plumbing" — HTTP entry/exit points, DB calls, cache calls — which is exactly the stuff that's tedious and error-prone to hand-instrument, and where 80% of latency/error root causes live (a slow DB query, an N+1 pattern, a downstream service timeout).

## Manual instrumentation

Auto-instrumentation can't know your business logic — it doesn't know that "step 3 of checkout is applying a discount code" is meaningful. For that, you instrument explicitly with the SDK's tracer API:

```python
from opentelemetry import trace

tracer = trace.get_tracer("checkout-service")

def apply_discount(order, code):
    with tracer.start_as_current_span("apply_discount") as span:
        span.set_attribute("discount.code", code)
        span.set_attribute("order.id", order.id)
        try:
            discount = discount_lookup(code)
            span.set_attribute("discount.amount", discount.amount)
            return discount
        except InvalidCodeError as e:
            span.record_exception(e)
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            raise
```

Key points about this pattern:
- `start_as_current_span` makes the new span both a child of whatever span is currently active (correct parent/child nesting is automatic via context propagation, covered in file 3) **and** the active span for any nested auto-instrumented calls made inside the `with` block.
- `set_attribute` adds structured, queryable metadata — this is what lets you later ask a backend "show me every trace where `discount.code = SUMMER25` and status = ERROR."
- `record_exception` + `set_status(ERROR)` is the idiomatic way to mark a span as failed **and** attach the exception details as a span event — critical, because a span that silently swallows an exception without marking itself ERROR will look like a fast, successful operation in your trace view, hiding the real failure.

### Custom spans vs. custom attributes — when to use which

- Add a **new span** when you want to measure the *duration* of a distinct logical step (e.g., "cache lookup," "call payment gateway," "render template").
- Add an **attribute** to an existing span when you want to enrich context without adding a timing boundary (e.g., `user.id`, `order.total`, `feature_flag.variant`).
- Avoid span explosion: a span for every single line of business logic makes traces unreadable and expensive to store. Instrument at the granularity you'd actually want to see in a flame graph during an incident.

## Composing auto + manual

Because both mechanisms share the same underlying `TracerProvider` and active-span context, a manual span created inside a request handled by auto-instrumentation nests correctly under the auto-created HTTP span, and any auto-instrumented DB call made inside your manual span nests under *that*. You get one coherent trace tree combining both, with zero extra wiring — this is the payoff of OTel's context-propagation design (see file 3).

## The Python SDK — the pieces you configure explicitly

If you're not using the `opentelemetry-instrument` launcher (e.g., inside a script, a Lambda, or when you want fine control), you wire up the SDK yourself:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

resource = Resource.create({"service.name": "checkout-service", "service.version": "1.4.2"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="localhost:4317", insecure=True)))
trace.set_tracer_provider(provider)
```

- `Resource` — static metadata describing *what* is emitting telemetry (service name, version, environment, host). Every span/metric/log from this process is tagged with it — this is what lets a backend group/filter telemetry by service.
- `BatchSpanProcessor` — buffers finished spans and exports them in batches on a timer/size threshold, instead of one network call per span (which would be prohibitively slow). There's also a `SimpleSpanProcessor` (exports synchronously, one at a time) — useful for debugging/tests, never for production due to the latency/throughput hit.
- `OTLPSpanExporter` — ships batches to a Collector (or directly to a backend) over gRPC/HTTP using the OTLP protocol.

## Points to Remember

- Auto-instrumentation = bytecode/monkey-patch magic for third-party libraries; zero code changes; installed via `opentelemetry-bootstrap` + run via `opentelemetry-instrument`.
- Manual instrumentation = explicit `tracer.start_as_current_span(...)` calls for business logic auto-instrumentation can't see.
- Real services combine both — auto for plumbing (HTTP/DB/cache/queue), manual for domain-specific steps.
- Always mark failed spans with `record_exception` + `set_status(ERROR)` — an unmarked failing span looks identical to a successful one in the UI.
- `BatchSpanProcessor` is the production choice; `SimpleSpanProcessor` is synchronous and only appropriate for local debugging.
- `Resource` attributes (service name/version/environment) are what make multi-service traces filterable/groupable in a backend.

## Common Mistakes

- Forgetting to run the app via `opentelemetry-instrument` (or otherwise initializing the auto-instrumentors before other imports) — auto-instrumentation must patch libraries *before* your app imports and uses them, or the patches silently don't apply.
- Catching an exception, logging it, and continuing — but never calling `span.record_exception`/`set_status(ERROR)` — so the trace shows a "successful," fast span while the real failure is buried in a separate log stream with no trace_id link.
- Using `SimpleSpanProcessor` in production, adding synchronous network I/O to the hot path of every request and tanking throughput under load.
- Over-instrumenting manually — one span per trivial function call — producing traces with hundreds of near-zero-duration spans that are unreadable and expensive to store; instrument at the granularity you'd actually want during an incident.
- Putting high-cardinality data (raw user emails, full request bodies, unbounded IDs) into span **names** instead of attributes — span names should be low-cardinality templates (`GET /orders/:id`, not `GET /orders/48291`), otherwise backends can't aggregate spans of "the same kind" for latency stats.
