# Day 99 — Cheatsheet: AWS-native Observability

## Container Insights (EKS)

```bash
# Install/upgrade the managed add-on (modern path)
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability \
  --resolve-conflicts OVERWRITE

aws eks describe-addon --cluster-name my-cluster --addon-name amazon-cloudwatch-observability
aws eks delete-addon --cluster-name my-cluster --addon-name amazon-cloudwatch-observability

# Check the DaemonSets it deploys
kubectl get pods -n amazon-cloudwatch
kubectl get ds -n amazon-cloudwatch
```

## CloudWatch Logs Insights query language

```
fields @timestamp, @message           # project fields
filter @message like /ERROR/          # WHERE-equivalent
filter level = "ERROR" and status >= 500
sort @timestamp desc                  # order
limit 20                              # cap rows returned

parse @message "user=* status=*" as user, status   # extract fields from unstructured text

stats count() as errors by bin(1m)                  # count per minute
stats avg(latency_ms), pct(latency_ms, 99) by route # avg + p99 grouped by field
stats count(*) by ispresent(error_code)             # existence-based grouping
```

```bash
# Run a query from the CLI
aws logs start-query \
  --log-group-name "/my/log/group" \
  --start-time $(date -u -v-1H +%s) \
  --end-time $(date -u +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /error/i | limit 20'

aws logs get-query-results --query-id <id>
```

## X-Ray

```bash
# X-Ray daemon listens on UDP 2000 locally; app SDK sends segments here
# Trace ID header format: X-Amzn-Trace-Id: Root=1-...;Parent=...;Sampled=1

aws xray get-service-graph --start-time $(date -u -v-1H +%s) --end-time $(date -u +%s)
aws xray get-trace-summaries --start-time $(date -u -v-1H +%s) --end-time $(date -u +%s)
aws xray batch-get-traces --trace-ids <trace-id-1> <trace-id-2>

# Default sampling: 1 req/sec + 5% of additional requests, per service
```

## ADOT Collector (OpenTelemetry)

```yaml
receivers:
  otlp:
    protocols: { grpc: {}, http: {} }
exporters:
  awsxray:
    region: us-east-1
  awsemf:
    region: us-east-1
    namespace: MyApp
service:
  pipelines:
    traces:  { receivers: [otlp], exporters: [awsxray] }
    metrics: { receivers: [otlp], exporters: [awsemf] }
```

```bash
# OTLP ports the SDK sends to (collector receiver defaults)
# gRPC: 4317   HTTP: 4318
```

## CloudWatch Synthetics

```bash
aws synthetics create-canary \
  --name my-heartbeat \
  --code S3Bucket=my-bucket,S3Key=canary.zip,Handler=index.handler \
  --artifact-s3-location s3://my-bucket/canary-artifacts \
  --execution-role-arn arn:aws:iam::123456789012:role/CanaryRole \
  --schedule Expression="rate(5 minutes)" \
  --runtime-version syn-nodejs-puppeteer-6.2

aws synthetics start-canary --name my-heartbeat
aws synthetics get-canary-runs --name my-heartbeat
aws synthetics stop-canary --name my-heartbeat
aws synthetics delete-canary --name my-heartbeat
```

## Amazon Managed Prometheus (AMP)

```bash
aws amp create-workspace --alias my-workspace
aws amp list-workspaces
aws amp describe-workspace --workspace-id <id>
aws amp delete-workspace --workspace-id <id>
```

```yaml
# prometheus.yml
remote_write:
  - url: https://aps-workspaces.us-east-1.amazonaws.com/workspaces/<id>/api/v1/remote_write
    sigv4:
      region: us-east-1
```

## Amazon Managed Grafana (AMG)

```bash
aws grafana create-workspace \
  --account-access-type CURRENT_ACCOUNT \
  --authentication-providers AWS_SSO \
  --permission-type SERVICE_MANAGED \
  --workspace-name my-grafana

aws grafana list-workspaces
aws grafana delete-workspace --workspace-id <id>
```

## Managed vs. self-managed decision, one line each

```
Self-managed Prometheus/Grafana: cheaper at steady high scale, full ecosystem control, you own HA/upgrades/retention (Thanos/Cortex).
AMP + AMG: pay-per-sample/per-user, AWS owns HA/scaling/upgrades, PromQL-compatible, faster time-to-value, less on-call surface.
```
