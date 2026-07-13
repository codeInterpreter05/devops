# Day 117 — Model Serving & Kubeflow: Kubeflow Pipelines & Argo Workflows

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** MLOps | **Flag:** ⚡ Interview-critical

## Brief

Serving a model (previous file) is the last step of the ML pipeline named on Day 116 — but the data → train → evaluate → deploy sequence itself needs to be orchestrated as a repeatable, versioned, parameterizable workflow, not a notebook someone runs by hand. **Argo Workflows** is the general-purpose Kubernetes-native workflow engine; **Kubeflow Pipelines** is built on top of it, purpose-built for ML with an SDK that lowers the bar for data scientists (who aren't necessarily Kubernetes experts) to define these pipelines themselves.

## Argo Workflows: the substrate

Argo Workflows is a Kubernetes-native workflow engine where each step in a workflow (a "template") runs as one or more pods, and workflows are defined as a DAG (Directed Acyclic Graph) or sequential steps via a CRD.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: ml-pipeline-
spec:
  entrypoint: ml-pipeline
  templates:
    - name: ml-pipeline
      dag:
        tasks:
          - name: validate-data
            template: validate-data-step
          - name: train
            template: train-step
            dependencies: [validate-data]
          - name: evaluate
            template: evaluate-step
            dependencies: [train]
          - name: deploy
            template: deploy-step
            dependencies: [evaluate]
            when: "{{tasks.evaluate.outputs.parameters.auc}} > 0.85"   # conditional deploy gate
    - name: train-step
      container:
        image: my-registry/train:v3
        command: ["python", "train.py"]
```

The `when` conditional gate on the `deploy` task is the key operational pattern: a pipeline should never deploy a model unconditionally just because training completed without error — it should gate on the **evaluate** step's actual metric output, so a model that trained successfully but scored below an acceptable threshold never reaches production automatically.

Argo Workflows is intentionally general-purpose — it's equally used for CI/CD pipelines, ETL, and infrastructure automation, not just ML. This is why it's the substrate Kubeflow Pipelines builds on rather than reinventing.

## Kubeflow Pipelines: the ML-specific layer

Kubeflow Pipelines (KFP) provides a Python SDK for defining pipelines as code, which compiles down to an Argo Workflow (or, in KFP v2, its own more portable IR YAML that can run on multiple backends) — abstracting away raw YAML DAG authoring for data scientists.

```python
from kfp import dsl, compiler

@dsl.component(base_image="python:3.11", packages_to_install=["pandas", "scikit-learn"])
def train_model(data_path: str, n_estimators: int) -> str:
    import pandas as pd
    from sklearn.ensemble import RandomForestClassifier
    import joblib
    df = pd.read_parquet(data_path)
    model = RandomForestClassifier(n_estimators=n_estimators).fit(df.drop("label", axis=1), df["label"])
    joblib.dump(model, "/tmp/model.joblib")
    return "/tmp/model.joblib"

@dsl.component(base_image="python:3.11")
def evaluate_model(model_path: str, test_data_path: str) -> float:
    # ... load model, score against held-out test set
    return 0.91   # e.g., AUC

@dsl.pipeline(name="fraud-detection-pipeline")
def fraud_pipeline(data_path: str, n_estimators: int = 100):
    train_task = train_model(data_path=data_path, n_estimators=n_estimators)
    eval_task = evaluate_model(model_path=train_task.output, test_data_path=data_path)
    with dsl.If(eval_task.output > 0.85):
        # deploy step only triggers above threshold — same gating idea as the raw Argo example
        pass

compiler.Compiler().compile(fraud_pipeline, "fraud_pipeline.yaml")
```

**What KFP adds over raw Argo Workflows specifically for ML:**
- **Component reuse and versioning** — a `@dsl.component` is a portable, containerized, versionable unit, and the KFP component registry/hub lets teams share common steps (e.g., a standard data-validation component) across pipelines rather than re-implementing them.
- **Artifact lineage tracking** — KFP automatically tracks which exact model artifact came from which exact pipeline run with which exact inputs, integrated with **ML Metadata (MLMD)**, giving you an audit trail from a deployed model back to the training run and data that produced it — directly supporting the reproducibility concern from Day 116.
- **A UI built for ML iteration** — visualizing metrics, comparing pipeline runs, and inspecting intermediate artifacts (confusion matrices, ROC curves) inline, versus Argo's more generic workflow-execution view.

## SageMaker Pipelines vs. Kubeflow

A common interview/design comparison: **SageMaker Pipelines** is AWS's fully-managed equivalent, and the tradeoff is the classic managed-vs-self-hosted one:

| | Kubeflow Pipelines | SageMaker Pipelines |
|---|---|---|
| Infrastructure | You run/operate the Kubernetes cluster + Kubeflow components | Fully managed by AWS, no cluster to operate |
| Portability | Runs on any Kubernetes cluster, any cloud or on-prem | Locked to AWS |
| Flexibility | Full control over every layer (custom operators, any container) | Constrained to SageMaker's step types and integration points, though quite broad |
| Operational burden | You own upgrades, scaling, and troubleshooting of the whole Kubeflow stack (historically a real burden — Kubeflow has a reputation for being heavy to operate) | AWS owns the undifferentiated heavy lifting |
| Cost model | Pay for underlying compute only | Pay for underlying compute + SageMaker's management premium |

**The honest framing for an interview answer**: choose Kubeflow when you need multi-cloud portability, are already deeply invested in Kubernetes, or need flexibility SageMaker's step types don't offer. Choose SageMaker Pipelines when you're AWS-committed and want to minimize the operational burden of running the orchestration layer itself — Kubeflow's own operational complexity (particularly historically) is a real, frequently-cited cost that shouldn't be hand-waved away just because it's "open source and portable."

## Points to Remember

- Argo Workflows is the general-purpose Kubernetes DAG engine; Kubeflow Pipelines is the ML-specific layer on top, adding a Python SDK, component reuse, and artifact lineage tracking (MLMD) that raw Argo doesn't provide out of the box.
- A production ML pipeline should gate the deploy step on the evaluate step's actual metric output (`when` conditions / `dsl.If`), never deploy unconditionally on "training succeeded" — training succeeding and the model being good enough are two different things.
- KFP's automatic artifact lineage tracking is what makes "which data and code produced this deployed model" an answerable question after the fact — directly operationalizing Day 116's reproducibility concerns at the pipeline level.
- SageMaker Pipelines trades portability and low-level flexibility for near-zero infrastructure operational burden; Kubeflow trades operational burden for portability and full control — this is a real tradeoff with no universally correct answer, decided by your cloud commitment and team's Kubernetes maturity.
- Kubeflow's operational complexity is a legitimate, often-cited downside, not FUD — it's a large, multi-component system (pipelines, serving, notebooks, training operators) and running it well requires real platform investment.

## Common Mistakes

- Building an ML pipeline that always proceeds to deploy once training finishes, with no explicit evaluation gate — the most common way a genuinely bad model reaches production automatically.
- Choosing Kubeflow purely because "it's the open-source/Kubernetes-native answer" without accounting for the real operational cost of running it well, then under-resourcing the platform team that has to keep it healthy.
- Re-implementing common pipeline steps (data validation, standard preprocessing) from scratch in every new KFP pipeline instead of publishing them as reusable, versioned components — loses the main leverage KFP offers over ad hoc scripting.
- Treating Argo Workflows and Kubeflow Pipelines as competitors rather than layers — KFP pipelines compile down to (or share the execution model of) Argo Workflows; asking "Argo or Kubeflow" as an either/or in an interview signals a misunderstanding of the relationship.
- Assuming SageMaker Pipelines and Kubeflow Pipelines are feature-for-feature interchangeable — they have genuinely different step-type coverage and integration points, and a lift-and-shift between them is rarely trivial.
