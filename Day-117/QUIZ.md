# Day 117 — Quiz: Model Serving & Kubeflow

Try to answer without looking at your notes. Answers are at the bottom.

1. When would batch inference be a better choice than a real-time REST/gRPC serving endpoint, even though real-time is technically possible?
2. What specific KServe feature makes it economically viable to serve dozens of independent, low-traffic models, and what underlying technology powers it?
3. What does KServe's standardized prediction protocol give you across different ML frameworks (sklearn, XGBoost, PyTorch)?
4. What's the difference between a canary deployment and a shadow deployment for models, in terms of what traffic actually reaches the challenger?
5. What is the relationship between Argo Workflows and Kubeflow Pipelines — are they competitors?
6. Why should a pipeline's "deploy" step be conditionally gated on the "evaluate" step's output rather than running unconditionally after training succeeds?
7. What does Kubeflow Pipelines' artifact lineage tracking (via ML Metadata) actually let you answer, after the fact?
8. What's the core tradeoff between Kubeflow Pipelines and SageMaker Pipelines?
9. Why is `nvidia.com/gpu` treated as a non-fractional resource by Kubernetes by default, and what must be true about `requests` vs. `limits` for it?
10. What's the purpose of tainting GPU nodes, and what goes wrong if you skip it?
11. Why is Karpenter's aggressive scale-to-zero especially high-ROI specifically for GPU node pools compared to general CPU node pools?
12. **Interview question:** How would you deploy 50 different ML models to production with independent scaling and versioning?

---

## Answers

1. When the consuming system doesn't actually need a synchronous, sub-second answer — e.g., a dashboard refreshed daily or a nightly risk-scoring job. Batch is cheaper (no idle always-on capacity between runs) and operationally simpler (no autoscaling, no cold-start latency, no inference-endpoint availability SLA) whenever predictions don't need to reflect data newer than the last batch run.
2. Scale-to-zero — an `InferenceService` with `minReplicas: 0` scales down to zero pods (and zero cost) when idle and scales back up on the next request, powered by **Knative Serving** underneath KServe. Without it, running dozens of always-on Deployments for low-traffic models 24/7 is pure waste.
3. A consistent request/response shape (`POST /v1/models/<name>:predict`) regardless of which ML framework trained the model — client/integration code doesn't need framework-specific logic to call different models.
4. A canary deployment routes a real percentage of live traffic to the challenger, and its actual response is returned to the user (real, bounded risk, but genuine causal measurement possible). A shadow deployment sends the challenger a copy of live traffic, but its output is never returned to the user (zero risk, but no causal business-outcome measurement possible since the shadow decision never actually happened).
5. Not competitors — layered. Argo Workflows is the general-purpose Kubernetes-native DAG/workflow engine; Kubeflow Pipelines is an ML-specific layer built on top (or sharing its execution model), adding a Python SDK, reusable component packaging, and ML-specific artifact lineage tracking that raw Argo doesn't provide out of the box.
6. Because training completing successfully and the resulting model being good enough for production are two different things — an unconditional deploy-after-training pipeline will ship a model that trained without error but scored below an acceptable accuracy/business-metric threshold, which a metric-gated `when`/`dsl.If` condition on the evaluate step's output prevents.
7. Which exact model artifact came from which exact pipeline run, with which exact input data and code — giving an audit trail from a deployed model back to the training run and data that produced it, directly supporting reproducibility.
8. Kubeflow Pipelines trades operational burden (you run and maintain the Kubernetes/Kubeflow stack yourself) for portability and full flexibility across clouds/on-prem. SageMaker Pipelines trades that flexibility and portability (locked to AWS, constrained to SageMaker's step types) for near-zero infrastructure operational burden, since AWS manages it fully.
9. Because a single physical GPU historically couldn't be safely subdivided between arbitrary pods at the kernel/driver level the way CPU/memory can be fractionally shared — so Kubernetes requires GPU `requests` to equal `limits`; you cannot request less GPU than you're limited to, since there's no meaningful "burst" model for GPUs by default.
10. Tainting GPU nodes (and requiring GPU-requesting pods to tolerate that taint) keeps ordinary CPU-only pods from landing on GPU nodes. Without the taint, a CPU pod can get scheduled onto a GPU node purely based on CPU/memory fit, occupying that node's general capacity and stranding the unused GPU sitting right next to it, since the scheduler has no notion that the GPU is going to waste.
11. GPU nodes are the highest cost-per-hour compute in most ML infrastructure budgets, and training/serving workloads are frequently bursty or scheduled (e.g., training runs for a few hours a day) rather than constantly busy — so keeping a GPU node pool running 24/7 when it's only used a fraction of the time wastes far more absolute dollars per idle hour than an equivalently idle CPU node pool would.
12. Strong answer: "Use KServe (or Seldon Core) so each model is its own `InferenceService`, independently versioned via its `storageUri` and independently autoscaled — including scale-to-zero for low-traffic models, so cost tracks actual usage rather than a fixed always-on footprint per model. Version and promote models through an MLflow Model Registry so redeploying a new version is a metadata/manifest change, not a rebuild. Use canary traffic splitting per model for safe rollout of new versions, gated by a pipeline (Argo/Kubeflow Pipelines) that only promotes a model past an evaluation metric threshold. Standardize the serving protocol across frameworks so client integration code doesn't fork per model. For cost control at 50-models scale, monitor per-model resource efficiency (Day 115's Kubecost concepts extended to inference workloads) since over-provisioned inference pods multiply badly across dozens of models." Mention a specific number/example if you've operated something at this scale.
