# Day 92 — Cheatsheet: Loki & Log Aggregation

## LogQL stream selectors + line filters

```logql
{namespace="checkout", app="api"}              # stream selector (like PromQL labels)
{app="api"} |= "timeout"                        # line contains string
{app="api"} != "healthcheck"                    # line does NOT contain string
{app="api"} |~ "err(or)?"                        # line matches regex
{app="api"} !~ "debug|trace"                     # line does NOT match regex
```

## LogQL parsers

```logql
{app="api"} | json                                # parse JSON, extract fields
{app="api"} | json | status_code="500"             # filter on parsed field
{app="api"} | logfmt                               # parse key=value logfmt lines
{app="api"} | logfmt | duration > 500ms
{app="api"} | pattern "<ip> - - <_> \"<method>"    # extract via pattern template
{app="api"} | regexp "(?P<level>\\w+):"            # extract via named regex groups
```

## LogQL metric queries

```logql
sum(rate({app="api"} |= "error" [5m]))                          # error lines/sec
count_over_time({app="api"} |= "error" [1h])                      # total error lines in 1h
sum by (status_code) (count_over_time({app="api"} | json [5m]))
quantile_over_time(0.99, {app="api"} | json | unwrap duration [5m])   # p99 from a parsed numeric field
bytes_rate({app="api"}[5m])                                       # log volume rate in bytes/sec
```

## Fluent Bit config skeleton

```ini
[INPUT]
    Name    tail
    Path    /var/log/containers/*.log
    Parser  docker
    Tag     kube.*

[FILTER]
    Name    kubernetes
    Match   kube.*
    Merge_Log On

[OUTPUT]
    Name    loki
    Match   *
    Host    loki.observability.svc.cluster.local
    Labels  job=fluentbit
```

## Promtail config skeleton

```yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
    pipeline_stages:
      - json: {expressions: {level: level}}
      - labels: {level: }
```

## Vector (VRL) config skeleton

```toml
[sources.k8s_logs]
type = "kubernetes_logs"

[transforms.parse]
type = "remap"
inputs = ["k8s_logs"]
source = '. = parse_json!(.message)'

[sinks.loki]
type = "loki"
inputs = ["parse"]
endpoint = "http://loki:3100"
labels.namespace = "{{ kubernetes.pod_namespace }}"
```

## Loki vs ELK/OpenSearch — one-line comparison

```
ELK/OpenSearch : index full text at ingest -> fast arbitrary search, high storage/index cost
Loki           : index only labels -> cheap storage, fast IF query starts from a good label filter
```

## Cardinality discipline (Loki)

```
GOOD labels : namespace, app/job, container, level, environment
NEVER labels: trace_id, request_id, user_id, order_id, raw dynamic path segments
  -> put these in the JSON body, query via | json at query time instead
```

## Helm quick install

```bash
helm install loki grafana/loki -n observability --create-namespace \
  --set loki.storage.type=filesystem --set singleBinary.replicas=1
helm install fluent-bit fluent/fluent-bit -n observability
```
