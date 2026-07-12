# Day 96 — Lab: SLOs, SLAs & Error Budgets

**Goal:** Define real SLOs for a test app, generate multi-window burn-rate alerts with Sloth, and load Prometheus/Alertmanager so you can watch a burn-rate alert actually fire.

**Prerequisites:** Docker + Docker Compose, `curl`. A running Prometheus + Alertmanager (the lab sets these up). `sloth` binary (installed in Lab 2).

---

### Lab 1 — Define SLIs and an SLO for a test app

1. Stand up a simple app that exposes Prometheus metrics with an error-injection knob. If you don't have one handy, run this minimal Flask app behind `prometheus_flask_exporter` (or reuse any service from a previous day that exposes `http_requests_total`/`http_request_duration_seconds`).
2. Write down, in a `slo-notes.md` file, explicit answers to:
   - What is the SLI? (e.g., `sum(rate(http_requests_total{code!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`)
   - What is the SLO target and window? (e.g., 99.5% over a rolling 30 days — pick something realistic for a test app, not 99.99%)
   - Compute the error budget in minutes for your chosen target/window by hand: `window_minutes × (1 - target)`.

**Success criteria:** You have a written SLI query, an SLO target+window, and a hand-calculated error budget in minutes that you can defend/explain out loud.

---

### Lab 2 — Install Sloth and generate burn-rate alert rules

1. Install Sloth:
   ```bash
   docker pull ghcr.io/slok/sloth:v0.11.0
   ```
2. Write `checkout-slo.yaml` using your Lab 1 SLI/SLO:
   ```yaml
   version: "prometheus/v1"
   service: "checkout-api"
   labels:
     owner: "platform-team"
   slos:
     - name: "requests-availability"
       objective: 99.5
       description: "99.5% of requests should not be 5xx"
       sli:
         events:
           error_query: sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[{{.window}}]))
           total_query: sum(rate(http_requests_total{job="checkout-api"}[{{.window}}]))
       alerting:
         name: CheckoutAPIHighErrorRate
         labels:
           category: "availability"
         page_alert:
           labels: {severity: "page"}
         ticket_alert:
           labels: {severity: "ticket"}
   ```
3. Generate the Prometheus rules:
   ```bash
   docker run --rm -v "$(pwd)":/work ghcr.io/slok/sloth:v0.11.0 \
     generate -i /work/checkout-slo.yaml -o /work/checkout-slo-rules.yaml
   ```
4. Open `checkout-slo-rules.yaml` and identify: the recording rules for the SLI ratio, and the two (or more) alert rules with different burn-rate multipliers/windows. Note the `for:` durations and thresholds Sloth chose for you.

**Success criteria:** You have a generated Prometheus rules file with at least a page-severity and a ticket-severity burn-rate alert, and you can explain the multiplier/window pairing on at least one of them.

---

### Lab 3 — Load the rules into Prometheus and Alertmanager

1. Minimal `docker-compose.yml` for Prometheus + Alertmanager:
   ```yaml
   version: "3"
   services:
     prometheus:
       image: prom/prometheus:v2.53.0
       volumes:
         - ./prometheus.yml:/etc/prometheus/prometheus.yml
         - ./checkout-slo-rules.yaml:/etc/prometheus/checkout-slo-rules.yaml
       ports: ["9090:9090"]
     alertmanager:
       image: prom/alertmanager:v0.27.0
       ports: ["9093:9093"]
   ```
2. `prometheus.yml`:
   ```yaml
   global: {scrape_interval: 15s}
   rule_files: ["checkout-slo-rules.yaml"]
   alerting:
     alertmanagers: [{static_configs: [{targets: ["alertmanager:9093"]}]}]
   scrape_configs:
     - job_name: "checkout-api"
       static_configs: [{targets: ["host.docker.internal:5000"]}]  # point at your test app
   ```
3. `docker compose up -d`, then open `http://localhost:9090/rules` and confirm your Sloth-generated rules loaded without errors.

**Success criteria:** Prometheus's `/rules` page shows the SLI recording rule and both burn-rate alert rules as active/loaded (state `inactive` is expected until burn rate actually rises).

---

### Lab 4 — Force a burn-rate alert to fire

1. Modify your test app to return HTTP 500 for a configurable percentage of requests (e.g., an env var `ERROR_RATE_PCT`).
2. Set `ERROR_RATE_PCT` high enough to blow through the fast-burn threshold (e.g., 20% errors), and hammer the endpoint for several minutes:
   ```bash
   for i in $(seq 1 2000); do curl -s "http://localhost:5000/checkout" >/dev/null; done
   ```
3. Watch Prometheus's `/alerts` page — confirm the page-severity burn-rate alert transitions `inactive → pending → firing` once both its short and long windows exceed the multiplier threshold.
4. Check Alertmanager (`http://localhost:9093`) — confirm the firing alert is visible there too.

**Success criteria:** You watch, with your own eyes, a multi-window burn-rate alert transition from inactive to firing as a direct result of injected errors, and can explain why it needed *both* windows to agree before firing.

---

### Cleanup

```bash
docker compose down -v
rm -f checkout-slo-rules.yaml
```

### Stretch challenge

Lower `ERROR_RATE_PCT` to something that would only trip the *slow-burn/ticket* alert (not the fast/page one) — sustain it, and confirm only the ticket-severity alert fires while the page-severity one stays inactive. Explain in your own words why the same absolute error rate can be "fine" or "an emergency" depending purely on which burn-rate window it's measured against.
