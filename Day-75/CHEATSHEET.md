# Day 75 — Cheatsheet: Database Migrations in CI/CD

## Expand-contract phases

```
Phase 1 Expand:    add new column/table (nullable, with default) — old + new code both work
Phase 2 Migrate:   deploy code that writes new shape; backfill historical rows in batches
Phase 3 Contract:  remove old shape / add NOT NULL — only after 100% cutover + zero NULLs verified
```

## Safe NOT NULL column addition (SQL)

```sql
-- 1. Expand
ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;

-- 2. Backfill (batched, not one giant UPDATE)
UPDATE users SET email_verified = FALSE
WHERE email_verified IS NULL
ORDER BY id LIMIT 1000;
-- repeat until 0 rows affected

-- Verify before contract
SELECT COUNT(*) FROM users WHERE email_verified IS NULL;  -- must be 0

-- 3. Contract
ALTER TABLE users ALTER COLUMN email_verified SET NOT NULL;
```

## Flyway

```bash
flyway -url=jdbc:postgresql://host:5432/db -user=u -password=$PW migrate
flyway info                  # applied vs pending
flyway validate               # detect checksum mismatch / manual tampering
```
```
sql/V1__create_users.sql
sql/V2__add_email_verified.sql   # immutable once applied; never edit in place
```

## Liquibase

```bash
liquibase update
liquibase rollback-count 1
liquibase status --verbose
```
```yaml
changeSet:
  id: 2
  changes:
    - addColumn: {tableName: users, columns: [{column: {name: email_verified, type: boolean}}]}
  rollback:
    - dropColumn: {tableName: users, columnName: email_verified}
```

## Alembic

```bash
alembic revision --autogenerate -m "add email_verified"   # ALWAYS review the diff by hand
alembic upgrade head
alembic downgrade -1
alembic history
```
```python
def upgrade():
    op.add_column('users', sa.Column('email_verified', sa.Boolean(), server_default='false'))
def downgrade():
    op.drop_column('users', 'email_verified')
```

## Kubernetes Job pattern (migration gates the rollout)

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: migrate-v2 }
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp-migrations:v2
          command: ["alembic", "upgrade", "head"]
```

```bash
kubectl apply -f job-migrate.yaml
kubectl wait --for=condition=complete job/migrate-v2 --timeout=300s
kubectl apply -f deployment-v2.yaml   # only after the wait succeeds
```

## Init container (per-pod, risk of concurrent races) — usually avoid for migrations

```yaml
initContainers:
  - name: migrate
    image: myapp:v2
    command: ["flyway", "migrate"]
```

## Blue/Green with shared DB

```
1. Run EXPAND migration (Blue still receiving 100% traffic, must keep working)
2. Cut traffic over to Green
3. Keep Blue warm for rollback window — do NOT contract yet
4. After rollback window passes / Blue decommissioned -> run CONTRACT migration
```
