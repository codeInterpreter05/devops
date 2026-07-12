# Day 101-109 — Lab: Observability Capstone Build-Out

**Goal:** Leave this 9-day block with a real, working observability stack on Kubernetes (Prometheus + Grafana + Loki + Tempo), a sample app instrumented with OpenTelemetry emitting all three signal types with correlated trace IDs, a working SLO with multi-window burn-rate alerts wired to PagerDuty, a runbook, and — in parallel — a sat AWS SAA-C03 exam.

**Prerequisites:** A Kubernetes cluster (kind/minikube locally, or a real cluster — this lab assumes ~4+ CPUs and 8GB+ RAM available to the cluster), `kubectl`, `helm` v3, a free PagerDuty developer/trial account, and your AWS SAA-C03 practice-exam results from Day 81-88's kickoff.

This lab is a single 9-day arc — do the parts roughly in order across the block; Part 5 (AWS exam) runs in parallel with Parts 1-4 rather than strictly after them.

---

### Part 1 (Days 1-2) — Deploy the stack

1. Create a namespace and add the Helm repos:
   ```bash
   kubectl create namespace observability

   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update
   ```
2. Deploy **kube-prometheus-stack** (Prometheus Operator + Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics, all in one chart):
   ```bash
   helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
     --namespace observability \
     --set grafana.adminPassword='changeme' \
     --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
     --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false
   ```
   The two `SelectorNilUsesHelmValues=false` flags matter: without them, Prometheus only picks up `ServiceMonitor`/`PodMonitor` resources labeled to match this specific Helm release, which will silently ignore the sample app's ServiceMonitor in Part 2.
3. Deploy **Loki** (log storage) with **Promtail** (log shipper DaemonSet):
   ```bash
   helm install loki grafana/loki-stack \
     --namespace observability \
     --set grafana.enabled=false \
     --set promtail.enabled=true
   ```
4. Deploy **Tempo** (trace storage):
   ```bash
   helm install tempo grafana/tempo \
     --namespace observability \
     --set tempo.receivers.otlp.protocols.grpc.endpoint=0.0.0.0:4317 \
     --set tempo.receivers.otlp.protocols.http.endpoint=0.0.0.0:4318
   ```
5. Wire Loki and Tempo into Grafana as data sources (Grafana came bundled with kube-prometheus-stack). Create a ConfigMap so Grafana auto-provisions them instead of clicking through the UI:
   ```yaml
   # datasources.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: grafana-extra-datasources
     namespace: observability
     labels:
       grafana_datasource: "1"
   data:
     datasources.yaml: |
       apiVersion: 1
       datasources:
         - name: Loki
           type: loki
           url: http://loki:3100
           jsonData:
             derivedFields:
               - datasourceUid: tempo
                 matcherRegex: 'trace_id=(\w+)'
                 name: TraceID
                 url: '$${__value.raw}'
         - name: Tempo
           uid: tempo
           type: tempo
           url: http://tempo:3100
           jsonData:
             tracesToLogsV2:
               datasourceUid: loki
             tracesToMetrics:
               datasourceUid: prometheus
   ```
   ```bash
   kubectl apply -f datasources.yaml
   kubectl rollout restart deployment kube-prometheus-stack-grafana -n observability
   ```
6. Port-forward and log in:
   ```bash
   kubectl port-forward -n observability svc/kube-prometheus-stack-grafana 3000:80
   # open http://localhost:3000, user: admin, password: changeme
   ```

**Success criteria:** Grafana loads, and under Connections → Data sources you see Prometheus, Loki, and Tempo all reporting a successful health check. `kubectl get pods -n observability` shows Prometheus, Alertmanager, Grafana, Loki, Promtail (one per node), and Tempo pods all `Running`.

---

### Part 2 (Days 3-4) — Instrument a sample app with OpenTelemetry

1. Deploy the OpenTelemetry Collector as the ingestion/fan-out point:
   ```yaml
   # otel-collector-config.yaml
   receivers:
     otlp:
       protocols:
         grpc:
           endpoint: 0.0.0.0:4317
         http:
           endpoint: 0.0.0.0:4318
   processors:
     batch: {}
   exporters:
     prometheusremotewrite:
       endpoint: "http://kube-prometheus-stack-prometheus:9090/api/v1/write"
     loki:
       endpoint: "http://loki:3100/loki/api/v1/push"
     otlp/tempo:
       endpoint: "tempo:4317"
       tls:
         insecure: true
   service:
     pipelines:
       metrics:
         receivers: [otlp]
         processors: [batch]
         exporters: [prometheusremotewrite]
       logs:
         receivers: [otlp]
         processors: [batch]
         exporters: [loki]
       traces:
         receivers: [otlp]
         processors: [batch]
         exporters: [otlp/tempo]
   ```
   Enable remote-write receiving on Prometheus (`--set prometheus.prometheusSpec.enableRemoteWriteReceiver=true` on the Part 1 Helm install, or `helm upgrade` with that flag added) so the Collector's `prometheusremotewrite` exporter has somewhere to push to.
