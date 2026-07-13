# Day 117 — Model Serving & Kubeflow: Serving Patterns & KServe

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** MLOps | **Flag:** ⚡ Interview-critical

## Brief

Day 116 covered getting a model *trained and registered*; today covers getting it *in front of real traffic*, at scale, with independent versioning per model — the "deploy 50 different ML models to production with independent scaling and versioning" interview question is asked because it's exactly the operational reality of any team running more than a handful of models, and naive approaches (one Flask app per model, manually deployed) collapse immediately at that scale. KServe is the tool most directly built to solve this on Kubernetes.

This day is split into three focused files:

1. **This file** — serving patterns (REST/gRPC/batch) and KServe/Seldon Core.
2. **[02-README-Kubeflow-Pipelines-And-Argo-Workflows.md](02-README-Kubeflow-Pipelines-And-Argo-Workflows.md)** — Argo Workflows, Kubeflow Pipelines, and SageMaker Pipelines comparison.
3. **[03-README-GPU-Workload-Scheduling.md](03-README-GPU-Workload-Scheduling.md)** — GPU scheduling for training/serving workloads.

## Serving patterns: REST vs. gRPC vs. batch

Three fundamentally different consumption patterns, and picking the wrong one is a common design mistake:

- **REST (HTTP/JSON)**: simplest to integrate with anything, human-debuggable, but JSON serialization overhead and HTTP/1.1 connection handling add latency that matters for high-QPS, low-latency use cases (e.g., real-time fraud scoring on the request path). Universally supported by every serving framework as the default.
- **gRPC (HTTP/2 + Protobuf)**: lower latency and smaller payloads than REST/JSON, supports streaming (useful for token-by-token LLM output or continuous sensor-data inference), and multiplexes many requests over one connection. Costs you: less human-debuggable, requires `.proto` schema management, and not every client environment has equally mature gRPC support (browsers need a gRPC-Web proxy layer).
- **Batch inference**: no live request/response at all — a scheduled job scores a large dataset offline (e.g., nightly churn-risk scoring for every customer) and writes results to a table/warehouse for downstream consumption. Appropriate whenever predictions don't need to reflect data newer than the last batch run, and dramatically cheaper per-prediction than always-on real-time serving because there's no idle capacity between runs.

