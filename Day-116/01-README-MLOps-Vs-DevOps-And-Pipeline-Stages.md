# Day 116 — MLOps Fundamentals: MLOps vs DevOps & Pipeline Stages

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** MLOps | **Flag:** ⚡ Interview-critical

## Brief

This is the pivot day into MLOps — where everything you've learned about CI/CD, containers, and Kubernetes gets reapplied to a fundamentally different artifact: a model, not just code. Interviewers ask "what makes deploying ML models different from deploying regular microservices" (today's interview question) precisely because a shocking number of "MLOps engineer" candidates can only describe the DevOps parts and go blank the moment data/model-specific concerns come up. This note builds the conceptual foundation; the next two files go deep on the specific tools (MLflow, Feast) that operationalize it.

This day is split into three focused files:

1. **This file** — MLOps vs. DevOps, and the ML pipeline's stages.
2. **[02-README-Model-Versioning-With-MLflow.md](02-README-Model-Versioning-With-MLflow.md)** — MLflow, DVC, and reproducibility.
3. **[03-README-Feature-Stores-And-Model-Monitoring.md](03-README-Feature-Stores-And-Model-Monitoring.md)** — Feast, model drift detection, A/B testing.

## MLOps vs. DevOps: what's actually different

DevOps optimizes the path from **code → tested artifact → deployed service**, and the core insight is that code is the only thing that changes between versions — given the same code and the same inputs, you get the same outputs (barring genuine bugs). MLOps adds a second, independently-changing axis: **data**. A model's behavior is a function of *code + data + hyperparameters + training environment*, and any of those four can change independently and silently alter production behavior — which is why "it passed CI" means something categorically weaker for an ML system than for a traditional service.

| Dimension | DevOps | MLOps |
|---|---|---|
| What changes between releases | Code | Code **and** data **and** model weights |
| Testing | Deterministic (same input → same output) | Statistical (same input might legitimately vary output within tolerance; success is a *distribution* of metrics, not a pass/fail assertion) |
| "It works" means | Passes unit/integration tests | Passes tests **and** meets an accuracy/business metric threshold on a held-out evaluation set |
| Rollback | Redeploy previous code version | Redeploy previous model version — but if it was trained on now-stale data, "rollback" doesn't undo the underlying data drift that made the old model necessary again |
| Degrades over time even with zero code changes | No (code doesn't rot on its own) | Yes — **model drift**: the world the model was trained on changes, and accuracy silently decays even though nothing was "deployed" |
| Artifact to version | Container image / binary | Container image **and** model weights **and** the training dataset/features that produced them |

The practical consequence: an MLOps pipeline needs everything a CI/CD pipeline needs (build, test, deploy, rollback, observability) *plus* a parallel set of concerns DevOps never had to solve — dataset versioning, experiment tracking, drift detection, and a feedback loop from production predictions back into retraining decisions.

## The ML pipeline: data → train → evaluate → deploy

Every ML pipeline, regardless of tooling, decomposes into the same four stages, and each stage has a distinct failure mode worth knowing cold:

1. **Data** — ingestion, validation, and feature engineering. Failure mode: **data quality/schema drift** — an upstream table changes a column type or a source starts sending nulls, and training silently proceeds on corrupted input. Guarded against with data validation tools (e.g., Great Expectations, TFDV) as a pipeline gate, not an afterthought.
2. **Train** — the actual model-fitting step, given data + code + hyperparameters. Failure mode: **non-reproducibility** — re-running "the same" training run six months later produces a meaningfully different model because a dependency version, a random seed, or the underlying data silently changed. This is the #1 reason experiment tracking (MLflow) and data versioning (DVC) exist — covered in the next file.
3. **Evaluate** — scoring the trained model against a held-out test set and business-relevant metrics (accuracy, F1, AUC, latency, fairness metrics — whichever apply). Failure mode: **evaluating on data the model has already seen** (train/test leakage), producing an inflated metric that doesn't hold in production; or evaluating only on an aggregate metric that hides poor performance on an important subgroup/segment.
4. **Deploy** — serving the model for inference (covered in depth on Day 117: KServe, Seldon, batch vs. real-time serving). Failure mode: **training/serving skew** — the feature engineering code path used at training time doesn't exactly match the one used at inference time (different language, different library version, different null-handling), so the model receives inputs shaped differently than what it was trained on, degrading accuracy in a way that's invisible until someone notices bad predictions in production.

**Why this matters for interviews**: naming the four stages is table-stakes; naming the *failure mode specific to each stage* and how you'd guard against it is what signals real experience.

## Reproducibility as a first-class concern

Reproducibility means: given the same code commit, the same data version, and the same hyperparameters, you can regenerate the same (or statistically equivalent) model. This requires deliberately pinning and recording every one of these, because none of them are pinned by default the way a container image pins an OS:

- **Code**: git commit SHA (same as DevOps).
- **Data**: a specific, immutable version/snapshot of the training dataset — not "the `training_data` table as it exists right now," which is a moving target. This is what DVC and dataset versioning solve.
- **Environment**: exact library versions (a `scikit-learn` or `torch` minor-version bump can change numeric results), ideally pinned in a container image, not just a loose `requirements.txt`.
- **Hyperparameters + random seed**: logged explicitly per run, not left as whatever was hardcoded in a notebook that got overwritten.
- **Hardware nondeterminism**: some GPU operations are not bit-for-bit deterministic across runs even with a fixed seed (parallel floating-point reduction order varies) — for cases where this matters, deep learning frameworks expose deterministic-mode flags at a performance cost; know this exists rather than being surprised when "identical" runs diverge slightly.

## Points to Remember

- The categorical difference from DevOps: models are a function of **data** as well as code, and data changes independently, silently, and continuously — this is the root of almost every MLOps-specific concern.
- Model quality can degrade to zero code changes at all (drift) — DevOps has no equivalent failure mode; a correctly-deployed, unchanged service does not spontaneously get worse over time the way a model does.
- Each pipeline stage (data/train/evaluate/deploy) has its own characteristic failure mode: schema drift, non-reproducibility, train/test leakage, and training/serving skew respectively — know all four, not just the stage names.
- Reproducibility requires pinning code, data, environment, and hyperparameters/seed together — pinning only code (the DevOps default) is not sufficient for ML.
- "It passed CI" is a necessary but not sufficient bar for an ML system — it also has to clear a statistical accuracy/business-metric threshold on a held-out set, which is a fundamentally different kind of gate than a deterministic test assertion.

## Common Mistakes

- Treating an ML deployment pipeline as "just CI/CD with an extra step" — missing that the entire premise of reproducible, deterministic builds breaks down once data becomes a first-class, independently-changing input.
- Evaluating a model only on an aggregate metric (overall accuracy) without checking performance on important subgroups/segments — a model can look great in aggregate while quietly failing for a minority segment that matters a lot to the business.
- Not distinguishing model **rollback** from model **retraining need** — rolling back to a previous model version doesn't fix the underlying reason the current model needed replacing (usually drift), so a rollback is a stopgap, not a fix.
- Building the feature engineering logic twice — once in a training notebook/pipeline, once in the serving path — instead of sharing one implementation, which is the direct cause of training/serving skew.
- Assuming a fixed random seed guarantees bit-for-bit reproducible training on GPU — some operations are non-deterministic by default regardless of seed, and "close enough" results get mistaken for a reproducibility bug.
