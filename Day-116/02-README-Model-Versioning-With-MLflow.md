# Day 116 — MLOps Fundamentals: Model Versioning with MLflow

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** MLOps | **Flag:** ⚡ Interview-critical

## Brief

If Day 116's first note establishes *why* reproducibility is hard in ML, this note covers the tool that operationalizes it: **MLflow**, the de facto standard for experiment tracking and model registry, plus **DVC** for the dataset-versioning half of the problem MLflow doesn't solve on its own. Today's hands-on activity — standing up an MLflow tracking server on Kubernetes — is exactly the kind of task you'd be handed in a real MLOps role within the first month, so understanding its architecture (not just its Python API) matters.

## MLflow's four components

MLflow is often described as "one tool" but is really four loosely-coupled components you can adopt independently:

1. **Tracking** — logs parameters, metrics, and artifacts for each training run. This is what people mean by "experiment tracking."
2. **Projects** — a packaging format for reproducible runs (a `MLproject` file describing entry points and dependencies) — less universally adopted than Tracking/Registry.
3. **Models** — a standard packaging format for models (`MLmodel` file + flavors like `sklearn`, `pytorch`, `pyfunc`) so a model can be loaded/served consistently regardless of what library trained it.
4. **Model Registry** — a versioned, stage-aware store for registered models (`Staging`, `Production`, `Archived` stages, or newer alias-based tagging), sitting on top of Tracking.

## Tracking: what actually gets logged and where

```python
import mlflow

mlflow.set_tracking_uri("http://mlflow-tracking.mlops.svc.cluster.local:5000")
mlflow.set_experiment("fraud-detection")

with mlflow.start_run(run_name="xgboost-v3"):
    mlflow.log_param("max_depth", 6)
    mlflow.log_param("learning_rate", 0.1)
    mlflow.log_param("git_commit", get_git_sha())        # pin code version explicitly
    mlflow.log_param("dataset_version", "dvc:v2.3.0")     # pin data version explicitly (see DVC below)

    model = train_xgboost(X_train, y_train, max_depth=6, learning_rate=0.1)

    preds = model.predict(X_test)
    mlflow.log_metric("auc", roc_auc_score(y_test, preds))
    mlflow.log_metric("f1", f1_score(y_test, preds))

    mlflow.xgboost.log_model(model, artifact_path="model")
    mlflow.log_artifact("feature_importance.png")
```

Two backends matter architecturally: the **backend store** (a database — Postgres/MySQL in production, never SQLite for a shared team server) holds params/metrics/metadata; the **artifact store** (S3, GCS, Azure Blob) holds the actual model files, plots, and other binary artifacts. Confusing these — e.g., pointing the artifact store at the same small disk as the tracking server — is the most common reason a self-hosted MLflow server falls over once models start including multi-GB weight files.

**Why logging `git_commit` and `dataset_version` explicitly matters**: MLflow's Tracking UI shows you *metrics*, but it has no built-in knowledge of your git history or your dataset's version lineage unless you tell it. Treating these as just two more logged parameters — not a separate, informal "I'll remember which commit this was" — is what actually makes a run reproducible months later.

## Model Registry: the promotion workflow

```python
# Register a model version from a completed run
result = mlflow.register_model(
    model_uri=f"runs:/{run_id}/model",
    name="fraud-detector"
)

# Promote via alias (modern MLflow) or stage (classic API)
client = mlflow.MlflowClient()
client.set_registered_model_alias("fraud-detector", "champion", result.version)
```

The Registry is what turns "a pile of experiment runs" into "an answerable question: what's running in production right now, and what ran before it." A serving system (Day 117's KServe/Seldon) pulls the model tagged `champion` (or in `Production` stage) rather than a hardcoded run ID — so promoting a new model version is a metadata operation (repoint the alias), not a redeploy of serving infrastructure.

```bash
# Compare two registered versions' metrics side by side (via UI or API)
mlflow.search_runs(experiment_ids=["1"], order_by=["metrics.auc DESC"])
```

## DVC: versioning the data MLflow doesn't

MLflow tracks *runs* and *models* well, but it isn't built to version large datasets efficiently — that's **DVC (Data Version Control)**'s job. DVC works like Git for large files: it stores lightweight pointer files in your git repo (`.dvc` files, containing a content hash) while the actual data lives in remote storage (S3, GCS, etc.), so your git history stays small while your dataset history stays fully versioned and reproducible.

```bash
dvc init
dvc remote add -d storage s3://my-bucket/dvc-store
dvc add data/training_set.csv         # creates training_set.csv.dvc, stages the real file in DVC's cache
git add data/training_set.csv.dvc .gitignore
git commit -m "Track training set v1 with DVC"
dvc push                              # uploads actual data to the S3 remote

# Later, after the dataset changes:
dvc add data/training_set.csv
git commit -m "Update training set to v2"
dvc push

# Reproduce an exact past training run's data:
git checkout <commit-from-that-run>
dvc pull                              # pulls the exact data version referenced by that commit
```

This is the mechanism behind logging `dataset_version: dvc:v2.3.0` in the MLflow example above — a DVC-tracked commit tag gives you an exact, retrievable snapshot of the training data, which combined with the git commit SHA and MLflow's logged hyperparameters is what makes a training run *actually* reproducible, not just documented.

## Points to Remember

- MLflow Tracking, Models, and Registry are separate but complementary — Tracking logs runs, Models standardizes the packaging format, Registry adds versioned promotion workflow (`Staging`/`Production`/aliases) on top.
- Use a real database (Postgres) as the backend store and object storage (S3/GCS) as the artifact store for any shared/production MLflow server — SQLite and local disk don't scale past a single user's laptop.
- The Registry's alias/stage mechanism is what lets serving infrastructure stay pointed at "whatever is `champion`/`Production`" instead of a hardcoded run ID — promotion becomes a metadata change, not a redeploy.
- MLflow does not version large datasets efficiently on its own — DVC (or an equivalent dataset-versioning tool) fills that specific gap, using Git-like semantics with data stored in a separate remote.
- True reproducibility requires three things logged together: the exact code (git SHA), the exact data (DVC-tracked commit), and the exact hyperparameters/seed (MLflow logged params) — any one alone is insufficient.

## Common Mistakes

- Running a shared-team MLflow server backed by SQLite and local-disk artifact storage, then being surprised when it corrupts or fills up disk once multiple people log large model artifacts concurrently.
- Logging metrics and hyperparameters but not the git commit SHA or dataset version — leaving a run "trackable" in MLflow's UI but not actually reproducible months later.
- Hardcoding a specific MLflow `run_id` in serving/deployment config instead of referencing the Registry's `champion`/`Production` alias — turns every model promotion into a deployment config change instead of a metadata update.
- Committing raw large data files directly into git "since DVC seems like overkill for now" — bloats the git repo permanently (git doesn't forget large blobs even after they're removed from HEAD) and gives up the ability to cheaply version large datasets.
- Treating MLflow's autologging (`mlflow.autolog()`) as sufficient without checking what it actually captures for your specific framework — autologging coverage varies by library and can silently miss custom metrics or preprocessing steps outside the framework's standard `fit`/`predict` calls.