2. Deploy a sample instrumented app — any small HTTP service with an OpenTelemetry auto-instrumentation agent attached (e.g., a Python Flask app with `opentelemetry-instrument`, or a Node Express app with `@opentelemetry/auto-instrumentations-node`). Point its OTLP exporter at the Collector:
   ```bash
   OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.observability.svc:4318
   OTEL_SERVICE_NAME=checkout-service
   OTEL_RESOURCE_ATTRIBUTES=deployment.environment=lab
   ```
3. Add a `ServiceMonitor` (if the app also exposes a native `/metrics` endpoint alongside OTLP) so Prometheus scrapes it directly:
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: checkout-service
     namespace: observability
     labels:
       release: kube-prometheus-stack
   spec:
     selector:
       matchLabels:
         app: checkout-service
     endpoints:
       - port: http-metrics
         path: /metrics
   ```
4. Generate load against the app (`hey`, `k6`, or even a `while true; do curl ...; done` loop) and confirm in Grafana Explore:
   - **Prometheus**: `checkout-service` request-count and latency metrics are visible.
   - **Loki**: log lines from the app's pod show up, and the derived field links a `trace_id=` mention to a Tempo trace.
   - **Tempo**: a trace search by service name (`checkout-service`) returns real spans, and clicking a span jumps to correlated logs.

**Success criteria:** You can start from a single Tempo trace, click through to the exact log lines for that trace, and separately jump from a Prometheus latency graph's exemplar dot to that same trace — full round-trip correlation working, not just three data sources sitting side by side.

---

### Part 3 (Days 5-6) — Define SLOs and burn-rate alerts

1. Decide the SLOs for `checkout-service`: 99.9% availability (non-5xx), 95% of requests under 300ms, both over a rolling 30-day window (see 02-README for the reasoning).
2. Write Prometheus **recording rules** for the SLI (pre-aggregating keeps burn-rate alert queries fast and cheap):
   ```yaml
   # slo-recording-rules.yaml
   groups:
     - name: checkout-service-slo
       rules:
         - record: slo:requests:rate5m
           expr: sum(rate(http_requests_total{job="checkout-service"}[5m]))
         - record: slo:errors:rate5m
           expr: sum(rate(http_requests_total{job="checkout-service", code=~"5.."}[5m]))
         - record: slo:error_ratio:rate5m
           expr: slo:errors:rate5m / slo:requests:rate5m
   ```
3. Write the **multi-window, multi-burn-rate alerting rules** (see CHEATSHEET.md for the burn-rate-multiplier table this derives from):
   ```yaml
   # slo-burn-rate-alerts.yaml
   groups:
     - name: checkout-service-burn-rate
       rules:
         - alert: CheckoutServiceErrorBudgetBurnFast
           expr: |
             slo:error_ratio:rate1h > (14.4 * 0.001)
             and
             slo:error_ratio:rate5m > (14.4 * 0.001)
           labels:
             severity: critical
           annotations:
             summary: "checkout-service burning error budget fast (14.4x)"
             runbook_url: "https://runbooks.internal/checkout-service-high-burn"
         - alert: CheckoutServiceErrorBudgetBurnSlow
           expr: |
             slo:error_ratio:rate24h > (3 * 0.001)
             and
             slo:error_ratio:rate2h > (3 * 0.001)
           labels:
             severity: warning
           annotations:
             summary: "checkout-service burning error budget (3x, ticket-level)"
             runbook_url: "https://runbooks.internal/checkout-service-high-burn"
   ```
   (`0.001` is the error budget for a 99.9% SLO — `1 - 0.999`. You'll need matching `slo:error_ratio:rate1h`, `rate24h`, `rate2h` recording rules alongside the `rate5m` one above — add them following the same pattern with different range vectors.)
4. Apply both as a `PrometheusRule` custom resource (the Operator picks these up automatically):
   ```bash
   kubectl apply -f slo-recording-rules.yaml
   kubectl apply -f slo-burn-rate-alerts.yaml
   ```
5. Build a Grafana SLO dashboard panel: a single-stat panel for "error budget remaining" (`1 - (slo:error_ratio:rate30d / 0.001)` expressed as a percentage) plus a time-series panel of burn rate against the 14.4x/6x/3x/1x threshold lines.

**Success criteria:** `kubectl get prometheusrules -n observability` shows your rules; in Prometheus's UI (Alerts tab), both burn-rate alerts appear in `inactive` state; forcing a synthetic error spike (have your load generator return 500s deliberately for a few minutes) flips the fast-burn alert to `firing`, and it clears back to `inactive` once the error spike stops and the short window rolls past it.

---

### Part 4 (Days 7-8) — PagerDuty integration and runbook

1. In PagerDuty, create a new **Service** (e.g., "Checkout Service") with an **Events API v2** integration, and copy the generated integration/routing key.
2. Configure Alertmanager to route `severity: critical` alerts to PagerDuty:
   ```yaml
   # alertmanager-config.yaml (merge into the existing Alertmanager config)
   route:
     receiver: default
     routes:
       - matchers:
           - severity = critical
         receiver: pagerduty-critical
   receivers:
     - name: default
     - name: pagerduty-critical
       pagerduty_configs:
         - routing_key: "<your-integration-key>"
           severity: "critical"
           description: '{{ .CommonAnnotations.summary }}'
   ```
   Apply this via the `AlertmanagerConfig` CRD (Operator-managed) or by updating the `alertmanager.yaml` secret directly, depending on how kube-prometheus-stack was installed.
3. Re-trigger the synthetic error spike from Part 3 and confirm a real PagerDuty incident is created and (if you've set up a phone/push-enabled on-call schedule) actually pages you.
4. Write the runbook referenced by `runbook_url` in the alert annotation. Use this template:
   ```markdown
   # Runbook: checkout-service high error-budget burn

   ## Symptom
   Alertmanager fired `CheckoutServiceErrorBudgetBurnFast` or `...BurnSlow`.

   ## Impact
   Customers are seeing failed checkouts at a rate exceeding the SLO's allowed budget.

   ## Immediate mitigation (try these first, in order)
   1. Check recent deploys: `kubectl rollout history deployment/checkout-service -n prod`
   2. If a deploy in the last hour correlates with the spike, roll it back:
      `kubectl rollout undo deployment/checkout-service -n prod`
   3. If not deploy-related, check pod health: `kubectl get pods -n prod -l app=checkout-service`
   4. If pods are crash-looping, check logs: `kubectl logs -n prod -l app=checkout-service --tail=100`

   ## Diagnosis (once mitigated / if mitigation doesn't resolve it)
   - Open the Grafana SLO dashboard, jump to a failing trace via exemplar, follow to correlated logs.
   - Check downstream dependency health (payment-gateway, inventory-service).

   ## Escalation
   - Secondary on-call: #platform-oncall in PagerDuty
   - Service owner: @checkout-team
   ```

**Success criteria:** A synthetic incident produces a real PagerDuty page end to end (Prometheus alert → Alertmanager → PagerDuty → phone/push notification), and the runbook is reachable via the `runbook_url` link and contains copy-pasteable commands, not prose-only instructions.

---

### Part 5 (Days throughout the block, in parallel) — AWS SAA-C03 exam week

1. Schedule your Pearson VUE exam slot now if you haven't already, based on your Day 81-88 baseline and any subsequent practice-exam scores.
2. Run the final-week checklist from [03-README-AWS-SAA-C03-Exam-Day.md](03-README-AWS-SAA-C03-Exam-Day.md): one more timed full practice exam, targeted re-study of your single weakest domain, a fast pass over the Well-Architected pillars.
3. Complete the exam-day logistics prep (system check, ID, clear desk) the day before your slot, not the morning of.
4. **Sit the exam.**
5. Regardless of outcome, write down: your score (or pass/fail + domain breakdown), your single biggest content gap if any, and whether your practice-exam scores predicted your real result — this becomes input to Phase 4 if AWS content resurfaces there.

**Success criteria:** The exam is sat (not just scheduled), and you have a concrete record of the outcome and domain breakdown, pass or fail.

---

## Cleanup

```bash
helm uninstall tempo -n observability
helm uninstall loki -n observability
helm uninstall kube-prometheus-stack -n observability
kubectl delete configmap grafana-extra-datasources -n observability
kubectl delete namespace observability
```
If running in a cloud-provisioned cluster (not local kind/minikube), also confirm any LoadBalancer/PersistentVolume resources created by the charts were actually torn down — `kubectl get pvc,svc -n observability` before deleting the namespace, since some cloud PVs/LBs can outlive the namespace deletion and keep billing.

## Ready for Phase 4 checklist

- [ ] Prometheus, Grafana, Loki, and Tempo are all deployed on Kubernetes and Grafana shows all three as healthy data sources.
- [ ] A sample app instrumented with OpenTelemetry emits metrics, logs, and traces that share a trace ID, and I can navigate metric → trace → log → back in Grafana without manually copying IDs between tabs.
- [ ] I can state, from memory, the difference between SLI/SLO/SLA and explain error budget and burn rate with the actual formulas, not just the vocabulary.
- [ ] I have a working multi-window, multi-burn-rate PromQL alert that I've watched fire and clear against a synthetic error spike.
- [ ] I have a real PagerDuty integration that pages on a critical alert, with an escalation policy configured (even if it's just me as both primary and secondary for this lab).
- [ ] I have a runbook linked from the alert itself, structured mitigation-first, with exact commands.
- [ ] I have sat the AWS SAA-C03 exam and recorded the outcome, pass or fail, with a domain-level breakdown to act on if needed.

If more than one or two boxes are unchecked, that's the specific gap to close before treating Phase 3 as done — better to find it here than assumed-away going into Phase 4.
