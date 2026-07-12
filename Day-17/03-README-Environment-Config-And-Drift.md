# Day 17 — YAML / JSON / Config Parsing: Environment Config & Drift Detection

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Python DevOps | **Flag:** —

## Brief

"How do you manage application configuration across dev/staging/prod without duplicating YAML?" is a question every DevOps interviewer asks in some form, because the naive answer — three nearly-identical copies of every config file, one per environment — is exactly the failure mode that causes real incidents: someone updates the prod value and forgets staging, or a typo in one copy goes unnoticed until it deploys. This file covers the actual toolchain for solving it: local developer convenience with `python-dotenv`, typed/validated settings from environment variables with `pydantic-settings`, the base+overlay pattern for eliminating YAML duplication, and the discipline of detecting when a deployed system's *actual* configuration has quietly diverged from what's declared in version control (config drift). This is the interview-critical file of the day — read it slowly.

## The core principle: one shape, environment-specific values

The right mental model (formalized as factor III of the [Twelve-Factor App](https://12factor.net/config)) is: **configuration that varies between deploys lives outside the code, injected at runtime — never duplicated inside it.** Concretely: one schema or one template defines the *shape* of config exactly once; each environment supplies only the *values* that differ. If your dev/staging/prod config files are each hundreds of lines and 95% identical, you've built the exact problem this file solves.

## `python-dotenv` — local developer convenience

```bash
# .env — local only, never committed (must be in .gitignore)
DATABASE_URL=postgres://localhost/dev_db
LOG_LEVEL=DEBUG
AWS_PROFILE=dev
```

```python
from dotenv import load_dotenv
import os

load_dotenv()   # reads .env into os.environ for this process; does NOT override a var already set
db_url = os.environ["DATABASE_URL"]
```

`python-dotenv` solves exactly one problem: letting a developer keep local-only environment variables in a file instead of exporting them in their shell every session. It provides **no validation, no type coercion, and no defaults** beyond what `os.environ` already gives you — it's a loading mechanism, not a config framework, and by default it will not clobber a variable your shell already exported (useful for letting a CI system's real env vars take precedence over a stray `.env` file). In staging/prod, environment variables are normally injected directly by the deployment platform (Kubernetes `env`/`envFrom`/`Secret`, ECS task definitions, systemd `EnvironmentFile=`) — `.env` files and `load_dotenv()` are a **local-dev-only** convenience and should never be the mechanism by which a real environment receives its config.

## `pydantic-settings` — typed, validated config

```python
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DB_", env_file=".env", extra="ignore")

    host: str
    port: int = 5432
    user: str
    password: SecretStr          # never renders/logs as plaintext
    name: str

    @property
    def dsn(self) -> str:
        return (
            f"postgresql://{self.user}:{self.password.get_secret_value()}"
            f"@{self.host}:{self.port}/{self.name}"
        )

settings = DatabaseSettings()     # raises ValidationError immediately if a required var is missing/wrong type
print(settings)
# host='localhost' port=5432 user='app_user' password=SecretStr('**********') name='payments'
```

Reading `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` from the environment (or from `.env` if `env_file` is set and the process env doesn't already have them — process env always wins over the file), this is the meaningful upgrade over `os.environ.get(...)` scattered through a codebase: **fail fast, with a clear error, at import/startup time** if configuration is missing or malformed, instead of a `TypeError` or a silent wrong default deep inside business logic hours later. Every setting is typed and validated exactly once, in one place, and the rest of the application imports a fully-validated `Settings` object instead of re-parsing environment variables ad hoc across a dozen files.

`SecretStr` matters specifically because config objects get printed, logged, and included in stack traces constantly during debugging — a raw `str` password field will eventually end up in a log line someone pastes into a Slack channel or a bug tracker. `SecretStr`'s `repr()`/`str()` always shows `**********`; you must explicitly call `.get_secret_value()` to get the real value, which makes "did I just log a password" a visible, deliberate act instead of an accident.

For structured, nested config, `pydantic-settings` maps a delimiter in the env var name to nested model fields:

```python
class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(env_nested_delimiter="__")

    debug: bool = False
    database: DatabaseSettings
```

```bash
DEBUG=true
DATABASE__HOST=localhost
DATABASE__PORT=5432
```

## Solving "config across dev/staging/prod" without duplicating YAML

Three complementary patterns, usually combined:

**1. One schema, environment-specific values only.** Define the *shape* of config once (a `pydantic-settings` class, or a JSON Schema from file 1), and let each environment supply only its differing *values* via environment variables — nothing about the structure is ever copy-pasted.

**2. Base + overlay YAML.** A `base.yaml` holds shared defaults; a small `overlays/<env>.yaml` holds only the handful of keys that differ, deep-merged at deploy time:

```yaml
# config/base.yaml
service_name: payments-api
image: registry.example.com/payments-api
container_port: 8080
resources:
  cpu: "250m"
  memory: "256Mi"
env:
  LOG_LEVEL: info
```

```yaml
# config/overlays/production.yaml
replicas: 6
environment: production
resources:
  cpu: "500m"
  memory: "512Mi"
```

```python
from copy import deepcopy
from typing import Any
import yaml

def deep_merge(base: dict[str, Any], override: dict[str, Any]) -> dict[str, Any]:
    result = deepcopy(base)
    for key, value in override.items():
        if isinstance(value, dict) and isinstance(result.get(key), dict):
            result[key] = deep_merge(result[key], value)
        else:
            result[key] = value
    return result

base = yaml.safe_load(open("config/base.yaml"))
overlay = yaml.safe_load(open("config/overlays/production.yaml"))
config = deep_merge(base, overlay)
```

Note the merge is recursive on nested dicts (`resources.cpu` gets overridden while `resources.memory`... in this example both happen to be overridden, but if the overlay had only changed `cpu`, `memory` would survive untouched from base) — a naive `{**base, **overlay}` shallow merge would instead **replace the entire `resources` sub-dict**, silently dropping any base key the overlay didn't repeat. This is exactly the mechanism Kustomize formalizes for Kubernetes specifically (a `kustomization.yaml` + `patchesStrategicMerge`), and what Helm achieves via layered `values.yaml` files (`-f values.yaml -f values-prod.yaml`). The size of the overlay file is a direct, honest measure of how much actually differs between environments — a handful of lines, not a full duplicate.

**3. Template + per-environment data file (yesterday's Jinja2 pattern, file 2).** One `.j2` template (structure) rendered against `dev.yaml`/`staging.yaml`/`prod.yaml` (small, values-only data files). Functionally the same idea as pattern 2, expressed through templating instead of dict-merging — pick whichever your tooling ecosystem already leans toward (Kustomize/Helm for Kubernetes specifically, Jinja2 for anything custom).

**The interview-ready answer:** "I keep one canonical schema or template describing the *shape* of config, and only the environment-specific *values* differ per environment — supplied via environment variables validated through something like `pydantic-settings`, or small overlay files deep-merged onto a shared base. Secrets never live in the YAML at all; they come from environment variables or a secret manager injected at deploy time. I validate the fully-resolved config against a schema before it's used, so a typo or missing key fails in CI, not at 2am in production. I never maintain three full copies of the same YAML file, because that's exactly how drift and copy-paste mistakes happen."

## Config drift detection

**Config drift** is when the configuration actually running in an environment no longer matches what's declared in version control — a manual `kubectl edit`, an emergency hotfix applied directly to a server, a console change to a cloud resource that Terraform doesn't know about. Left undetected, drift means your "source of truth" is lying: the next deploy or `terraform apply` can silently revert someone's emergency fix, or fail unexpectedly because reality doesn't match what the tooling assumes is there. Drift is also how "works in staging, breaks in prod" bugs happen in reverse — staging quietly diverges from the config prod actually needs, and nobody notices until a promotion fails.

Detection approaches, by ecosystem:

- **Terraform/OpenTofu** — `terraform plan` against unchanged `.tf` source is a drift check by definition: a non-empty diff means something in real infrastructure changed outside Terraform. Run it on a schedule (not only right before a deploy) specifically to catch drift between deploys. **`driftctl`** is a purpose-built open-source tool that scans real cloud resources and reports anything not accounted for in Terraform state, including resources created entirely outside IaC.
- **Kubernetes, imperative check** — `kubectl diff -f manifest.yaml` shows exactly what would change if you applied a manifest, without applying it — a manual drift check against a specific file.
- **Kubernetes, GitOps** — Argo CD marks an application **OutOfSync** the moment the live cluster state and the Git-declared state diverge, and can be configured to **self-heal** (automatically re-apply the Git state, reverting the manual change) rather than just alerting. Flux's reconciliation loop works the same way. This turns drift detection from a periodic audit into a continuous, enforced invariant.
- **Ansible/Chef/Puppet** — these tools are naturally idempotent and convergent: a scheduled re-run of `ansible-playbook --check --diff` (check mode, no changes applied) against a fleet reports exactly what's out of sync with the playbook's declared state, without touching anything.
- **AWS Config** — records configuration changes to cloud resources over time and evaluates them against defined rules, flagging any resource that's drifted from an expected baseline, independent of which IaC tool (if any) manages it.

The pattern all of these share: **treat "no diff" as the expected, boring, alertable-on-violation state**, and make detection **continuous** — a cron job, a GitOps controller's reconciliation loop, a scheduled `plan` — not a one-time audit that only runs when someone remembers to look.

## Points to Remember

- `python-dotenv` loads a `.env` file into `os.environ` for local development convenience only — no validation, no type coercion, and it must never be how staging/prod receive config (that's the deployment platform's job).
- `pydantic-settings` gives typed, validated configuration that fails fast at startup with a clear error if something's missing or malformed; use `SecretStr` for any credential field so it never accidentally prints/logs in plaintext.
- The answer to "avoid duplicating YAML across environments" is always some form of "one shared shape + small environment-specific values" — via env vars, overlay files, or a template — never "N nearly-identical full copies."
- A deep merge (recursing into nested dicts) is required for base+overlay to work correctly; a shallow `{**base, **overlay}` silently drops any base keys inside a sub-dict the overlay didn't fully repeat.
- Config drift is the live system silently diverging from its version-controlled source of truth — it's only caught by *continuous* checking (scheduled `terraform plan`/`driftctl`, GitOps reconciliation with self-heal, `ansible-playbook --check --diff`), not a one-time review.
- Secrets belong in environment variables or a secret manager (Vault, AWS Secrets Manager, SSM Parameter Store), injected at deploy time — never committed inside the base/overlay YAML itself, even in a "private" repo.

## Common Mistakes

- Committing a real `.env` file (with actual secrets/credentials) to version control instead of `.gitignore`-ing it and committing only a `.env.example` with placeholder keys.
- Reaching for `os.environ.get("X", "default")` scattered across a dozen files instead of one centralized, typed `Settings` class — the same config key can silently have different assumed types or defaults in different parts of the codebase.
- Printing or logging a `pydantic` settings object (or any config dict) that contains a plain `str` password/token field instead of `SecretStr` — the value ends up in application logs, and logs frequently end up somewhere less access-controlled than the secret store itself.
- Maintaining `config-dev.yaml`, `config-staging.yaml`, `config-prod.yaml` as three independently hand-edited full copies — guarantees eventual drift between them and turns "what's actually different between environments" into a manual diffing exercise instead of a one-line answer (the overlay file's contents).
- Implementing an overlay merge as a shallow `dict` update instead of a recursive deep merge, so overriding one key inside a nested section (`resources.cpu`) silently wipes out sibling keys (`resources.memory`) that the overlay never mentioned.
- Applying an emergency manual fix directly to a running server/cluster/resource and never back-porting it into version control — the next automated deploy or GitOps self-heal cycle silently reverts the fix, often during an incident when it's least expected.
- Only running `terraform plan`/drift checks immediately before a deploy, missing drift that occurred and went unnoticed in the (sometimes weeks-long) gap between deploys.
