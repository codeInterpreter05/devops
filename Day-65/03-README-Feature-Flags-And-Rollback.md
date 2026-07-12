# Day 65 — Deployment Strategies: Feature Flags & Automated Rollback

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** GitOps | **Flag:** ⚡ Interview-critical

## Brief

Deployment strategies (files 1 and 2) control *how new code reaches infrastructure*. Feature flags control an entirely separate axis: *whether a specific piece of functionality is active for a specific user or segment*, decoupled from the deploy itself. Combined with automated rollback triggers, these are what let teams deploy constantly (many times a day) while still controlling exactly when and for whom a feature actually goes live — and they're the answer to the hardest part of today's interview question: handling a stateful service with a database migration safely.

## Feature flags — decoupling deploy from release

The critical distinction: **deploying** code (it exists and is running somewhere) and **releasing** a feature (users can actually see/use it) are traditionally the same event. Feature flags split them apart.

```javascript
// Application code, using an SDK like LaunchDarkly or Unleash
if (flags.isEnabled('new-checkout-flow', { userId: user.id, plan: user.plan })) {
  return renderNewCheckout();
}
return renderOldCheckout();
```

- Code for `new-checkout-flow` can be **deployed to production, fully running, behind a flag that's off** — completely decoupled from the actual rollout risk. The deploy itself becomes low-stakes (it's just running dormant code), and the *release* (flipping the flag) is a separate, instant, reversible action that doesn't require a new deploy at all.
- **Targeting rules** let you enable a flag for specific users, percentages, or segments — e.g., internal employees first, then 5% of paid customers, then everyone — independent of which pods are running which container image. This is a finer-grained, application-level analog to canary traffic splitting (file 2), but operating on *feature visibility* rather than *infrastructure traffic*.
- **Kill switch behavior**: if a newly released feature causes problems, flipping the flag off is **instant** (a config change propagated by the flag service, typically via a long-lived streaming connection or fast polling) — no redeploy, no rollback pipeline, no waiting for a new ReplicaSet to roll out. This is often faster than any infrastructure-level rollback.

**LaunchDarkly** (commercial, hosted) and **Unleash** (open-source, self-hostable) are the two most commonly referenced tools: LaunchDarkly for turnkey enterprise features (audit logs, experimentation, approval workflows), Unleash when you want to self-host or avoid per-seat/per-flag-evaluation pricing.

### Why flags and deployment strategies are complementary, not competing

A mature pipeline typically layers both: canary/blue-green (files 1–2) manage *infrastructure* risk (is the new binary stable, does it crash, does it leak memory), while feature flags manage *product* risk (is this specific feature ready for users, should marketing/support be looped in before wider release, do we want a fast A/B test). Using only deployment strategies means every feature ships to 100% of users the moment the rollout completes; using only feature flags without a real deployment strategy means a crashing binary still takes down 100% of infrastructre instantly regardless of what's flagged — you generally want both layers for anything with real risk.

## Automated rollback on failure

Automated rollback closes the loop between "we detected a problem" and "we're back to a known-good state," without requiring a human to notice and act. Argo Rollouts' `AnalysisTemplate` (file 2) is the concrete mechanism for canary/blue-green, but the *pattern* generalizes:

```yaml
strategy:
  canary:
    steps:
      - setWeight: 20
      - analysis:
          templates: [{ templateName: error-rate-check }]
    # if analysis fails failureLimit times, Argo Rollouts automatically:
    # - aborts the rollout
    # - scales the canary back to 0
    # - restores 100% traffic to the previous stable ReplicaSet
```

- **The rollback target must actually be known-good** — Argo Rollouts keeps the previous ReplicaSet alive (not scaled to zero) specifically so rollback is fast and doesn't require rebuilding/redeploying from scratch.
- **Rollback triggers should be based on symptoms users would notice** (error rate, latency percentiles, saturation) not just "did the process start" — a pod can report `Running`/`Ready` while still serving 500s to real requests if the readiness probe doesn't reflect true application health (echoing the readiness-probe lesson from file 1).
- **Automated rollback needs a time-boxed observation window** — too short, and normal noise triggers false rollbacks; too long, and a genuinely broken release keeps hurting real users for longer than necessary before the system reacts.

## Zero-downtime deployment with a database migration — the actual hard case

This is a direct answer to today's interview question: deploying application code changes is comparatively easy to make zero-downtime; the genuinely hard part is when a **schema change** is involved, because the database can't be blue/green'd the way stateless app pods can (there's usually one database, not two you swap between).

The standard safe pattern is the **expand/contract migration**:

1. **Expand**: deploy a migration that only *adds* to the schema (new nullable column, new table) — fully backward compatible with the *old* application code still running (critical, since Rolling Update/canary/blue-green all have old and new code coexisting, however briefly).
2. **Deploy new code that writes to both old and new schema** (dual-write), or that can read either — this version works whether the migration has fully propagated or not, and works with data written by both old and new code.
3. **Backfill** existing data into the new schema/column in the background, without locking the table for the live application.
4. **Deploy the final code** that only uses the new schema, once backfill is confirmed complete and you're confident no old-code writers remain.
5. **Contract**: only *after* all of the above is verified, drop the old column/table in a final migration — this step is irreversible, so it's deliberately last and separated in time from the risky code-deploy steps.

This pattern is exactly why "just roll out the new version" is insufficient for stateful services — the migration must be decomposed into backward-compatible steps that tolerate *both* old and new application code running simultaneously, because that overlap is unavoidable with essentially any zero-downtime deployment strategy (Rolling Update, canary, and even blue/green's cutover window).

## Points to Remember

- Feature flags decouple **deploy** (code exists and runs, possibly dormant) from **release** (users can see/use it) — the fastest, lowest-risk rollback mechanism available, since flipping a flag doesn't require any infrastructure change.
- LaunchDarkly (commercial/hosted) and Unleash (open-source/self-hosted) are the two names to know; both support percentage/segment-based targeting analogous to canary traffic splitting but at the feature level.
- Deployment strategies manage infrastructure/binary risk; feature flags manage product/feature risk — mature pipelines use both layers together, not one instead of the other.
- Automated rollback (via Argo Rollouts `AnalysisTemplate` or equivalent) must be judged on real symptom metrics (error rate, latency), not just process liveness, and needs a sensibly time-boxed observation window.
- Zero-downtime deploys involving a schema change require the expand/contract migration pattern — because old and new application code will run simultaneously during any real rollout strategy, the schema must tolerate both at once until a final, separate, irreversible "contract" step.

## Common Mistakes

- Treating a feature flag purely as a deploy-time toggle checked once at startup, instead of a runtime-evaluated condition — losing the ability to flip it instantly without a restart/redeploy, which defeats most of the point.
- Shipping a breaking (non-additive) database migration in the same deploy as the application code that depends on it, assuming Rolling Update's brief transition window doesn't matter — it does, because old code is still running against the new (incompatible) schema during that window.
- Setting automated rollback thresholds against noisy, low-volume metrics without accounting for statistical significance, causing false-positive rollbacks on perfectly fine releases (or the reverse: thresholds so loose that real problems don't trigger anything).
- Forgetting the final "contract" step (dropping the old column/table) forever, leaving permanent dual-schema cruft and confusing future engineers about which column is actually authoritative.
- Relying solely on feature flags for risk control while skipping any real deployment strategy — a genuinely crashing binary (not just a bad feature) still takes down all traffic immediately regardless of flag state, since flags gate features, not infrastructure stability.
