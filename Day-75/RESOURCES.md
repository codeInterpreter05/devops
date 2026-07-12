# Day 75 — Resources: Database Migrations in CI/CD

## Primary (assigned)

- **PlanetScale: "Safely making database schema changes"** / expand-contract pattern posts (planetscale.com/blog) — the assigned starting point. PlanetScale writes some of the clearest practical material on zero-downtime schema change because it's their product's core value proposition; directly informs today's hands-on activity.

## Deepen your understanding

- **"Evolutionary Database Design" by Martin Fowler** (martinfowler.com/articles/evodb.html) — the original, widely cited articulation of expand-contract (there called "parallel change") and why database schemas need the same incremental, versioned discipline as application code.
- **Flyway documentation — "Migrations"** (documentation.red-gate.com/fd): the canonical reference for naming conventions, checksum validation, and the `flyway_schema_history` mechanics.
- **Liquibase documentation — "Rollback"** (docs.liquibase.com): explains authoring rollback blocks, automatic vs. custom rollback, and `rollback-count`/`rollback-to-date` usage.
- **Alembic documentation — "Auto Generating Migrations"** (alembic.sqlalchemy.org): explicitly documents autogenerate's known limitations (renames, server-side defaults, some constraint types) — essential reading before trusting its output.

## Reference

- **Kubernetes docs — "Jobs"** (kubernetes.io/docs/concepts/workloads/controllers/job): `backoffLimit`, `restartPolicy`, and `kubectl wait --for=condition=complete` semantics used to gate a rollout on migration success.
- **PostgreSQL docs — "ALTER TABLE"** (postgresql.org/docs): confirms which schema changes take an exclusive lock (like adding a column with a volatile default, historically) vs. which are metadata-only and near-instant in modern Postgres versions — useful for reasoning about lock/downtime risk per statement.

## Practice

- **A local Postgres + Alembic/Flyway sandbox with `pgbench` or a simple load-generating script** — run a continuous stream of writes against a table while performing an expand → backfill → contract sequence live, and observe directly whether any writes fail during each phase. This turns the pattern from theory into something you've watched fail (naively) and succeed (correctly) with your own eyes.
