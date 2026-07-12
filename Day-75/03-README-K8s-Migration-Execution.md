# Day 75 — Database Migrations in CI/CD: Running Migrations in Kubernetes

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** CI/CD | **Flag:** ⚡ Interview-critical

## Brief

Knowing the expand-contract pattern (file 1) and picking a migration tool (file 2) doesn't answer the operational question every team has to solve: **exactly when and how does the migration actually run**, relative to the new application version rolling out across N replicas? Get this wrong — running a migration inside every pod's init container, for example — and you can accidentally run the same migration N times concurrently, causing lock contention or race conditions on the migration history table itself. This file covers the concrete execution mechanics: init containers vs. Kubernetes Jobs, rollback strategy for schema changes, and how Blue/Green deploys interact with migrations.

## Init containers vs. Jobs: why "vs." is a bit of a trick question

**Init containers** run once per **pod** before the main container starts — if you have 3 replicas, an init-container-based migration runs 3 times, once per pod, as each pod starts. Migration tools are generally designed to be idempotent against their own history table (a migration already recorded as applied won't re-run), so this *usually* doesn't reapply the same SQL twice — but it does mean up to 3 processes race to acquire whatever locking the tool uses on its history table simultaneously, which can cause transient failures, retries, or, in tools/databases without proper locking, a genuine race condition.

```yaml
# Anti-pattern (or at least a footgun): migration as init container
spec:
  template:
    spec:
      initContainers:
        - name: migrate
          image: myapp:v2
          command: ["flyway", "migrate"]
      containers:
        - name: app
          image: myapp:v2
```

**A Kubernetes Job**, run once as a separate step *before* the Deployment rollout begins, runs the migration exactly once, and only after it completes successfully does the rollout of the actual application Deployment proceed:

```yaml
# job-migrate.yaml — run once, gate the deploy on its success
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-v2
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp-migrations:v2
          command: ["flyway", "migrate"]
          envFrom:
            - secretRef: { name: db-credentials }
      # (Job's own restartPolicy handles single-run semantics)
```

```bash
kubectl apply -f job-migrate.yaml
kubectl wait --for=condition=complete job/migrate-v2 --timeout=300s
# only proceed to the deployment rollout if the wait succeeds
kubectl apply -f deployment-v2.yaml
```

**This is the pattern to default to.** A Job is a natural fit for "run this exactly once, and block on its result" — which is exactly the semantics a schema migration needs. In a CI/CD pipeline (tying back to Day 73), the pipeline step literally waits on `kubectl wait --for=condition=complete` and only proceeds to the rollout step if the Job succeeded — giving you a hard gate identical in spirit to the test/scan/sign gates from Day 73's pipeline.

## Ordering: migration must be expand-only, and must run *before* new code, but new code must still tolerate old schema being briefly present

Because of expand-contract (file 1), the Job's migration is always an **expand** step relative to the code it precedes — it only adds, never removes. This means:

1. The migration Job runs and completes (adds new column/table/index).
2. The Deployment rollout begins, replacing old-version pods with new-version pods one by one (rolling update).
3. During the rollout, **some pods run old code, some run new code, against the *same, already-expanded* schema** — which is fine, because expand-only changes don't break old code (the old column/behavior is still there).
4. Only in a *later* release, once 100% of traffic is confirmed on new code, does a separate migration Job run the **contract** step (drop old column, add `NOT NULL`, etc.).

This is why expand and contract must be genuinely separate deploys/migrations, not just separate SQL statements run back-to-back in the same Job — the contract step's safety depends on 100% cutover to new code, which a same-deploy Job cannot verify has actually happened yet.

## Rollback strategy for schema changes

Application code rollback is comparatively easy — redeploy the previous container image. **Schema rollback is much harder because it can be destructive to data created under the new schema**, and this asymmetry is the single most important thing to internalize about migration rollback strategy:

- **Expand-phase migrations are safe to "roll forward past," not backward.** If you deployed an expand migration (added a nullable column) and need to roll back the *application* code, you almost never need to roll back the *schema* too — the old code simply ignores the new column. Rolling back schema is usually unnecessary and adds risk for no benefit.
- **Never roll back a contract-phase migration that's already run in production** if any new data may have been written in a shape that depends on the now-removed old structure — reversing a `DROP COLUMN` means the data in that column is already gone; there is no "undo" that recovers it from nothing. The real safety net here is: *don't run the contract phase until you're certain you don't need to roll back the corresponding code anymore.*
- **Practical rollback plan:** treat contract-phase migrations as a one-way door with a mandatory waiting period (e.g., "we only drop the old column a full release cycle after 100% cutover, once we're confident"), and rely on database backups/point-in-time-recovery as the actual disaster-recovery mechanism for anything a forward-only migration tool can't cleanly undo.

```bash
# Liquibase: rollback the last N changesets using their defined rollback blocks (file 2)
liquibase rollback-count 1

# Alembic: step back one revision
alembic downgrade -1

# Flyway (no built-in down-migration in community edition):
# write and apply a new forward migration that reverses the change
```

## Blue/Green deploys with database migrations

Blue/Green deployment runs two complete environments (Blue = current live, Green = new version) and cuts traffic over atomically (or near-atomically) at the load balancer/ingress level. The wrinkle: **a shared database between Blue and Green means the same expand-contract constraints apply**, because both environments are live simultaneously during the cutover window (and Blue must remain instantly reusable as a rollback target).

- Run the **expand** migration before switching any traffic to Green — Blue (still receiving 100% of traffic) must continue working against the expanded (not yet contracted) schema.
- After cutover to Green is complete and verified, Blue is kept warm for a rollback window (commonly minutes to hours) — during this window, do **not** run the contract migration, because rolling back to Blue would then hit a schema missing something Blue's code expects.
- Only run the contract migration after the Blue environment is decommissioned or after the rollback window has definitively passed — this is functionally the same rule as the rolling-update case, just with "old code" replaced by "the entire Blue environment."

## Points to Remember

- Prefer a Kubernetes **Job** run to completion *before* the Deployment rollout over an init-container-per-pod approach — a Job runs the migration exactly once and gives you a clean, waitable gate (`kubectl wait --for=condition=complete`) analogous to a CI pipeline stage.
- Init-container migrations run once per pod (N times for N replicas), relying entirely on the migration tool's own idempotency/locking against its history table to avoid problems — riskier than it looks at first glance.
- The migration Job must complete (expand only) before the rollout begins; contract-phase migrations run in a later, separate deploy only after 100% cutover is confirmed.
- Schema rollback is fundamentally asymmetric with code rollback — rolling back application code is cheap and safe; rolling back a contract-phase (destructive) migration can be data-destroying and often isn't truly reversible.
- Blue/Green with a shared database has the identical expand-before-cutover, contract-only-after-Blue-is-gone constraint as a rolling update — the "old code" is just the entire Blue environment instead of a subset of pods.

## Common Mistakes

- Using an init container for migrations without realizing it runs once per replica — under high replica counts this can create real lock contention on the migration history table, causing flaky/failing deploys that are hard to diagnose because they "usually" work.
- Not gating the Deployment rollout on the migration Job's completion (`kubectl wait`) — the rollout starts in parallel with (rather than after) the migration, so some pods can come up before the schema change they depend on has actually landed.
- Running contract-phase migrations too early — right after cutover, before confirming 100% traffic has moved and before the old code's rollback window has passed — leaving no safe path back if a regression is found in the new version.
- Treating schema rollback as symmetric with code rollback — attempting to "just roll back the migration" after a `DROP COLUMN` has already run in production, not realizing the underlying data is genuinely gone, not just hidden.
- In Blue/Green setups, forgetting that Blue and Green share one database — deploying a breaking schema change scoped only to "Green" as if it were an isolated environment, then being surprised when Blue (still receiving traffic, or needed as a rollback target) breaks too.
