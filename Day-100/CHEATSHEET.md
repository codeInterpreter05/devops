# Day 100 — Cheatsheet: Datadog / New Relic

## Datadog Agent install (Kubernetes, via Helm)

```bash
helm repo add datadog https://helm.datadoghq.com
helm repo update

helm install datadog-agent datadog/datadog \
  --set datadog.apiKey=<API_KEY> \
  --set datadog.site='datadoghq.com' \
  --set clusterAgent.enabled=true \
  --set datadog.logs.enabled=true \
  --set datadog.apm.portEnabled=true

kubectl get daemonset,deployment -l app.kubernetes.io/name=datadog
helm uninstall datadog-agent
```

## Autodiscovery pod annotations

```yaml
metadata:
  annotations:
    ad.datadoghq.com/<container-name>.check_names: '["postgres"]'
    ad.datadoghq.com/<container-name>.init_configs: '[{}]'
    ad.datadoghq.com/<container-name>.instances: '[{"host": "%%host%%", "port": "5432"}]'
```

## DogStatsD (custom metrics from app code)

```
# StatsD-compatible protocol, sent to the local Agent (default UDP 8125)
myapp.request.count:1|c
myapp.request.duration:120|ms|#route:checkout,env:prod
```

## dd-trace / APM env vars (pod spec)

```yaml
env:
  - name: DD_AGENT_HOST
    valueFrom: { fieldRef: { fieldPath: status.hostIP } }
  - name: DD_TRACE_ENABLED
    value: "true"
  - name: DD_ENV
    value: "prod"
  - name: DD_SERVICE
    value: "checkout"
```

## Datadog metric query syntax

```
avg:trace.http.request.duration{service:checkout} by {env}
sum:system.cpu.user{*}
avg(last_5m):avg:kubernetes.cpu.usage.total{pod_name:my-pod} > 90
```

## Monitor types

```
Metric monitor      -> static threshold on a query
APM monitor          -> threshold on trace-derived metric (error rate, p99 latency)
Anomaly monitor      -> seasonal/trend baseline instead of static threshold
Composite monitor    -> boolean combination of other monitors (reduces false positives)
```

## Log management

```
Indexed logs        -> parsed, tagged, searchable in Log Explorer; counts toward billed retention
Archived logs        -> cheap cold storage (e.g., your own S3), not searchable until rehydrated
Log rehydration       -> pull an archived time range back into the indexed tier on demand
Indexing/exclusion filters -> control WHICH logs get indexed, to bound cost
```

## New Relic (NRQL basics)

```sql
SELECT average(cpuUsedCores) FROM K8sContainerSample WHERE clusterName = 'my-cluster' TIMESERIES
SELECT count(*) FROM Transaction WHERE appName = 'checkout' SINCE 1 hour ago
SELECT percentile(duration, 99) FROM Transaction FACET name
```

```bash
# New Relic Kubernetes integration installer
newrelic install -n kubernetes-open-source-integration
```

## Datadog vs. Prometheus/Grafana vs. New Relic, one line each

```
Datadog:        per-host/per-span/per-GB pricing; native cross-pillar (metrics/traces/logs) correlation; high lock-in; near-zero ops burden.
Prometheus/Grafana: infra-cost pricing; requires deliberate label discipline for cross-pillar correlation; low lock-in (PromQL/OTel portable); you own HA/scaling.
New Relic:      ingest (GB) + per-seat pricing; single NRDB datastore queried via NRQL; generous free tier; similar lock-in profile to Datadog.
```
