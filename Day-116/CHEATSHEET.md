# Day 116 — Cheatsheet: MLOps Fundamentals

## MLflow — tracking

```python
import mlflow

mlflow.set_tracking_uri("http://mlflow:5000")
mlflow.set_experiment("my-experiment")

with mlflow.start_run(run_name="run-1"):
    mlflow.log_param("lr", 0.01)
    mlflow.log_params({"lr": 0.01, "batch_size": 32})   # bulk
    mlflow.log_metric("accuracy", 0.94)
    mlflow.log_metrics({"accuracy": 0.94, "f1": 0.91})
    mlflow.log_artifact("plot.png")
    mlflow.sklearn.log_model(model, "model")            # flavor-specific
    mlflow.pyfunc.log_model(...)                         # generic flavor

mlflow.autolog()   # auto-capture params/metrics for supported frameworks
```

```bash
mlflow server --backend-store-uri postgresql://user:pass@host/db \
  --default-artifact-root s3://bucket/path --host 0.0.0.0 --port 5000
mlflow ui --backend-store-uri sqlite:///mlflow.db   # local-only quick UI
```

## MLflow — model registry

```python
client = mlflow.MlflowClient()

result = mlflow.register_model(model_uri=f"runs:/{run_id}/model", name="my-model")
client.set_registered_model_alias("my-model", "champion", result.version)
client.set_registered_model_alias("my-model", "challenger", other_version)

model = mlflow.pyfunc.load_model("models:/my-model@champion")   # alias-based load
model = mlflow.pyfunc.load_model("models:/my-model/3")           # version-based load

# Classic stage API (older, still widely seen)
client.transition_model_version_stage("my-model", version=3, stage="Production")
```

```bash
mlflow.search_runs(experiment_ids=["1"], order_by=["metrics.f1 DESC"], max_results=5)
```

## DVC

```bash
dvc init
dvc remote add -d storage s3://bucket/dvc-store
dvc add data/train.csv          # track a large file
git add data/train.csv.dvc .gitignore
git commit -m "track dataset v1"
dvc push                         # upload data to remote
dvc pull                         # download data for current git checkout
dvc status                       # what's out of sync between cache/remote/workspace
```

## Feast

```bash
feast init my_feature_repo
feast apply                                    # register entities/feature views
feast materialize-incremental $(date +%F)      # push latest features to online store
feast materialize 2026-01-01 2026-07-01        # backfill a date range
```

```python
from feast import FeatureStore
store = FeatureStore(repo_path=".")

training_df = store.get_historical_features(
    entity_df=entity_df, features=["fv:feature_name"]
).to_df()

online = store.get_online_features(
    features=["fv:feature_name"], entity_rows=[{"user_id": 1}]
).to_dict()
```

## Drift detection (conceptual reference)

```
Data drift (covariate shift)    -> compare INPUT distributions vs training baseline
                                    tools: PSI, KL divergence, KS test, Evidently AI, WhyLabs
Concept drift                    -> INPUT-OUTPUT relationship changes; needs ground-truth labels
                                    often detected late (labels lag)
PSI thresholds (rule of thumb)   -> < 0.1 stable | 0.1-0.25 warning | > 0.25 significant drift
```

## A/B / canary / shadow — quick definitions

```
Champion/Challenger  -> split live traffic %, measure winner statistically before full rollout
Shadow deployment     -> challenger scores real traffic, output NOT returned to user (zero risk,
                         no causal business-outcome measurement possible)
Canary                -> small % of real traffic gets the real challenger response, ramp on success
```
