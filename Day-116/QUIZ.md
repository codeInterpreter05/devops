# Day 116 — Quiz: MLOps Fundamentals

Try to answer without looking at your notes. Answers are at the bottom.

1. What is the categorical difference between what changes a DevOps deployment's behavior versus what can change an ML model's behavior?
2. What is model drift, and why does it have no direct equivalent in traditional DevOps?
3. Name the four stages of a typical ML pipeline, and one characteristic failure mode for each.
4. What is training/serving skew, and what's the structural fix for it?
5. What are the four components of MLflow, and what does each one actually do?
6. Why should a production MLflow server use Postgres + S3 rather than SQLite + local disk?
7. What problem does DVC solve that MLflow does not solve on its own?
8. What is point-in-time correctness in a feature store's offline retrieval, and what bug does it prevent?
9. What's the difference between data drift and concept drift, and why is concept drift generally harder to detect?
10. What's the difference between a shadow deployment and a canary/A/B deployment, and what's the key limitation of shadow deployments?
11. Why is a fixed random seed not a guarantee of bit-for-bit reproducible training?
12. **Interview question:** What makes deploying ML models different from deploying regular microservices?

---

## Answers

1. A DevOps deployment's behavior is a function of code alone — given the same code and inputs, outputs are deterministic. An ML model's behavior is a function of code **and** data **and** hyperparameters **and** training environment, any of which can change independently and silently alter production behavior.
2. Model drift is the silent degradation of a model's accuracy over time as the real-world data distribution or the input-output relationship diverges from what the model was trained on — with **zero code or deployment changes**. Traditional DevOps has no equivalent: a correctly deployed, unchanged service does not spontaneously get worse purely from time passing.
3. Data (ingestion/validation/feature engineering — failure mode: schema/data drift corrupting training input), Train (fitting the model — failure mode: non-reproducibility), Evaluate (scoring against held-out data — failure mode: train/test leakage or hidden subgroup underperformance), Deploy (serving for inference — failure mode: training/serving skew).
4. Training/serving skew is when the feature engineering logic used at training time doesn't exactly match what's used at inference time (different implementation, library version, or null-handling), so the model receives differently-shaped inputs in production than it saw during training. The structural fix is a **feature store** (e.g., Feast) with a single feature definition serving both the offline (training) and online (serving) paths.
5. **Tracking** (logs params/metrics/artifacts per run), **Projects** (packaging format for reproducible runs), **Models** (standard model packaging format across ML libraries), **Model Registry** (versioned, stage/alias-aware store for registered models, enabling promotion workflows).
6. SQLite and local disk don't scale for a shared, concurrent, production use case — SQLite handles concurrent writes poorly under multi-user load, and local disk artifact storage doesn't scale once teams log multi-GB model weight files, and offers no durability/replication guarantees a database or S3 provides.
7. DVC efficiently versions large datasets using Git-like semantics (lightweight pointer files in git, actual data in a separate remote like S3), which MLflow does not do natively — MLflow tracks runs/models/metrics well but isn't built to version large training datasets themselves.
8. Point-in-time correctness means that when assembling historical training data, features are joined using their value **as of each training example's timestamp**, not the latest/current value. It prevents a subtle leakage bug where a model is trained on feature values that wouldn't actually have existed yet at that point in time, which inflates offline metrics without genuinely improving the model.
9. Data drift is a shift in the distribution of **input features** (detectable from inputs alone, e.g., via PSI/KL divergence/KS test against a training baseline). Concept drift is a shift in the **relationship between inputs and the correct output** — detecting it generally requires ground-truth labels, which often arrive late or not at all, making it harder to catch in real time than data drift.
10. A shadow deployment sends the challenger a copy of live traffic without ever returning its output to the user — zero risk, but it can't measure true causal business-outcome impact since the shadow model's decision never actually happened. A canary/A/B deployment routes a real (small) percentage of traffic to the challenger's actual response, allowing causal measurement of business/accuracy metrics, at the cost of bounded real risk.
11. Some GPU operations are not bit-for-bit deterministic by default even with a fixed seed, because parallel floating-point reduction order can vary between runs — deep learning frameworks expose separate deterministic-mode flags (at a performance cost) for cases where exact reproducibility is required.
12. Strong answer: "A regular microservice's behavior is fully determined by its code — same input, same output, and passing CI is a reliable predictor of correct behavior. A model's behavior additionally depends on the data it was trained on, so it needs a parallel set of concerns a microservice doesn't: experiment tracking and dataset versioning for reproducibility, a feature store to avoid skew between training and serving computation, continuous drift monitoring because the model can silently degrade with zero code changes as the real world diverges from training data, and statistical (not just pass/fail) evaluation and A/B/canary rollout because 'better' is a distributional claim, not a deterministic assertion. Rollback also means something weaker for a model — reverting to a prior version doesn't fix the underlying drift that made a new model necessary in the first place." Follow up with a concrete example if you have one (a model you've seen drift, or a training/serving skew bug you've debugged).
