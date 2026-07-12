# Day 75 — Database Migrations in CI/CD: The Expand-Contract Pattern

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** CI/CD | **Flag:** ⚡ Interview-critical

## Brief

Schema migrations are where "just add a `NOT NULL` column" turns into a production outage. The moment you have more than one instance of your application running (which is every production system worth deploying carefully), the application code and the database schema are **never atomically upgraded together** — there's always a window where old code and new code, or old schema and new schema, coexist. The expand-contract pattern is the standard, battle-tested technique for making schema changes safe under that reality, and it's one of the most reliably asked senior-level interview questions in this space precisely because so many engineers have only ever worked with a single-instance dev database where this problem never surfaces.

This day is split into three files:

1. **This file** — the expand-contract pattern itself and why naive migrations break zero-downtime deploys.
2. **[02-README-Migration-Tools.md](02-README-Migration-Tools.md)** — Flyway vs. Liquibase vs. Alembic, how each fits into a CI/CD pipeline.
3. **[03-README-K8s-Migration-Execution.md](03-README-K8s-Migration-Execution.md)** — running migrations via init containers vs. Jobs, rollback strategy, and Blue/Green with DB changes.

## Why a "simple" schema change isn't simple

Say you deploy a new application version that renames a column, and you have 3 running replicas behind a load balancer, deploying via a rolling update. The instant you `ALTER TABLE ... RENAME COLUMN`, **every replica still running the old code** starts failing every query against that table — because the old code references the old column name, which no longer exists. There is no instant in a rolling deploy where 100% of replicas are simultaneously updated *and* the schema change simultaneously applies — there's always overlap, and a naive destructive migration guarantees an outage during that overlap window, however brief.

The same problem applies to:
- Dropping a column the *previous* app version still reads.
- Adding a `NOT NULL` column with no default — old code doesn't know to populate it, so its inserts fail immediately.
- Changing a column's type in place — old code serializes/deserializes assuming the old type.

## The expand-contract pattern

The fix is to never make a breaking schema change in one step. Instead, split every schema change into phases where **both the old and new schema shapes are valid simultaneously**, and only remove the old shape once you're certain nothing depends on it anymore.

**Phase 1 — Expand.** Add the new schema element *without removing or breaking the old one*. Both old and new application code work against the database in this state.

**Phase 2 — Migrate/backfill.** Deploy new application code that writes to *both* old and new locations (dual-write) or reads from whichever is populated, while a backfill job populates historical data into the new structure.

**Phase 3 — Contract.** Once 100% of traffic is on the new code path and backfill is complete and verified, remove the old schema element (drop the old column, remove the dual-write logic, add the `NOT NULL` constraint, etc.).

### Concrete example: adding a `NOT NULL` column

This is precisely the assigned hands-on activity today, and it's the canonical expand-contract example:

```sql
-- Phase 1 (Expand): add the column as NULLABLE, with a default for new rows
ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;
-- Safe: old code ignores the new column entirely; it has a default so
-- inserts from OLD code (which don't mention this column) still succeed.
```

```sql
-- Phase 2 (Backfill): populate historical rows in batches, not one giant transaction
UPDATE users SET email_verified = FALSE
WHERE email_verified IS NULL
  AND id BETWEEN 1 AND 10000;
-- repeat in batches to avoid a long-held lock / huge transaction log on a large table
```

Deploy new application code in this window that actively *writes* `email_verified` correctly for new/updated rows — so by the time you reach Phase 3, every row (old and new) has a correct, non-null value.

```sql
-- Phase 3 (Contract): now that every row has a value and all code writes it,
-- enforce the constraint
ALTER TABLE users ALTER COLUMN email_verified SET NOT NULL;
```

Only run the Phase 3 statement after confirming via query (`SELECT COUNT(*) FROM users WHERE email_verified IS NULL` returns 0) **and** after 100% of running application instances are on code that populates the column — otherwise you'll break inserts from any lingering old-version pod exactly like the naive single-step migration would have.

### Why "expand" must always come before "contract" — never combined

A common mistake is to try to do the rename/drop *in the same deploy* as the expand, reasoning "the new code is what's rolling out anyway." This ignores that a rolling deployment is never instantaneous — for the entire rollout window (seconds to minutes depending on replica count and health-check timing), old and new code run side-by-side. Expand-contract's core insight is: **decouple the schema timeline from the code timeline**, so schema changes are never gated on "did every replica finish deploying at the exact same instant" — an assumption that's false by construction in any rolling/canary/blue-green deployment strategy.

## Points to Remember

- The core problem: rolling deploys guarantee a window where old and new application code run simultaneously against the same database — a one-step breaking schema change is guaranteed to break whichever code version doesn't match it during that window.
- Expand-contract has three phases: expand (add new, don't touch old), migrate/backfill (populate + dual-write/dual-read), contract (remove old, only after 100% cutover and verified backfill).
- Adding a `NOT NULL` column safely always means: add nullable with a default first, backfill, deploy code that populates it, *then* add the constraint — never all in one migration.
- Never combine "expand" and "contract" in the same deploy — the whole point is decoupling the schema change timeline from the rolling-deploy timeline, which are never atomic together.
- Backfills on large tables should run in small batches with brief pauses, not one giant `UPDATE`/transaction — long transactions hold locks, bloat the write-ahead log, and can cause replication lag or timeouts.

## Common Mistakes

- Adding a `NOT NULL` column with no default in the same migration as the code that populates it, breaking every old-version replica's inserts the instant the migration runs, before the new code has even finished rolling out.
- Dropping a column or table in the same release as removing the code that used it — some fraction of requests will still hit an old-version replica during rollout and error immediately.
- Running a single massive `UPDATE` to backfill millions of rows, holding a long transaction/lock that blocks other writes and can trigger replication lag or lock timeouts in production.
- Skipping the verification step before contract ("assuming" the backfill finished instead of querying to confirm zero NULLs remain) and then having the `NOT NULL` constraint fail to apply — or worse, succeed and then have some old code path insert a NULL that slipped through.
- Treating expand-contract as only relevant to "big" migrations — the same pattern applies to seemingly small changes like renaming a column, which needs an expand (add new column), a dual-write phase, and a contract (drop old column), never a direct `RENAME COLUMN` under live rolling traffic.
