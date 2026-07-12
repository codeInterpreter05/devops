# Day 90 — Lab: Grafana Dashboards

**Goal:** Build a real Grafana dashboard for a Kubernetes cluster covering node utilization, pod restarts, and request latency — using variables, provisioning as code, and cross-source correlation — not just click around the UI once and forget it.

**Prerequisites:** The kube-prometheus-stack from Day 89's lab still deployed (it ships Grafana by default), `kubectl`, and (optional but recommended) a Loki stack — you can also do steps involving Loki after Day 92 and revisit this lab.

---

### Lab 1 — Access Grafana and inspect the default data source

1. Get the auto-generated admin password and port-forward:
   ```bash
   kubectl get secret -n monitoring kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d
   kubectl port-forward -n monitoring svc/kps-grafana 3000:80
   ```
2. Log in at `http://localhost:3000` (user `admin`).
3. Go to **Connections → Data sources** and open the pre-provisioned Prometheus data source. Note it was set up via provisioning, not manually — find the `access` mode and `httpMethod` and explain why each is set the way it is.

**Success criteria:** You can log in, find the Prometheus data source, and explain in one sentence why it uses `access: proxy`.

---

### Lab 2 — Core hands-on activity: build the EKS/cluster dashboard

This is the assigned hands-on activity for today: build a dashboard for node utilization, pod restarts, and request latency.

1. Create a new dashboard, add a **node utilization** row using node-exporter metrics (already scraped by kube-prometheus-stack):
   ```promql
   # CPU utilization per node
   100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
   # Memory utilization per node
   (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
   ```
2. Add a **pod restarts** panel:
   ```promql
   sum(increase(kube_pod_container_status_restarts_total[1h])) by (namespace, pod)
   ```
   Set the panel visualization to a table, sorted descending — this is the practical "which pod is crash-looping" view an SRE actually uses.
3. Add a **request latency (p99)** panel using the API server as a stand-in for a real app:
   ```promql
   histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb))
   ```
4. Arrange panels so the RED/USE summary (utilization + restarts + latency) sits in the top row, no scrolling required.

**Success criteria:** A single dashboard, saved, showing all three panel types with real live data from your cluster.

---

### Lab 3 — Add variables and templating

1. Create a `namespace` variable: Type = Query, data source = Prometheus, query `label_values(kube_pod_info, namespace)`.
2. Create a chained `pod` variable filtered by the selected namespace: `label_values(kube_pod_info{namespace="$namespace"}, pod)`.
3. Update the pod-restarts panel query to use `$namespace`:
   ```promql
   sum(increase(kube_pod_container_status_restarts_total{namespace="$namespace"}[1h])) by (pod)
   ```
4. Enable "Multi-value" on the namespace variable, select two namespaces at once, and confirm Grafana rewrote your query's operator to `=~` under the hood (check the panel's inspect/query view).

**Success criteria:** Switching the namespace dropdown re-renders the pod-restarts panel without editing any query, and the pod dropdown only shows pods from the currently selected namespace(s).

---

### Lab 4 — Provision the dashboard as code

1. Export the dashboard: **Dashboard settings → JSON Model**, copy the JSON to a file `cluster-overview-dashboard.json`.
2. Wrap it in a Kubernetes ConfigMap and label it for the Grafana sidecar to auto-load (kube-prometheus-stack's Grafana ships with the dashboard sidecar enabled by default):
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: cluster-overview-dashboard
     namespace: monitoring
     labels:
       grafana_dashboard: "1"
   data:
     cluster-overview.json: |
       <paste JSON here>
   ```
3. Apply it: `kubectl apply -f cluster-overview-dashboard.yaml`.
4. Delete the dashboard from the Grafana UI, then confirm it reappears automatically within `updateIntervalSeconds` — proving the ConfigMap, not the UI, is now the source of truth.

**Success criteria:** Deleting the dashboard in the UI and having it reappear automatically proves provisioning (not manual clicks) now owns this dashboard.

---

### Cleanup

```bash
kubectl delete configmap cluster-overview-dashboard -n monitoring
# leave kube-prometheus-stack running if you plan to reuse it in later labs (Day 91+)
```

### Stretch challenge

Build a `grafonnet` (Jsonnet) source file that generates the same RED-method row (rate/errors/duration panels) as a reusable function, then instantiate it twice for two different `job` label values in one dashboard — proving the panel definition is shared, not copy-pasted.
