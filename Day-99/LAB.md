# Day 99 — Lab: AWS-native Observability

**Goal:** Turn on AWS-native observability for a real EKS cluster, write real Logs Insights queries against it, and understand — hands-on, not just conceptually — the trade-off between self-managed and managed Prometheus/Grafana.

**Prerequisites:** An AWS account with an existing EKS cluster (or willingness to spin up a small one — `eksctl create cluster` or reuse one from an earlier day), AWS CLI v2 configured, `kubectl` pointed at the cluster, and IAM permissions for CloudWatch, X-Ray, AMP, and AMG (or an admin role for lab purposes).

---

### Lab 1 — Enable Container Insights on EKS

1. Check whether the add-on is already installed:
   ```bash
   aws eks describe-addon --cluster-name my-cluster --addon-name amazon-cloudwatch-observability 2>&1 || echo "not installed"
   ```
2. Install it:
   ```bash
   aws eks create-addon \
     --cluster-name my-cluster \
     --addon-name amazon-cloudwatch-observability \
     --resolve-conflicts OVERWRITE
   ```
3. Confirm the DaemonSets landed in the `amazon-cloudwatch` namespace:
   ```bash
   kubectl get pods -n amazon-cloudwatch
   ```
4. In the CloudWatch console, open **Container Insights → Resources**, select your cluster, and drill from cluster → node → pod. Identify the CPU/memory of your busiest pod.
5. Generate some load against a workload in the cluster (`kubectl run -it --rm loadgen --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://my-service; done"`) and watch the Container Insights dashboard update within a couple of minutes.

**Success criteria:** You can navigate the Container Insights dashboard from cluster level down to an individual container's CPU/memory graph, and you've watched a metric move in response to load you generated yourself.

---

### Lab 2 — Write real CloudWatch Logs Insights queries

1. Find a log group with real traffic (your app's log group, or `/aws/containerinsights/my-cluster/application` created by Lab 1):
   ```bash
   aws logs describe-log-groups --query "logGroups[].logGroupName" --output table
   ```
2. Run a basic query in the Logs Insights console (or via CLI) for the last 30 minutes:
   ```bash
   aws logs start-query \
     --log-group-name "/aws/containerinsights/my-cluster/application" \
     --start-time $(date -u -v-30M +%s 2>/dev/null || date -u -d '30 minutes ago' +%s) \
     --end-time $(date -u +%s) \
     --query-string 'fields @timestamp, @message | filter @message like /error/i | sort @timestamp desc | limit 20'
   ```
   Then fetch results with `aws logs get-query-results --query-id <id-from-above>`.
3. Write a `stats ... by bin()` query that counts log volume per minute, grouped by pod name (or another label present in your logs).
4. Save your best query as a **Saved query** in the console under a name like `error-rate-last-hour`.

**Success criteria:** You've run at least one `filter` query and one `stats ... by bin()` aggregation query against real log data, and saved one query for reuse.

---

### Lab 3 — Compare CloudWatch Logs Insights query cost sensitivity to time range

1. Run the same `filter` query from Lab 2 against a 1-hour window, then a 24-hour window, then (if you have retention) a 7-day window.
2. Note the "records scanned" and "records matched" figures shown by the console (or in the CLI response's `statistics` block) for each run.
3. Write down, in your own words, why cost/latency tracked with the *scanned* size and not with how selective the `filter` clause was.

**Success criteria:** You can explain concretely (with your own numbers from this lab) why narrowing the time range before adding filters is the primary lever for Logs Insights cost and speed.

---

### Lab 4 — Stand up an AMP workspace and remote_write into it

1. Create an Amazon Managed Prometheus workspace:
   ```bash
   aws amp create-workspace --alias my-lab-workspace
   ```
2. Note the `workspaceId` and construct the remote_write URL: `https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspaceId>/api/v1/remote_write`.
3. If you have a Prometheus instance running in the cluster (from an earlier day's lab), add a `remote_write` block pointing at that URL with `sigv4` auth, and restart/reload Prometheus.
4. Verify data is arriving by querying AMP directly:
   ```bash
   aws amp query-metrics --workspace-id <workspaceId> --query 'up' 2>/dev/null || \
     echo "Use awscurl or the console Query editor for AMP — the AWS CLI has limited direct query support"
   ```
5. Create an Amazon Managed Grafana workspace (console is faster for the SSO wiring step than CLI for a first attempt), add AMP as a data source, and build one panel showing a metric from your cluster.

**Success criteria:** You have a working remote_write pipeline from a Prometheus instance (or the ADOT Collector) into AMP, and one Grafana panel in AMG rendering real data from it.

---

### Cleanup

```bash
# Remove the Container Insights add-on
aws eks delete-addon --cluster-name my-cluster --addon-name amazon-cloudwatch-observability

# Delete the AMP workspace (irreversible — confirm workspaceId first)
aws amp delete-workspace --workspace-id <workspaceId>

# Delete the AMG workspace via console (Grafana workspace deletion isn't a one-liner CLI call worth scripting for a lab)
# Delete the loadgen pod if it's still running
kubectl delete pod loadgen --ignore-not-found
```

### Stretch challenge

Write a single CloudWatch Logs Insights query that computes p50, p90, and p99 request latency per HTTP route, bucketed into 5-minute windows, from JSON-structured application logs — then save it and wire a CloudWatch dashboard widget to render it as a line graph.
