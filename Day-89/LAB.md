# Day 89 — Lab: Prometheus Fundamentals

**Goal:** Deploy a real Prometheus + Alertmanager stack on Kubernetes, instrument it against real workload metrics, and write the PromQL queries an SRE actually uses day-to-day (p99 latency, error rate, pod restarts).

**Prerequisites:** A local or cloud Kubernetes cluster (`kind`, `minikube`, or an EKS/GKE cluster), `kubectl`, and `helm` installed. Cluster needs at least 2 vCPU / 4GB RAM free for the stack.

---

### Lab 1 — Deploy kube-prometheus-stack

1. Add the Helm repo and install:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   helm install kps prometheus-community/kube-prometheus-stack \
     --namespace monitoring --create-namespace
   ```
2. Confirm everything is up:
   ```bash
   kubectl get pods -n monitoring
   kubectl get prometheus,alertmanager -n monitoring
   ```
3. Port-forward the Prometheus UI and confirm targets are being scraped:
   ```bash
   kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090:9090
   ```
   Open `http://localhost:9090/targets` — every target should show `UP`. If any show `DOWN`, note the error shown (this is a real debugging skill — don't skip it).

**Success criteria:** `kubectl get pods -n monitoring` shows all pods `Running`, and the Prometheus `/targets` page shows kubelet, node-exporter, kube-state-metrics, and apiserver targets all `UP`.

---

### Lab 2 — Generate real traffic and inspect the data model

1. Deploy a sample app that exposes Prometheus metrics (any app with a `/metrics` endpoint works; a minimal example: deploy `prom/pushgateway` or use `kube-state-metrics` already running in-cluster).
2. In the Prometheus UI, query `kube_pod_status_phase` and inspect the raw labels on the result — note the `pod`, `namespace`, `phase` label set that makes each row a distinct time series.
3. Query `up` and confirm you can see one row per scrape target, each with `job` and `instance` labels.

**Success criteria:** You can explain, using a real query result on screen, why `metric_name{labelA="x"}` and `metric_name{labelA="y"}` are two different time series, not two rows of one series.

---

### Lab 3 — Core hands-on activity: write PromQL for p99 latency, error rate, and pod restarts

This is the assigned hands-on activity for today.

1. **Pod restarts** (a gauge/counter-style metric from kube-state-metrics):
   ```promql
   increase(kube_pod_container_status_restarts_total[1h])
   sum(increase(kube_pod_container_status_restarts_total[1h])) by (namespace, pod)
   ```
2. **Error rate** (requires an app exposing `http_requests_total{status=...}` — use kube-state-metrics' `apiserver_request_total` as a stand-in against the Kubernetes API server itself):
   ```promql
   sum(rate(apiserver_request_total{code=~"5.."}[5m]))
     /
   sum(rate(apiserver_request_total[5m]))
   ```
3. **p99 latency** (using the API server's built-in histogram):
   ```promql
   histogram_quantile(0.99,
     sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb)
   )
   ```
4. Save all three as a **recording rule** group and apply it:
   ```yaml
   # promrules.yaml
   groups:
     - name: sre-basics
       rules:
         - record: cluster:apiserver_requests:error_ratio5m
           expr: sum(rate(apiserver_request_total{code=~"5.."}[5m])) / sum(rate(apiserver_request_total[5m]))
         - record: cluster:apiserver_request_duration_seconds:p99
           expr: histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb))
   ```
   Apply via a `PrometheusRule` CRD (kube-prometheus-stack watches for these):
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: sre-basics
     namespace: monitoring
     labels:
       release: kps
   spec:
     groups:
       - name: sre-basics
         rules:
           - record: cluster:apiserver_requests:error_ratio5m
             expr: sum(rate(apiserver_request_total{code=~"5.."}[5m])) / sum(rate(apiserver_request_total[5m]))
   ```
   ```bash
   kubectl apply -f prometheusrule.yaml
   ```
5. Confirm the recorded metric now exists as its own series: query `cluster:apiserver_requests:error_ratio5m` directly in the Prometheus UI.

**Success criteria:** All three queries return non-empty results, and the recording rule's output metric is queryable by name like any other metric.

---

### Lab 4 — Wire up a real alert through Alertmanager

1. Add an alerting rule to the same `PrometheusRule` CRD:
   ```yaml
       - alert: HighAPIServerErrorRate
         expr: cluster:apiserver_requests:error_ratio5m > 0.02
         for: 5m
         labels:
           severity: warning
         annotations:
           summary: "API server error ratio above 2%"
   ```
2. Apply it, then check the Prometheus UI's `/alerts` page for the rule (it will be `inactive` unless the condition is actually true).
3. Port-forward Alertmanager and confirm it's aware of the rule group:
   ```bash
   kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-alertmanager 9093:9093
   ```
   Open `http://localhost:9093`.
4. Force the alert to fire for testing: temporarily lower the threshold to something you know is currently true (e.g., `> 0`), re-apply, watch it transition `pending` → `firing` in the Prometheus UI, then confirm it shows up in Alertmanager. Revert the threshold afterward.

**Success criteria:** You've watched an alert transition through `inactive` → `pending` → `firing` end-to-end and seen it land in the Alertmanager UI.

---

### Cleanup

```bash
kubectl delete -f prometheusrule.yaml
helm uninstall kps -n monitoring
kubectl delete namespace monitoring
```

### Stretch challenge

Write a single PromQL query that returns, per namespace, the ratio of pods currently in `Running` phase to total pods scheduled — then turn it into a recording rule named `namespace:pods_running:ratio`, and write an alert that fires if that ratio drops below 0.8 for more than 10 minutes.
