# Day 92 — Lab: Loki & Log Aggregation

**Goal:** Deploy a full Loki logging stack on Kubernetes, ship real container logs via Fluent Bit, and query them in Grafana with LogQL — including structured-log parsing and a deliberate cardinality mistake so you feel the failure mode, not just read about it.

**Prerequisites:** A Kubernetes cluster (reuse the one from Days 89–91), `kubectl`, `helm`, and the Grafana instance from Day 90's kube-prometheus-stack (Loki will be added as a new data source to it).

---

### Lab 1 — Deploy Loki (single-binary mode for the lab)

1. Install Loki via Helm in simple/single-binary mode (fine for a lab; production would use the distributed/microservices mode):
   ```bash
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update
   helm install loki grafana/loki -n observability --create-namespace \
     --set loki.commonConfig.replication_factor=1 \
     --set loki.storage.type=filesystem \
     --set singleBinary.replicas=1
   ```
2. Confirm it's running:
   ```bash
   kubectl get pods -n observability -l app.kubernetes.io/name=loki
   ```
3. Add Loki as a Grafana data source (point at `http://loki.observability.svc.cluster.local:3100`), via the UI or a provisioning ConfigMap (Day 90 pattern).

**Success criteria:** The Loki data source in Grafana shows "green"/connected when tested.

---

### Lab 2 — Core hands-on activity: ship container logs via Fluent Bit

This is the assigned hands-on activity for today.

1. Install Fluent Bit configured to tail container logs and ship to Loki:
   ```bash
   helm repo add fluent https://fluent.github.io/helm-charts
   helm install fluent-bit fluent/fluent-bit -n observability \
     --set config.outputs='[OUTPUT]\n    Name loki\n    Match *\n    Host loki.observability.svc.cluster.local\n    Port 3100\n    Labels job=fluentbit'
   ```
   (For real config control, prefer a values file with the full `[INPUT]`/`[FILTER]`/`[OUTPUT]` pipeline from this day's third README.)
2. Confirm Fluent Bit is running as a DaemonSet (one pod per node):
   ```bash
   kubectl get pods -n observability -l app.kubernetes.io/name=fluent-bit -o wide
   ```
3. Generate some log activity — restart a pod or deploy a small app that logs to stdout — then in Grafana Explore (Loki data source), run:
   ```logql
   {namespace="observability"}
   ```
   Confirm real log lines appear.

**Success criteria:** You can see live container logs from your cluster flowing into Loki and appearing in Grafana Explore within a few seconds of being written.

---

### Lab 3 — LogQL practice: filters, parsing, and metric queries

1. Filter to only error-looking lines:
   ```logql
   {namespace="observability"} |= "error"
   ```
2. Deploy (or reuse) an app that logs JSON, then parse and filter on a field:
   ```logql
   {app="my-app"} | json | level="error"
   ```
3. Turn the log stream into a rate metric and graph it as a Grafana panel:
   ```logql
   sum(rate({namespace="observability"} |= "error" [5m]))
   ```
4. If your test app logs a numeric duration field in JSON, compute a p99 directly from logs:
   ```logql
   quantile_over_time(0.99, {app="my-app"} | json | unwrap duration [5m])
   ```

**Success criteria:** You have at least one Grafana panel driven by a LogQL metric query (not just raw log lines), and can explain why it needed a range vector (`[5m]`) the same way a PromQL `rate()` query does.

---

### Lab 4 — Deliberately create and observe a Loki cardinality problem

1. Configure a test log stream (via a small script or a Fluent Bit/Promtail relabel rule) that promotes a random UUID into an actual Loki **label** (not just JSON body content) for every log line — e.g., attach `request_id=<uuid>` as a label rather than JSON content.
2. Generate a burst of a few hundred log lines, each with a new UUID label.
3. Observe in Loki's metrics (`loki_ingester_streams_created_total` if exposed, or simply the growing list of distinct streams in the Grafana Explore label browser) that stream count exploded.
4. Fix it: change the pipeline so the UUID stays in the JSON body instead of becoming a label, and confirm the label list stays flat while the field is still queryable via `| json | request_id="<uuid>"`.

**Success criteria:** You've directly observed a Loki stream-count explosion caused by a high-cardinality label, then fixed it by moving the field into log body content instead — and can explain the difference in one sentence.

---

### Cleanup

```bash
helm uninstall loki fluent-bit -n observability
kubectl delete namespace observability
```

### Stretch challenge

Configure a Loki `derivedFields` entry on the Grafana Loki data source that turns a `trace_id` field in your test app's JSON logs into a clickable link (even if you don't have a real Tempo instance, point the URL template at a dummy URL and confirm the link renders correctly with the extracted value substituted in).