**The decision that actually matters**: don't default to real-time REST serving because it's the most familiar pattern — ask whether the consuming system genuinely needs a synchronous, sub-second answer. A huge fraction of "we need to deploy a real-time endpoint" requests are actually batch use cases in disguise (a dashboard refreshed daily doesn't need millisecond inference), and batch is both cheaper and operationally simpler (no autoscaling, no cold-start latency, no availability SLA on an inference endpoint).

## KServe: Kubernetes-native model serving

KServe (formerly KFServing) provides a `InferenceService` CRD that abstracts away the serving-infrastructure boilerplate (autoscaling, canary rollout, request routing, standardized input/output protocol) that you'd otherwise hand-build per model.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: fraud-detector
  namespace: mlops
spec:
  predictor:
    sklearn:
      storageUri: "s3://mlflow-artifacts/models/fraud-detector/3"
      resources:
        requests: {cpu: "1", memory: "2Gi"}
        limits: {cpu: "2", memory: "4Gi"}
    minReplicas: 1
    maxReplicas: 10
```

What this one manifest gets you, that you'd otherwise build by hand:

- **Serverless autoscaling to zero** (via Knative Serving under the hood) — a rarely-used model can scale down to zero replicas and pay nothing until the next request, which matters enormously once you have dozens of low-traffic models (the "50 models" scenario in the interview question) — running 50 always-on Deployments 24/7 for models that each get a handful of requests/day is pure waste.
- **A standardized prediction protocol (V1/V2 inference protocol)** — every model, regardless of framework (sklearn, XGBoost, PyTorch, TensorFlow, custom), is queryable via the same `POST /v1/models/<name>:predict` shape, so client code doesn't need framework-specific logic.
- **Built-in canary rollout**, splitting traffic between two revisions of the same `InferenceService` by weight:
  ```yaml
  spec:
    predictor:
      canaryTrafficPercent: 10
      sklearn:
        storageUri: "s3://mlflow-artifacts/models/fraud-detector/4"   # new version
  ```
  KServe/Knative handles the traffic split and gradual promotion natively — you don't hand-build an Nginx/Istio weighted-routing config per model.

## A/B testing two model versions with KServe

Today's hands-on activity is exactly this: deploying a model via KServe and setting up A/B testing between two versions. Practically, this means running two `InferenceService`s (or one with a canary configuration) and comparing metrics collected from each via your observability stack:

```yaml
# Explicit A/B via a separate InferenceService + your own traffic-splitting Ingress/VirtualService
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata: {name: fraud-detector-challenger, namespace: mlops}
spec:
  predictor:
    sklearn: {storageUri: "s3://mlflow-artifacts/models/fraud-detector/4"}
```
```yaml
# Istio VirtualService splitting traffic 90/10 between champion and challenger
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: {name: fraud-detector-split, namespace: mlops}
spec:
  hosts: ["fraud-detector.mlops.example.com"]
  http:
    - route:
        - destination: {host: fraud-detector-predictor-default}
          weight: 90
        - destination: {host: fraud-detector-challenger-predictor-default}
          weight: 10
```

## Seldon Core: the alternative

**Seldon Core** solves largely the same problem as KServe — Kubernetes-native model serving with a CRD-based approach — but historically differentiated on more sophisticated **inference graphs** (multi-step pipelines: a preprocessor → model → outlier detector → explainer, all as a single deployable graph) and built-in support for advanced deployment strategies (multi-armed bandit traffic routing, not just fixed-weight A/B). The practical choice between KServe and Seldon Core in a real org usually comes down to ecosystem fit — KServe integrates more tightly with the broader Kubeflow ecosystem and Knative's serverless model, while Seldon Core has historically been favored when complex inference graphs (multiple chained models/business-logic steps) are the norm rather than the exception.

## Points to Remember

- Don't default to real-time REST serving — check whether the use case genuinely needs synchronous, sub-second responses; a large fraction of "real-time" requests are batch jobs in disguise, and batch is cheaper and operationally simpler.
- KServe's scale-to-zero (via Knative) is the specific feature that makes "50 independent models" economically sane — without it, you're paying for 50 always-on services regardless of individual traffic.
- KServe standardizes the prediction protocol across ML frameworks, so serving infrastructure/client code doesn't need per-framework special-casing.
- Canary/A-B traffic splitting for models is conceptually identical to canary deployments for regular microservices — the same weighted-routing mechanisms (Knative revisions, Istio VirtualServices) apply, just pointed at model-serving endpoints instead of app code.
- Seldon Core and KServe solve overlapping problems; the differentiator that actually matters when choosing is whether you need complex multi-step inference graphs (Seldon's traditional strength) versus tight Kubeflow/Knative ecosystem fit (KServe).

## Common Mistakes

- Deploying every model as an always-on Deployment with fixed replica counts "for simplicity," then being surprised at the cluster cost once dozens of low-traffic models accumulate — this is precisely the waste KServe's scale-to-zero exists to eliminate.
- Choosing gRPC for a serving endpoint mainly "because it's faster," without checking whether every consuming client environment (including browsers, if applicable) can actually speak gRPC without an extra proxy layer.
- Building a real-time REST endpoint for a use case that's actually consumed by a nightly batch job or a daily-refreshed dashboard — adds unnecessary availability/latency SLAs and autoscaling complexity for no real benefit.
- Running an A/B test between model versions with a traffic split too small or too short to reach statistical significance, then declaring a "winner" based on noise (same failure mode as Day 116's A/B testing note, but easy to repeat here because the deployment mechanics work regardless of sample size).
- Assuming KServe's standardized prediction protocol means zero code changes are ever needed when swapping frameworks — the *protocol* is standardized, but a custom pre/post-processing step (e.g., feature transforms not baked into the model artifact itself) still needs to be ported explicitly, often via a custom KServe "transformer" component.
