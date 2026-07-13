# Day 116 — Lab: MLOps Fundamentals

**Goal:** Stand up an MLflow tracking server on Kubernetes, log real experiments, register models, and compare versions — the assigned hands-on activity for today, broken into concrete steps.

**Prerequisites:** A Kubernetes cluster (kind/minikube is fine), `kubectl` and `helm` configured, `python3` with `pip` locally, and an S3-compatible bucket (real AWS S3, or MinIO run locally for a fully offline lab). Basic familiarity with a Deployment/Service manifest.

---

### Lab 1 — Deploy MLflow tracking server on Kubernetes

1. Deploy Postgres (backend store) and MinIO (artifact store, S3-compatible) if you don't already have S3 access:
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm install mlflow-db bitnami/postgresql -n mlops --create-namespace \
     --set auth.database=mlflow --set auth.username=mlflow --set auth.password=mlflowpass
   helm install minio bitnami/minio -n mlops \
     --set auth.rootUser=minio --set auth.rootPassword=minio123
   ```
2. Deploy the MLflow server itself (a minimal Deployment using the official `ghcr.io/mlflow/mlflow` image):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mlflow-tracking
     namespace: mlops
   spec:
     replicas: 1
     selector: {matchLabels: {app: mlflow-tracking}}
     template:
       metadata: {labels: {app: mlflow-tracking}}
       spec:
         containers:
           - name: mlflow
             image: ghcr.io/mlflow/mlflow:latest
             command: ["mlflow", "server",
               "--backend-store-uri", "postgresql://mlflow:mlflowpass@mlflow-db-postgresql:5432/mlflow",
               "--default-artifact-root", "s3://mlflow-artifacts/",
               "--host", "0.0.0.0", "--port", "5000"]
             env:
               - {name: AWS_ACCESS_KEY_ID, value: "minio"}
               - {name: AWS_SECRET_ACCESS_KEY, value: "minio123"}
               - {name: MLFLOW_S3_ENDPOINT_URL, value: "http://minio:9000"}
             ports: [{containerPort: 5000}]
   ---
   apiVersion: v1
   kind: Service
   metadata: {name: mlflow-tracking, namespace: mlops}
   spec:
     selector: {app: mlflow-tracking}
     ports: [{port: 5000, targetPort: 5000}]
   ```
3. Apply it, create the `mlflow-artifacts` bucket in MinIO, and port-forward:
   ```bash
   kubectl apply -f mlflow-deployment.yaml
   kubectl port-forward -n mlops svc/mlflow-tracking 5000:5000
   ```
4. Open `http://localhost:5000` — confirm the MLflow UI loads with no experiments yet.

**Success criteria:** MLflow UI is reachable, backed by Postgres + MinIO running in-cluster, not SQLite/local disk.

---

### Lab 2 — Log real experiment runs

1. Locally, `pip install mlflow scikit-learn pandas`, then point your client at the in-cluster server:
   ```bash
   export MLFLOW_TRACKING_URI=http://localhost:5000
   export AWS_ACCESS_KEY_ID=minio
   export AWS_SECRET_ACCESS_KEY=minio123
   export MLFLOW_S3_ENDPOINT_URL=http://localhost:9000   # port-forward MinIO too: kubectl port-forward -n mlops svc/minio 9000:9000
   ```
2. Train and log at least 3 runs with different hyperparameters against a toy dataset:
   ```python
   import mlflow, mlflow.sklearn
   from sklearn.datasets import load_breast_cancer
   from sklearn.ensemble import RandomForestClassifier
   from sklearn.model_selection import train_test_split
   from sklearn.metrics import accuracy_score, f1_score

   X, y = load_breast_cancer(return_X_y=True)
   X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

   mlflow.set_experiment("breast-cancer-classifier")
   for n_estimators in [50, 100, 200]:
       with mlflow.start_run(run_name=f"rf-{n_estimators}"):
           mlflow.log_param("n_estimators", n_estimators)
           model = RandomForestClassifier(n_estimators=n_estimators, random_state=42).fit(X_train, y_train)
           preds = model.predict(X_test)
           mlflow.log_metric("accuracy", accuracy_score(y_test, preds))
           mlflow.log_metric("f1", f1_score(y_test, preds))
           mlflow.sklearn.log_model(model, "model")
   ```
3. In the UI, compare all 3 runs side by side and identify the best one by `f1`.

**Success criteria:** 3 runs visible in the MLflow UI with distinct params/metrics, and you can identify the best run using the UI's comparison view (not just by re-reading your script's output).

---

### Lab 3 — The core hands-on activity: register and promote a model

1. Register the best run's model:
   ```python
   best_run_id = "<copy from UI>"
   result = mlflow.register_model(model_uri=f"runs:/{best_run_id}/model", name="breast-cancer-classifier")
   ```
2. Promote it via alias:
   ```python
   client = mlflow.MlflowClient()
   client.set_registered_model_alias("breast-cancer-classifier", "champion", result.version)
   ```
3. Train and register a second, different-hyperparameter model version, then compare both versions' metrics via the Registry UI.
4. Load the `champion` alias directly (not a hardcoded run ID) and run a prediction, proving the alias resolves to the correct version:
   ```python
   model = mlflow.pyfunc.load_model("models:/breast-cancer-classifier@champion")
   print(model.predict(X_test[:5]))
   ```

**Success criteria:** Two registered model versions exist, `champion` alias points to the better one, and you've loaded/predicted using the alias reference rather than a run ID.

---

### Cleanup

```bash
kubectl delete namespace mlops
```

### Stretch challenge

Add `mlflow.log_param("git_commit", ...)` sourced from a real `git rev-parse HEAD` in your training script, and manually simulate a DVC-style dataset version string logged alongside it. Then write one sentence explaining exactly what you would need in hand (which three artifacts) to reproduce today's `champion` run bit-for-bit six months from now.
