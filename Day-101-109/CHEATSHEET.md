# Day 101-109 — Cheatsheet: Observability Capstone + AWS SAA-C03

## Helm install commands — the full stack

```bash
kubectl create namespace observability

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics + Operator
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --set grafana.adminPassword='changeme' \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.enableRemoteWriteReceiver=true

# Loki (logs) + Promtail (shipper DaemonSet)
helm install loki grafana/loki-stack \
  --namespace observability \
  --set grafana.enabled=false \
  --set promtail.enabled=true

# Tempo (traces)
helm install tempo grafana/tempo \
  --namespace observability

# Upgrade any of the above after editing values
helm upgrade <release> <chart> -n observability -f values.yaml

# Inspect / uninstall
helm list -n observability
helm uninstall <release> -n observability
```

## PromQL — core patterns

```promql
# Request rate over 5m window (per-second average)
rate(http_requests_total[5m])

# Sum across all pods/instances of a service
sum(rate(http_requests_total{job="checkout-service"}[5m]))

# Error rate as a ratio (availability SLI)
sum(rate(http_requests_total{job="checkout-service", code=~"5.."}[5m]))
  /
sum(rate(http_requests_total{job="checkout-service"}[5m]))

# Latency SLI via histogram — quantile approach
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{job="checkout-service"}[5m])) by (le)
)

# Latency SLI as a ratio (preferred for burn-rate math — "% of requests under threshold")
sum(rate(http_request_duration_seconds_bucket{job="checkout-service", le="0.3"}[5m]))
  /
sum(rate(http_request_duration_seconds_count{job="checkout-service"}[5m]))

# CPU/memory (causes, not SLIs, but useful for diagnosis)
sum(rate(container_cpu_usage_seconds_total{namespace="prod"}[5m])) by (pod)
sum(container_memory_working_set_bytes{namespace="prod"}) by (pod)
```

## Burn-rate alerting — formula and threshold table

```
burn_rate = error_rate / (1 - SLO)
```

Standard 4-window setup for a 99.9% SLO (error budget = 0.001):

| Severity | Long window | Multiplier | Short window | Approx. budget burned if sustained |
|---|---|---|---|---|
| Page  | 1h  | 14.4x | 5m  | 2% of 30-day budget in 1h |
| Page  | 6h  | 6x    | 30m | 5% of 30-day budget in 6h |
| Ticket| 24h | 3x    | 2h  | 10% of 30-day budget in 1 day |
| Ticket| 3d  | 1x    | 6h  | 10% of 30-day budget in 3 days |

```yaml
- alert: ErrorBudgetBurnFast
  expr: |
    slo:error_ratio:rate1h > (14.4 * 0.001)
    and
    slo:error_ratio:rate5m > (14.4 * 0.001)
  labels: { severity: critical }

- alert: ErrorBudgetBurnSlow
  expr: |
    slo:error_ratio:rate24h > (3 * 0.001)
    and
    slo:error_ratio:rate2h > (3 * 0.001)
  labels: { severity: warning }
```

## SLO / error-budget formula reference

```
SLI            = good_events / valid_events   (e.g., non-5xx requests / total requests)
SLO            = target for the SLI over a window (e.g., 99.9% over 30 days)
Error budget   = 1 - SLO                       (e.g., 0.1% = 0.001)
Budget spent   = errors_observed / total_requests   (over the same window)
Budget remaining (%) = 1 - (budget_spent / error_budget)
Burn rate      = error_rate / error_budget
```

## LogQL basics (Loki)

```logql
# All log lines from a label set (namespace + app)
{namespace="prod", app="checkout-service"}

# Filter by substring (like grep)
{app="checkout-service"} |= "ERROR"

# Negative filter
{app="checkout-service"} != "healthcheck"

# Regex filter
{app="checkout-service"} |~ "timeout|deadline exceeded"

# Parse JSON log lines and filter on a parsed field
{app="checkout-service"} | json | status_code >= 500

# Parse logfmt lines
{app="checkout-service"} | logfmt | level="error"

# Rate of matching log lines (metric query from logs)
sum(rate({app="checkout-service"} |= "ERROR" [5m]))

# Extract trace_id from log body (for a Grafana derived field / manual correlation)
{app="checkout-service"} | regexp `trace_id=(?P<trace_id>\w+)`
```

## TraceQL / Tempo query basics

```traceql
# All traces for a service
{ resource.service.name = "checkout-service" }

# Traces with an error status
{ resource.service.name = "checkout-service" && status = error }

# Traces slower than 300ms
{ resource.service.name = "checkout-service" && duration > 300ms }

# Traces touching a specific span name (e.g., a DB call)
{ name = "SELECT orders" && duration > 100ms }

# Combine service + attribute filter
{ resource.service.name = "checkout-service" && span.http.status_code = 500 }
```
Direct trace lookup by ID (most common real-world path via exemplars/derived fields): paste the trace ID straight into Tempo's search box — no query language needed.

## PagerDuty + Alertmanager wiring

```yaml
route:
  routes:
    - matchers: [ severity = critical ]
      receiver: pagerduty-critical
receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: "<integration-key>"
        severity: "critical"
        description: '{{ .CommonAnnotations.summary }}'
```

## AWS SAA-C03 exam-day logistics quick reference

```
Format        : 65 questions (multiple-choice + multiple-response)
Time          : 130 minutes (~2 min/question average, uneven difficulty)
Score         : scaled 100-1000, pass at 720
Guessing      : NO penalty — always answer every question
Check-in      : arrive/log in 30 min early (remote-proctored)
Room          : fully cleared desk, 360° webcam scan, no phone/notes
ID            : two forms of government-issued ID, name must match registration
Domain weights: Resilient ~30% | High-Performing ~28% | Secure ~24% | Cost ~18%
After exam    : pass/fail shown immediately; retake wait ~14 days if failed
Recert        : every 3 years
```

## Domain weight quick reference

| # | Domain | Weight |
|---|---|---|
| 1 | Design Resilient Architectures | ~30% |
| 2 | Design High-Performing Architectures | ~28% |
| 3 | Design Secure Applications and Architectures | ~24% |
| 4 | Design Cost-Optimized Architectures | ~18% |
