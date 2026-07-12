# Day 89 — Cheatsheet: Prometheus Fundamentals

## Metric types

```
counter    only goes up (or resets to 0 on restart) — always wrap in rate()/increase()
gauge      goes up or down — query directly
histogram  cumulative _bucket{le=...} + _sum + _count — aggregatable across instances
summary    client-side quantiles + _sum + _count — NOT aggregatable across instances
```

## PromQL essentials

```promql
# Rate / increase
rate(http_requests_total[5m])                 # per-second avg rate, handles counter resets
increase(http_requests_total[1h])              # total increase over window = rate * seconds
irate(http_requests_total[5m])                 # instant rate (last 2 points) — spiky, use for graphs not alerts

# Histograms
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le))   # p99, keep `le` in `by`

# Aggregation
sum(x) by (label)      avg(x) by (label)      max(x) by (label)      min(x) by (label)
count(x)                topk(5, x)             bottomk(5, x)

# Classic RED-method error ratio
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# Vector matching / joins
metric_a / on(instance) metric_b
metric_a * on(instance) group_left(extra_label) metric_b

# Time
rate(x[5m]) offset 1d       # compare to yesterday
predict_linear(x[1h], 4*3600)   # linear-extrapolate 4h into the future (disk-fill alerts)
delta(gauge_metric[1h])     # gauge change over window
deriv(gauge_metric[1h])     # per-second derivative of a gauge

# Target health
up                              # 1 = scrape succeeded, 0 = failed
count(up == 0) by (job)          # how many targets down, per job
```

## Scrape config skeleton

```yaml
scrape_configs:
  - job_name: 'my-job'
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /metrics
    static_configs:
      - targets: ['host:port']
        labels: {env: prod}

  - job_name: 'k8s-pods'
    kubernetes_sd_configs: [{role: pod}]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'unwanted_metric.*'
        action: drop
```

`relabel_configs` = pre-scrape (target selection/address). `metric_relabel_configs` = post-scrape (sample filtering).

## Recording rule + alerting rule

```yaml
groups:
  - name: example
    interval: 30s
    rules:
      - record: job:errors:ratio5m
        expr: sum(rate(errors_total[5m])) by (job) / sum(rate(requests_total[5m])) by (job)

      - alert: HighErrorRate
        expr: job:errors:ratio5m > 0.05
        for: 10m
        labels: {severity: critical}
        annotations:
          summary: "Error rate above 5% for {{ $labels.job }}"
```

## Alertmanager routing skeleton

```yaml
route:
  receiver: default
  group_by: [alertname, team]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match: {severity: critical}
      receiver: pagerduty
      continue: true
    - match: {team: checkout}
      receiver: checkout-slack

receivers:
  - name: pagerduty
    pagerduty_configs: [{routing_key: '<key>'}]
  - name: checkout-slack
    slack_configs: [{api_url: '<webhook>', channel: '#checkout-alerts', send_resolved: true}]

inhibit_rules:
  - source_matchers: ['alertname="NodeDown"']
    target_matchers: ['severity="warning"']
    equal: ['node']
```

## kube-prometheus-stack quick commands

```bash
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl get pods -n monitoring
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090:9090
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-alertmanager 9093:9093
kubectl apply -f my-prometheusrule.yaml     # add custom recording/alerting rules via CRD
```
