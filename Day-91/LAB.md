# Day 91 — Lab: Thanos & Long-term Metrics

**Goal:** Deploy Thanos Sidecar with S3-compatible object storage, query historical data beyond Prometheus's local retention window, and directly observe a cardinality problem so it stops being an abstract warning.

**Prerequisites:** A Kubernetes cluster with Prometheus running (reuse Day 89's kube-prometheus-stack), `kubectl`, `helm`, and either real AWS S3 access or a local MinIO instance (recommended for a free lab — no cloud cost).

---

### Lab 1 — Stand up MinIO as a free local S3-compatible bucket

1. Deploy MinIO for local object storage:
   ```bash
   helm repo add minio https://charts.min.io/
   helm install minio minio/minio \
     --namespace observability --create-namespace \
     --set rootUser=admin,rootPassword=minio12345 \
     --set persistence.size=5Gi
   ```
2. Port-forward the MinIO console and create a bucket named `thanos`:
   ```bash
   kubectl port-forward -n observability svc/minio-console 9001:9001
   ```
   Log in and create the `thanos` bucket via the UI, or via `mc` CLI.

**Success criteria:** A bucket named `thanos` exists and is reachable from inside the cluster at `minio.observability.svc.cluster.local:9000`.

---

### Lab 2 — Core hands-on activity: deploy Thanos Sidecar + S3 storage

This is the assigned hands-on activity for today.

1. Create the object storage config secret Thanos Sidecar needs:
   ```yaml
   # objstore.yml
   type: S3
   config:
     bucket: thanos
     endpoint: minio.observability.svc.cluster.local:9000
     access_key: admin
     secret_key: minio12345
     insecure: true
   ```
   ```bash
   kubectl create secret generic thanos-objstore-config \
     --from-file=objstore.yml=objstore.yml -n monitoring
   ```
2. If using kube-prometheus-stack, enable the Thanos Sidecar via Helm values (or edit the `Prometheus` CRD directly):
   ```yaml
   # values-thanos.yaml
   prometheus:
     prometheusSpec:
       thanos:
         image: quay.io/thanos/thanos:v0.34.0
       externalLabels:
         cluster: lab-cluster
         replica: "0"
       objectStorageConfigFile: /etc/thanos/objstore.yml
   ```
   ```bash
   helm upgrade kps prometheus-community/kube-prometheus-stack \
     -n monitoring -f values-thanos.yaml
   ```
3. Confirm the Sidecar container is running alongside Prometheus:
   ```bash
   kubectl get pod -n monitoring -l app.kubernetes.io/name=prometheus -o jsonpath='{.items[0].spec.containers[*].name}'
   ```
   You should see a `thanos-sidecar` container listed.
4. Deploy Thanos Query, pointed at the Sidecar's gRPC StoreAPI:
   ```bash
   helm install thanos-query bitnami/thanos --set query.enabled=true \
     --set storegateway.enabled=false --set compactor.enabled=false \
     --set bucketweb.enabled=false --namespace monitoring \
     --set query.stores[0]="kps-kube-prometheus-stack-prometheus:10901"
   ```

**Success criteria:** Thanos Query's UI (`kubectl port-forward` to its service, port 10902 or similar) shows your Prometheus instance listed as a connected Store, and a PromQL query run through Thanos Query returns the same data as querying Prometheus directly.

---

### Lab 3 — Deploy Store Gateway and Compactor, query historical data

1. Deploy Store Gateway and Compactor pointed at the same MinIO bucket (via the bitnami/thanos chart or standalone manifests), using the same `objstore.yml` secret.
2. Wait for at least one Prometheus block (2 hours of data, or force it by lowering `--storage.tsdb.min-block-duration` in a test environment) to be uploaded by the Sidecar to the bucket — confirm via the MinIO console that objects appear under `thanos/01H.../`.
3. Query Thanos Query for a time range you know is now served only by object storage (older than the Sidecar's local retention) and confirm data returns — this proves Store Gateway, not the live Prometheus, served the result.

**Success criteria:** You can query a time range through Thanos Query and confirm (via the Store Gateway's logs or the Thanos UI's "stores" panel) that the data came from object storage, not from live Prometheus.

---

### Lab 4 — Deliberately create and observe a cardinality problem

1. Deploy a small test app (or use `pushgateway`) that exposes a counter with a label carrying a random UUID per request, e.g. `test_requests_total{request_id="<uuid>"}`.
2. Generate a loop of a few thousand fake "requests," each with a new UUID label value:
   ```bash
   for i in $(seq 1 5000); do
     curl -s --data-binary "test_requests_total{request_id=\"$(uuidgen)\"} 1" \
       http://localhost:9091/metrics/job/cardinality-test
   done
   ```
3. Query `prometheus_tsdb_head_series` before and after — observe it jump by roughly 5,000.
4. Query `count by (__name__)({__name__=~"test_requests.*"})` to see the exploded series count for this one metric.

**Success criteria:** You've watched `prometheus_tsdb_head_series` increase in near-real-time as a direct result of a single bad instrumentation decision, and can explain why in one sentence.

---

### Cleanup

```bash
helm uninstall thanos-query thanos-storegateway thanos-compactor -n monitoring
helm uninstall minio -n observability
kubectl delete secret thanos-objstore-config -n monitoring
kubectl delete namespace observability
helm upgrade kps prometheus-community/kube-prometheus-stack -n monitoring --reuse-values \
  --set prometheus.prometheusSpec.thanos=null
```

### Stretch challenge

Configure Thanos Compactor's downsampling and confirm (by querying a wide time range and inspecting the Thanos Query UI's resolution indicator, or Prometheus's own debug endpoints) that a year-scale query actually uses the 1-hour downsampled data rather than scanning raw samples.
