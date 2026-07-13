# Day 117 — Resources: Model Serving & Kubeflow

## Primary (assigned)

- **KServe documentation** (kserve.github.io/website) — the assigned starting point; covers `InferenceService` configuration, supported model frameworks, canary rollout, and the standardized V1/V2 inference protocols.

## Deepen your understanding

- **Kubeflow Pipelines documentation** (kubeflow.org/docs/components/pipelines) — the Python SDK (`@dsl.component`, `@dsl.pipeline`), artifact lineage via ML Metadata, and the pipeline UI.
- **Argo Workflows documentation** (argo-workflows.readthedocs.io) — DAG/steps templates, conditionals (`when`), and artifact passing between steps; useful to understand what KFP compiles down to (or builds on top of).
- **Seldon Core documentation** (docs.seldon.io) — inference graphs and advanced traffic-routing strategies (multi-armed bandits), a useful contrast to KServe's model.
- **AWS SageMaker Pipelines documentation** (docs.aws.amazon.com/sagemaker/latest/dg/pipelines.html) — for the managed-alternative comparison.
- **NVIDIA Kubernetes device plugin repo** (github.com/NVIDIA/k8s-device-plugin) — the actual mechanism that exposes `nvidia.com/gpu` as a schedulable resource, including time-slicing configuration.

## Reference / lookup

- **KServe `InferenceService` CRD API reference** (kserve.github.io/website/latest/reference/api) — every field, per predictor framework.
- **Karpenter GPU NodePool examples** (karpenter.sh — search "GPU" in the docs) — reference configs for GPU instance-type requirements and taints.

## Practice

- **KServe's own quickstart + sample models repo** (github.com/kserve/kserve, `docs/samples/`) — runnable end-to-end examples across sklearn, XGBoost, PyTorch, and custom predictors, directly usable for extending today's lab.
