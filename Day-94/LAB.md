# Day 94 — Lab: OpenTelemetry

**Goal:** Instrument a real Python FastAPI service with the OTel SDK, view the traces in Jaeger, add custom spans for business logic, and observe context propagation across a service-to-service call.

**Prerequisites:** Docker + Docker Compose, Python 3.10+, `pip`. Basic familiarity with FastAPI and `uvicorn`.

---

### Lab 1 — Stand up Jaeger locally

1. Run Jaeger all-in-one, with OTLP ingestion enabled:
   ```bash
   docker run -d --name jaeger \
     -p 16686:16686 \
     -p 4317:4317 \
     -p 4318:4318 \
     -e COLLECTOR_OTLP_ENABLED=true \
     jaegertracing/all-in-one:1.57
   ```
2. Open the Jaeger UI at `http://localhost:16686`. Confirm it loads (no traces yet).

**Success criteria:** Jaeger UI loads at `:16686`, and `docker ps` shows the container healthy with ports `4317`/`4318` (OTLP gRPC/HTTP) exposed.

---

### Lab 2 — Auto-instrument a FastAPI service

1. Create a project folder and a minimal app:
   ```bash
   mkdir -p ~/otel-lab && cd ~/otel-lab
   python3 -m venv venv && source venv/bin/activate
   pip install fastapi uvicorn requests \
     opentelemetry-distro opentelemetry-exporter-otlp
   opentelemetry-bootstrap -a install
   ```
2. Write `main.py`:
   ```python
   from fastapi import FastAPI
   import time, random

   app = FastAPI()

   @app.get("/checkout")
   def checkout():
       time.sleep(random.uniform(0.05, 0.3))
       return {"status": "ok", "order_id": random.randint(1000, 9999)}
   ```
3. Run it through the auto-instrumentation launcher:
   ```bash
   OTEL_SERVICE_NAME=checkout-service \
   OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
   OTEL_TRACES_EXPORTER=otlp \
   OTEL_METRICS_EXPORTER=none \
   opentelemetry-instrument uvicorn main:app --port 8000
   ```
4. Hit it a few times: `for i in {1..10}; do curl -s localhost:8000/checkout; done`
5. In Jaeger UI, select service `checkout-service`, click **Find Traces**. Open one — confirm you see a span for the `GET /checkout` request with timing.

**Success criteria:** You see `checkout-service` in Jaeger's service dropdown, and clicking a trace shows a span with realistic duration matching the `time.sleep` jitter.

---

### Lab 3 — Add a manual custom span for business logic

1. Modify `main.py` to add a manual span around a fake "discount" step:
   ```python
   from fastapi import FastAPI
   from opentelemetry import trace
   import time, random

   app = FastAPI()
   tracer = trace.get_tracer("checkout-service")

   def apply_discount(code: str):
       with tracer.start_as_current_span("apply_discount") as span:
           span.set_attribute("discount.code", code)
           time.sleep(0.05)
           if code == "INVALID":
               span.set_status(trace.Status(trace.StatusCode.ERROR, "unknown code"))
               raise ValueError("invalid discount code")
           span.set_attribute("discount.amount", 10)

   @app.get("/checkout")
   def checkout(code: str = "SAVE10"):
       time.sleep(random.uniform(0.05, 0.3))
       apply_discount(code)
       return {"status": "ok", "order_id": random.randint(1000, 9999)}
   ```
2. Restart the app (same launch command as Lab 2).
3. Call it twice: `curl localhost:8000/checkout?code=SAVE10` and `curl localhost:8000/checkout?code=INVALID`.
4. In Jaeger, open both new traces. Confirm the successful one shows a nested `apply_discount` child span with attribute `discount.code=SAVE10`, and the failing one shows the same span marked as an **error** (red) with the exception recorded.

**Success criteria:** You can point at the trace waterfall and explain, from the UI alone, which specific step failed and why — without reading any logs.

---

### Lab 4 — Observe context propagation across two services

1. Create a second service, `payment_service.py`, on port 8001:
   ```python
   from fastapi import FastAPI
   app = FastAPI()

   @app.post("/charge")
   def charge():
       return {"charged": True}
   ```
   Run it the same way, with `OTEL_SERVICE_NAME=payment-service`.
2. Modify `checkout-service`'s `apply_discount` caller to call `payment-service`:
   ```python
   import requests

   @app.get("/checkout")
   def checkout(code: str = "SAVE10"):
       apply_discount(code)
       requests.post("http://localhost:8001/charge")
       return {"status": "ok"}
   ```
3. `requests` is auto-instrumented, so it should automatically inject a `traceparent` header. Verify by adding a temporary log/print in `payment_service.py` of `request.headers.get("traceparent")` (or capture the request with `curl -v` against the running services to see headers manually if you prefer).
4. Call `/checkout` again, then find the trace in Jaeger — confirm it now shows spans from **both** `checkout-service` and `payment-service` under one trace tree.

**Success criteria:** One Jaeger trace contains spans from two different services, connected by a shared `trace_id`, proving `traceparent` propagated correctly over the HTTP call.

---

### Cleanup

```bash
deactivate
docker rm -f jaeger
rm -rf ~/otel-lab
```

### Stretch challenge

Change the `tail_sampling`-equivalent behavior manually: set `OTEL_TRACES_SAMPLER=traceidratio` and `OTEL_TRACES_SAMPLER_ARG=0.5` as env vars on `checkout-service`, hit `/checkout` 20 times, and count how many actually appear in Jaeger. Explain in one sentence why this is head-based (not tail-based) sampling, and what you'd lose if the one request that errored happened to be sampled out.
