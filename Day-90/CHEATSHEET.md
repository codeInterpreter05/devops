# Day 90 — Cheatsheet: Grafana Dashboards

## Data source provisioning

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy       # server-side fetch (almost always correct)
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST   # avoid URL length limits on complex queries
      timeInterval: 15s
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      derivedFields:
        - datasourceUid: <tempo-datasource-uid>
          matcherRegex: 'trace_id=(\w+)'
          name: TraceID
          url: '$${__value.raw}'
```

## RED method (request-driven services)

```promql
# Rate
sum(rate(http_requests_total{job="x"}[5m]))
# Errors (ratio)
sum(rate(http_requests_total{job="x",status=~"5.."}[5m])) / sum(rate(http_requests_total{job="x"}[5m]))
# Duration (p99)
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="x"}[5m])) by (le))
```

## USE method (resources)

```promql
# Utilization (CPU)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
# Saturation (load1 as proxy)
node_load1
# Errors
rate(node_network_receive_errs_total[5m])
```

## Variables

```
Type: Query
Query: label_values(kube_pod_info, namespace)     # populate dropdown from label values
Chained: label_values(kube_pod_info{namespace="$namespace"}, pod)
Multi-value query usage: my_metric{namespace=~"$namespace"}   # note =~ not =
Built-ins: $__interval, $__range, $__from, $__to
```

## Dashboard JSON / provisioning

```yaml
apiVersion: 1
providers:
  - name: default
    orgId: 1
    folder: 'SRE Dashboards'
    type: file
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

```bash
# Export dashboard JSON via API
curl -s -H "Authorization: Bearer $GRAFANA_TOKEN" \
  http://localhost:3000/api/dashboards/uid/<uid> | jq '.dashboard' > dashboard.json

# Kubernetes sidecar auto-load pattern (kube-prometheus-stack default)
kubectl create configmap my-dashboard \
  --from-file=my-dashboard.json \
  -n monitoring
kubectl label configmap my-dashboard grafana_dashboard=1 -n monitoring
```

## Grafonnet skeleton

```jsonnet
local g = import 'g.libsonnet';
g.dashboard.new('Checkout Service')
  + g.dashboard.withPanels([
      g.panel.timeSeries.new('Request Rate')
        + g.panel.timeSeries.queryOptions.withTargets([
            g.query.prometheus.new('$datasource', 'sum(rate(http_requests_total[5m]))')
          ]),
    ])
```

## Grafana-managed alerting

```
Query: any data source (Prometheus, Loki, CloudWatch, SQL, ...)
Condition: IS ABOVE / IS BELOW / OUTSIDE RANGE <threshold>
Evaluate every: <interval> for: <duration>
Contact point / Notification policy: routes to Slack/PagerDuty/email,
  or forward to an external Alertmanager (recommended for single source of routing truth)
```

## Grafana access essentials

```bash
kubectl get secret -n monitoring kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d
kubectl port-forward -n monitoring svc/kps-grafana 3000:80
```
