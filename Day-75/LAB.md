# Day 75 — Lab: Database Migrations in CI/CD

**Goal:** Implement a real expand-contract migration end-to-end — add a nullable column, deploy an app that populates it, backfill historical rows, then add a `NOT NULL` constraint — running the migration as a Kubernetes Job gating a rolling deploy, with a working rollback plan.

**Prerequisites:**
- A kind/minikube cluster.
- A Postgres instance (in-cluster via a simple Deployment+Service, or `docker run postgres:16`).
- Alembic (`pip install alembic sqlalchemy psycopg2-binary`) or Flyway CLI — pick one to work through concretely.
- A minimal app (even a 20-line Flask/FastAPI app with a `users` table) you can redeploy with 3 replicas.

---

### Lab 1 — Stand up the baseline

1. Deploy Postgres and create a `users` table with columns `id, email` and seed 20 rows.
2. Deploy your app (`v1`) with 3 replicas, connected to that database, confirm it's serving traffic.
3. Confirm the migration tool's history table exists (`alembic_version` or `flyway_schema_history`) and shows the baseline migration applied.

**Success criteria:** 3 replicas of `v1` running, serving reads/writes against a seeded `users` table, with a clean migration history baseline.

---

### Lab 2 — Expand: add the nullable column via a Job

1. Write the migration (Alembic example):
   ```python
   def upgrade():
       op.add_column('users', sa.Column('email_verified', sa.Boolean(), server_default='false', nullable=True))
   def downgrade():
       op.drop_column('users', 'email_verified')
   ```
2. Package it into a migration image, define a Job (see `03-README-K8s-Migration-Execution.md`), and run it:
   ```bash
   kubectl apply -f job-migrate-expand.yaml
   kubectl wait --for=condition=complete job/migrate-expand --timeout=120s
   ```
3. Confirm `v1` pods (still running, unaware of the new column) continue serving traffic without errors.

**Success criteria:** The column exists, defaults correctly for new rows, and zero `v1` pods error out — proving the expand step is non-breaking for old code.

---

### Lab 3 — Deploy code that populates the new column, then backfill

1. Deploy `v2` of the app: on every insert/update to `users`, it now sets `email_verified` explicitly. Roll it out (`kubectl rollout status deployment/app`) and confirm mixed `v1`/`v2` pods coexist briefly without error during the rollout.
2. Run a batched backfill job for historical rows:
   ```sql
   UPDATE users SET email_verified = FALSE
   WHERE email_verified IS NULL
   ORDER BY id
   LIMIT 1000;
   -- repeat until 0 rows affected
   ```
   Or script it as a loop that stops when `SELECT COUNT(*) FROM users WHERE email_verified IS NULL` returns 0.
3. Verify: `SELECT COUNT(*) FROM users WHERE email_verified IS NULL;` must return `0` before proceeding.

**Success criteria:** Every row has a non-null `email_verified` value, and all running pods are on `v2` (rollout fully complete).

---

### Lab 4 — Contract: enforce NOT NULL, only after verification

1. Confirm (again) zero NULLs and 100% of pods on `v2`.
2. Write and run the contract migration as a separate Job:
   ```python
   def upgrade():
       op.alter_column('users', 'email_verified', nullable=False)
   def downgrade():
       op.alter_column('users', 'email_verified', nullable=True)
   ```
3. Apply it, and confirm the constraint is now enforced:
   ```bash
   kubectl apply -f job-migrate-contract.yaml
   kubectl wait --for=condition=complete job/migrate-contract --timeout=120s
   ```
4. Try (and expect to fail) an insert omitting `email_verified` directly via `psql` to prove the constraint is live.

**Success criteria:** The `NOT NULL` constraint is enforced at the database level, and you can point to the exact sequence of expand → backfill → contract Jobs that got you there safely.

---

### Lab 5 — Rehearse a rollback

1. Simulate needing to roll back `v2` to `v1` *after* the expand step but *before* the contract step. Redeploy `v1` and confirm it still works fine against the expanded (but not yet contracted) schema.
2. Attempt a rollback *after* the contract step has run — try `alembic downgrade -1` (or the Liquibase/Flyway equivalent) and observe that while the constraint itself can be reversed, any data-shape assumptions from `v2`-only writes may not cleanly revert. Document in a sentence why contract-phase rollback is riskier than expand-phase rollback.

**Success criteria:** You can demonstrate live that rolling back application code after only the expand step is safe, and articulate concretely why rolling back after the contract step is not symmetric.

---

### Cleanup

```bash
kubectl delete job migrate-expand migrate-contract --ignore-not-found
kubectl delete deployment app postgres --ignore-not-found
kubectl delete pvc --all
```

### Stretch challenge

Wire the full expand → deploy-v2 → verify-zero-nulls → contract sequence into a single GitHub Actions workflow (tying back to Day 73) where each step is a job gated on `needs:`, and the "verify zero nulls" step queries the database and fails the pipeline (blocking the contract job) if any NULLs remain.
