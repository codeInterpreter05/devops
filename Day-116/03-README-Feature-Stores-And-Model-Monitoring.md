# Day 116 — MLOps Fundamentals: Feature Stores & Model Monitoring

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** MLOps | **Flag:** ⚡ Interview-critical

## Brief

Two problems remain even after you've solved reproducibility (MLflow/DVC): first, how do you guarantee the exact same feature computation happens at training time and at inference time (the training/serving skew problem named in the first note)? Second, how do you know when a model that was fine yesterday quietly stops being fine today, with zero code changes? Feature stores (Feast) answer the first; drift detection and A/B testing answer the second. Both are recurring interview topics because they distinguish "built one model in a notebook" from "operated ML in production."

## Feature stores: solving training/serving skew structurally

A **feature store** is a centralized system for defining, computing, storing, and serving features — the transformed inputs a model actually consumes (e.g., `user_avg_transaction_amount_7d`), as opposed to raw data. **Feast** is the leading open-source feature store.

The core problem it solves: without a feature store, feature engineering logic tends to get implemented twice — once in a batch/offline pipeline (Spark/Pandas, for training) and once in a low-latency online serving path (often a different language or framework, for real-time inference) — and those two implementations drift apart over time as one gets updated and the other doesn't. That drift is exactly what causes training/serving skew: the model receives inputs at inference time that were computed slightly differently than what it saw during training.

Feast's architecture addresses this with a **single feature definition, two storage backends**:

```python
# feature_repo/features.py
from feast import Entity, FeatureView, Field, FileSource
from feast.types import Float32

user = Entity(name="user_id", join_keys=["user_id"])

transaction_stats_source = FileSource(
    path="data/transaction_stats.parquet",
    timestamp_field="event_timestamp",
)

user_transaction_stats = FeatureView(
    name="user_transaction_stats",
    entities=[user],
    schema=[
        Field(name="avg_transaction_amount_7d", dtype=Float32),
        Field(name="transaction_count_7d", dtype=Float32),
    ],
    source=transaction_stats_source,
    online=True,
)
```

```bash
feast apply                  # register feature definitions
feast materialize-incremental $(date +%Y-%m-%d)   # push latest features into the online store (e.g. Redis/DynamoDB)
```

```python
# Training time: pull a point-in-time-correct historical dataset (offline store)
training_df = store.get_historical_features(
    entity_df=entity_df, features=["user_transaction_stats:avg_transaction_amount_7d"]
).to_df()

# Serving time: pull the latest feature values with millisecond latency (online store)
features = store.get_online_features(
    features=["user_transaction_stats:avg_transaction_amount_7d"],
    entity_rows=[{"user_id": 1001}],
).to_dict()
```

**Point-in-time correctness** is the subtlety that matters most here: when assembling historical training data, Feast joins features as they existed *as of each training example's timestamp*, not the latest value — otherwise you'd be training on information that didn't exist yet at that point in time (a subtle form of label/feature leakage that inflates offline metrics and doesn't hold up in production).

## Model drift detection

Drift is degradation in model performance over time caused by the world diverging from what the model was trained on — with zero code or deployment changes. Two distinct kinds, and they need different detection strategies:

- **Data drift (covariate shift)**: the distribution of *input features* changes — e.g., a fraud model trained pre-holiday-season sees a shift in typical transaction amounts/frequencies during the holidays. Detected by comparing production input feature distributions against the training distribution using statistical tests (Population Stability Index, KL divergence, Kolmogorov-Smirnov test) on a rolling window.
- **Concept drift**: the relationship between inputs and the *correct* output changes — e.g., fraud patterns themselves evolve as fraudsters adapt, so the same input pattern that used to indicate "legitimate" now indicates "fraud." Harder to detect directly because it requires ground-truth labels, which often arrive late (e.g., a fraud case confirmed weeks later) or not at all for some use cases.

```python
# A minimal PSI (Population Stability Index) check, conceptually:
# PSI > 0.25 typically flags significant drift; 0.1-0.25 is a warning zone
def psi(expected_dist, actual_dist, buckets=10):
    # bucket both distributions identically, then:
    return sum(
        (a - e) * np.log(a / e)
        for e, a in zip(expected_dist, actual_dist)
    )
```

In practice, teams rarely hand-roll this — tools like **Evidently AI**, **WhyLabs**, or Kubecost-adjacent observability stacks (Day 118 covers LLM-specific observability) compute these statistics continuously against a reference (training) dataset and alert when thresholds are crossed. The operational takeaway: drift detection needs a **scheduled, continuous job** comparing live inference inputs/outputs against a stored reference distribution — it is not something CI catches once at deploy time, because by definition it emerges *after* deployment with no new deploy triggering it.

## A/B testing models

Rather than fully replacing a production model with a new candidate, **A/B testing** (or canary/shadow deployment — covered operationally in Day 117) splits live traffic between the current model (control) and a challenger, measuring business/accuracy metrics on each split before committing to a full rollout.

- **Champion/challenger**: route a small percentage of traffic (e.g., 5%) to the challenger, measure statistically whether its metric (conversion, accuracy, latency) is significantly better, then ramp up gradually.
- **Shadow deployment**: send the challenger a copy of live traffic without it ever affecting the actual response returned to the user — useful when you want production-scale evaluation data with zero risk to users, at the cost of not being able to measure genuinely causal business-outcome metrics (since the shadow model's "decision" never actually happened).
- **Statistical significance matters here just like in traditional product A/B testing** — a 5% traffic split for two days is rarely enough sample size to distinguish real improvement from noise, especially for rare-event metrics like fraud catch rate.

## Points to Remember

- Feature stores solve training/serving skew structurally by having one feature definition serve both an offline (training) and online (serving) path, instead of two independently-maintained implementations drifting apart.
- Point-in-time correctness in the offline store is what prevents training on "future" feature values that wouldn't have been available at inference time — a subtle leakage bug that's easy to introduce with a naive join.
- Data drift (input distribution shift) and concept drift (input-output relationship shift) are different problems requiring different detection strategies — data drift is detectable from inputs alone; concept drift usually needs ground-truth labels, which often lag.
- Drift detection has to run as a continuous, scheduled comparison against a reference distribution — it's a production monitoring concern, not something a one-time CI check can catch.
- Shadow deployments give risk-free production-scale evaluation data but cannot measure true causal business impact, since the shadow model's outputs never actually reach the user; A/B/canary splits can measure causal impact but carry real (bounded) risk.

## Common Mistakes

- Implementing feature engineering logic separately for training (batch/offline) and serving (online/real-time) instead of through a shared feature store definition — the direct cause of training/serving skew.
- Computing historical training features with a naive join that pulls the *current* feature value instead of the value as of each example's timestamp — introduces leakage that inflates offline evaluation metrics without the model actually being better.
- Only monitoring data drift and assuming that covers "drift" generally — missing concept drift, which can degrade a model even when input distributions look stable, because labels (and thus the true input-output relationship) lag behind and aren't checked.
- Rolling out a challenger model to 100% of traffic immediately after a promising offline evaluation, skipping A/B/canary validation entirely — offline metrics on a static test set don't always predict live production behavior, especially under real user behavior and real latency constraints.
- Declaring an A/B test result significant off a tiny sample size or short time window, especially for rare-event target metrics, and shipping a "winning" challenger that was actually indistinguishable from noise.
