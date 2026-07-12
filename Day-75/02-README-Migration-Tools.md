# Day 75 — Database Migrations in CI/CD: Flyway vs. Liquibase vs. Alembic

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** CI/CD | **Flag:** ⚡ Interview-critical

## Brief

Every schema migration needs a tool that tracks *which migrations have already run* against a given database, applies new ones in order, and fails loudly if the tracked history doesn't match reality (someone manually ran a migration outside the tool, or two migrations were written with conflicting version numbers). Flyway, Liquibase, and Alembic all solve this same core problem with different philosophies — knowing which fits a given stack, and why, is a practical decision you'll actually make on real teams, not just trivia.

## The core mechanism all three share

Every migration tool maintains a **migration history table** inside the target database itself (Flyway: `flyway_schema_history`; Liquibase: `DATABASECHANGELOG`; Alembic: `alembic_version`). On every run, the tool:

1. Reads that table to see which migrations have already been applied.
2. Compares it against the migrations available in your codebase.
3. Applies only the ones not yet recorded, in a defined order, inside a transaction where the database supports transactional DDL.
4. Records success (or fails the whole run and leaves the table showing what did/didn't apply) — this is what prevents "someone ran migration 5 twice" or "migration 3 was skipped and nobody noticed."

This history table is *why* migrations must be run through the tool consistently and never manually via ad hoc `ALTER TABLE` in a psql session — a manual change makes the tracked history diverge from actual schema state, and the next real migration run may fail bafflingly or, worse, silently re-apply something already done.

## Flyway

Migrations are plain SQL files (or Java-based migrations for logic too complex for SQL), named with a strict versioned convention:

```
sql/
├── V1__create_users_table.sql
├── V2__add_email_verified_column.sql
├── V3__backfill_email_verified.sql
└── V4__add_email_verified_not_null.sql
```

```bash
flyway -url=jdbc:postgresql://localhost:5432/mydb -user=app -password=$DB_PASSWORD migrate
flyway info      # show applied vs. pending migrations
flyway validate  # confirm applied migrations match what's on disk (detects manual tampering)
```

Flyway's philosophy: **migrations are immutable, versioned, one-way SQL files.** Once `V2__...sql` has run anywhere, you never edit that file — a checksum mismatch (Flyway hashes each file) will fail validation and block deployment, which is a deliberate safety feature, not a bug people should suppress. Rollback isn't a first-class concept in the free/community edition — you write a new forward migration (`V5__revert_email_verified.sql`) that undoes the previous one, which is philosophically consistent with expand-contract (you're always moving forward, never rewriting history).

## Liquibase

Migrations are typically declarative — XML, YAML, JSON, or SQL — organized as **changesets** with unique IDs, and crucially, Liquibase supports **rollback blocks defined alongside the change itself**:

```yaml
# changelog/002-add-email-verified.yaml
databaseChangeLog:
  - changeSet:
      id: 2
      author: sarth
      changes:
        - addColumn:
            tableName: users
            columns:
              - column:
                  name: email_verified
                  type: boolean
                  defaultValueBoolean: false
      rollback:
        - dropColumn:
            tableName: users
            columnName: email_verified
```

```bash
liquibase update                 # apply pending changesets
liquibase rollback-count 1       # roll back the last changeset using its defined rollback block
liquibase status --verbose       # show pending changesets
```

Liquibase's philosophy: **explicit, declarative rollback is a first-class citizen**, defined at authoring time alongside the forward change — useful when your organization requires a documented rollback plan per migration as a change-management/compliance artifact (tying back to Day 74's SOC2 change-management controls). This comes at the cost of more verbose changesets compared to Flyway's plain SQL.

## Alembic (Python / SQLAlchemy ecosystem)

Alembic generates versioned Python migration scripts, often auto-drafted by diffing your SQLAlchemy ORM models against the current database schema:

```bash
alembic revision --autogenerate -m "add email_verified column"
# generates versions/<hash>_add_email_verified_column.py
alembic upgrade head          # apply all pending migrations
alembic downgrade -1          # roll back one migration
```

```python
# versions/<hash>_add_email_verified_column.py
def upgrade():
    op.add_column('users', sa.Column('email_verified', sa.Boolean(), server_default='false'))

def downgrade():
    op.drop_column('users', 'email_verified')
```

Alembic's philosophy: **migrations as Python code**, tightly integrated with the application's ORM models, with explicit `upgrade()`/`downgrade()` functions per migration. The `--autogenerate` convenience is powerful but must always be reviewed by hand — it reliably misses things like column renames (it sees a drop + an add, not a rename, unless you manually edit the generated script) and can miss data-migration logic entirely, since it only diffs schema shape, not data.

## Choosing between them

| | Flyway | Liquibase | Alembic |
|---|---|---|---|
| Format | Plain SQL (or Java) | XML/YAML/JSON/SQL changesets | Python scripts |
| Rollback | Write a new forward migration | Explicit rollback block per changeset | `downgrade()` function per migration |
| Best fit | Teams comfortable writing raw SQL, polyglot stacks | Teams wanting declarative, auditable rollback plans | Python/SQLAlchemy shops wanting ORM-integrated migrations |
| Autogenerate diffing | No | No | Yes (via SQLAlchemy model diffing) |
| License model | Community edition free; some features (undo, drift detection) paywalled in Teams/Enterprise | Fully open-source core | Fully open-source |

In practice: pick based on your language ecosystem and how much you value explicit rollback authorship vs. SQL simplicity. Alembic if you're already on SQLAlchemy; Flyway if your team is comfortable writing SQL directly and wants the simplest mental model; Liquibase if audited rollback plans per change are an organizational requirement.

## Points to Remember

- All three tools track applied migrations in a history table inside the target database — never bypass the tool with manual DDL, or the tracked history diverges from reality and future runs can fail or double-apply.
- Flyway: immutable versioned SQL files, checksummed — editing an already-applied migration file breaks validation by design (a safety feature, don't suppress it).
- Liquibase: changesets with explicit, authored rollback blocks — rollback is declared alongside the forward change, useful for change-management/audit requirements.
- Alembic: Python migration scripts, often autogenerated from SQLAlchemy model diffs — always manually review autogenerated migrations, since they can't detect renames or generate data-backfill logic.
- Regardless of tool, migrations should be run through CI/CD as part of the deploy pipeline (see file 3), never manually by an engineer against production.

## Common Mistakes

- Manually editing a Flyway migration file that has already run in any environment — this causes a checksum mismatch that (correctly) blocks the pipeline, and the fix is always a new forward migration, never editing history.
- Trusting Alembic's `--autogenerate` output blindly — it silently misses column renames (generates a drop+add, which loses data) and never generates backfill/data-migration logic, both of which need to be hand-written.
- Running migrations manually via a database client "just this once" during an incident, bypassing the tool entirely — this desyncs the history table from reality and can cause the next real migration run to fail or skip something.
- Writing a Liquibase changeset without a rollback block and only realizing there's no defined rollback path when an incident actually requires rolling back — write the rollback block at authoring time, not retroactively under pressure.
- Assuming any of these tools handles the *data* migration/backfill automatically — schema-migration tools change structure; backfilling values into that structure is a separate step you must write and run yourself (see file 1's Phase 2).
