# Day 95 — Lab: Jaeger & Tempo

**Goal:** Stand up Grafana Tempo, link traces to Loki logs by `trace_id`, and build a unified debugging workflow — plus compare it hands-on against Jaeger.

**Prerequisites:** Docker + Docker Compose, `curl`. Ideally completed Day 94's lab (an OTel-instrumented FastAPI app you can reuse), though this lab's step 2 provides a standalone trace generator if not.

---

### Lab 1 — Run Tempo + Grafana + Loki via Docker Compose

1. Create a working directory and `docker-compose.yml`:
   ```bash
   mkdir -p ~/tempo-lab && cd ~/tempo-lab
   ```
   ```yaml
   # docker-compose.yml
   version: "3"
   services:
     tempo:
       image: grafana/tempo:2.5.0
       command: ["-config.file=/etc/tempo.yaml"]
       volumes:
         - ./tempo.yaml:/etc/tempo.yaml
       ports:
         - "4317:4317"   # OTLP gRPC
         - "3200:3200"   # Tempo query API
     loki:
       image: grafana/loki:2.9.0
       ports:
         - "3100:3100"
     grafana:
       image: grafana/grafana:11.0.0
       environment:
         - GF_AUTH_ANONYMOUS_ENABLED=true
         - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
       ports:
         - "3000:3000"
   ```
2. Create `tempo.yaml`:
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
   metrics_generator:
     registry:
       external_labels:
         source: tempo
   ```
3. `docker compose up -d` and confirm all three containers are running: `docker compose ps`.

**Success criteria:** `docker compose ps` shows `tempo`, `loki`, and `grafana` all "running"; `curl localhost:3200/status` (or `/ready`) responds.

---

### Lab 2 — Send traces into Tempo and view them in Grafana

1. Reuse Day 94's `checkout-service` FastAPI app, pointing its OTLP exporter at `localhost:4317` (this Tempo instance instead of Jaeger). If you didn't do Day 94's lab, generate synthetic traces instead:
   ```bash
   docker run --rm -e OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:4317 \
     ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
     traces --otlp-insecure --traces 20
   ```
2. In Grafana (`http://localhost:3000`), add Tempo as a data source: Connections → Data sources → Add → Tempo → URL `http://tempo:3200` → Save & test.
3. Go to Explore, select the Tempo data source, and search for traces (TraceQL: `{}` to see everything, or `{ status = error }` to filter).
4. Open a trace — confirm the waterfall view renders in Grafana.

**Success criteria:** You can find and open at least one trace in Grafana's Explore view backed by Tempo, and the waterfall renders with correct span nesting/durations.

---

### Lab 3 — Link traces to Loki logs via trace ID

1. Add Loki as a second Grafana data source: URL `http://loki:3100`.
2. Configure the Tempo data source's "Trace to logs" setting (Data source settings → Trace to logs): select Loki, and set the tag mapping to look up `trace_id` (or your log field name).
3. Push a log line into Loki that includes a real `trace_id` from a trace you generated in Lab 2 (using `logcli` or a direct push API call):
   ```bash
   curl -H "Content-Type: application/json" -XPOST "http://localhost:3100/loki/api/v1/push" \
     --data-raw '{"streams": [{"stream": {"service": "checkout-service"}, "values": [["'"$(date +%s%N)"'", "ERROR processing order trace_id=<PASTE_REAL_TRACE_ID>"]]}]}'
   ```
4. Back in Grafana, open the trace from Lab 2 whose ID you used, and click the "Logs for this span" button. Confirm it jumps to (or queries for) the Loki log line you just pushed.

**Success criteria:** Clicking from a Tempo trace successfully surfaces the correlated Loki log line, demonstrating the trace-to-logs workflow without manual timestamp matching.

---

### Lab 4 — Compare against Jaeger side-by-side

1. Start Jaeger all-in-one alongside (different ports to avoid collision):
   ```bash
   docker run -d --name jaeger -p 16687:16686 -p 4318:4317 \
     -e COLLECTOR_OTLP_ENABLED=true jaegertracing/all-in-one:1.57
   ```
2. Send the same synthetic trace batch to both Tempo (`:4317`) and Jaeger (`:4318`) by running `telemetrygen` twice with each endpoint.
3. Compare: open the same logical trace in Jaeger's UI (`:16687`) vs. Grafana/Tempo. Note differences in default UI depth, tag-search experience, and how much setup each needed.

**Success criteria:** You can articulate, from direct hands-on comparison, one concrete reason a team would pick Tempo (cheaper storage, no dedicated DB cluster, native Grafana correlation) vs. one reason to pick Jaeger (mature standalone UI, ad-hoc tag search in Elasticsearch-backed deployments).

---

### Cleanup

```bash
docker compose down -v
docker rm -f jaeger
rm -rf ~/tempo-lab
```

### Stretch challenge

Enable Tempo's `metrics_generator` fully (add `overrides.defaults.metrics_generator.processors: [service-graphs, span-metrics]` to `tempo.yaml` and point `metrics_generator.storage.remote_write` at a running Prometheus), regenerate some traces, then find the auto-generated `traces_spanmetrics_latency` histogram in Prometheus and compute a p99 from it using `histogram_quantile`.
