# Day 29 — Terraform Fundamentals: The plan/apply/destroy Workflow & Tooling

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** Terraform | **Flag:** ⚡ Interview-critical

## Brief

Everything in Terraform funnels through three commands: `plan`, `apply`, `destroy`. Understanding exactly what each one does — and does *not* do — is what lets you use Terraform confidently against real infrastructure instead of being afraid of it. This file also covers the tooling layer around Terraform itself: `tfenv` for managing multiple Terraform versions, and OpenTofu, the open-source fork that exists because of a 2023 licensing change you should be able to speak to in an interview.

## The core loop

```bash
terraform init      # download providers/modules, set up backend
terraform fmt        # normalize formatting
terraform validate   # syntax + internal consistency check (no API calls)
terraform plan        # compute and show the diff — READ ONLY
terraform apply        # execute the plan — makes real changes
terraform destroy       # tear down everything this config manages
```

### `terraform init`

Downloads the providers and modules referenced in your config into `.terraform/`, and configures the backend (where state lives). You re-run `init` whenever you add a provider/module, change the backend config, or clone the repo fresh. `init -upgrade` allows provider versions to move within their constraint (respects the `~>`/`>=` bounds in `required_providers`).

### `terraform plan` — the most important command you'll run

```bash
terraform plan -out=tfplan
```

`plan` does three things: refreshes its view of real infrastructure (unless `-refresh=false`), diffs that + your config against state, and prints a human-readable summary using `+`/`-`/`~` prefixes (create / destroy / update-in-place). **It makes no changes.** This is why disciplined teams treat "did you read the plan output" as a hard gate before any `apply` — the plan is the last chance to catch "this is going to destroy and recreate my production RDS instance" before it happens.

Saving a plan with `-out` and applying *that exact file* (`terraform apply tfplan`) guarantees the plan you reviewed is the plan that executes — without `-out`, a second `plan` computed at `apply` time could differ if the real infrastructure changed in between (someone else applied, or drift occurred).

### `terraform apply`

```bash
terraform apply                 # interactive, shows plan, asks y/n
terraform apply -auto-approve   # no prompt — CI/CD use only, never for humans against prod
terraform apply tfplan          # apply a specific saved plan file
```

`apply` re-computes (or reuses a saved) plan, then executes each action against the provider's API in dependency order, updating state as it goes — **incrementally**, resource by resource, not as one atomic transaction. This matters: if `apply` fails partway through (API rate limit, a permissions error on resource #12 of 20), the first 11 resources are genuinely created and already recorded in state; you fix the error and re-`apply`, and Terraform picks up where it left off rather than starting over.

### `terraform destroy`

```bash
terraform destroy
terraform destroy -target=aws_instance.web   # destroy just one resource (use sparingly — see below)
```

Computes a plan where everything in scope is marked for deletion, in reverse dependency order, and executes it. **There is no confirmation beyond the interactive prompt** — in an automated context this is one of the most dangerous single commands in the entire DevOps toolchain. `-target` scopes it to specific resources but HashiCorp explicitly warns against relying on `-target` for routine use — it can leave a config and its state in a **surprising, partially-applied condition** that a subsequent full `plan` has to reconcile, and it's easy to forget a dependent resource that also needed targeting.

## OpenTofu — why a fork exists

In mid-2023, HashiCorp changed Terraform's license from Mozilla Public License 2.0 to the Business Source License (BUSL) — which restricts competing commercial offerings (like Spacelift, Scalr, env0 building competing products on top of it). The Linux Foundation, backed by those same companies plus many large users, forked the last MPL-licensed version into **OpenTofu**, a fully open-source, drop-in-compatible alternative.

Practically, for day-to-day use: OpenTofu's CLI commands, HCL syntax, and provider ecosystem are (as of today) essentially identical to Terraform's — `tofu plan`/`tofu apply` instead of `terraform plan`/`terraform apply`. The interview-relevant point isn't syntax, it's understanding *why* it exists (license divergence, not a technical fork) and that it signals the broader industry trend of hedging against vendor lock-in on core infra tooling — the same story as Kubernetes vs. proprietary orchestration.

## `tfenv` — managing multiple Terraform versions

```bash
tfenv install 1.7.5
tfenv install latest
tfenv use 1.7.5
tfenv list
echo "1.7.5" > .terraform-version   # auto-select this version when tfenv is on PATH
```

Different repos on the same laptop often require different Terraform versions (an old module pinned to `< 1.3`, a new one needing `>= 1.6` for `moved` blocks). `tfenv` (mirroring `rbenv`/`pyenv`/`nvm` for their respective languages) lets you install multiple versions side by side and switch per-project via a `.terraform-version` file — avoiding "works on my machine, breaks in CI because CI has a different Terraform binary" bugs.

## Points to Remember

- `plan` is read-only and safe to run as often as you want; `apply` and `destroy` make real changes — never confuse the two under time pressure.
- Save plans with `-out` and apply that exact file in CI to guarantee what was reviewed is what executes.
- `apply` proceeds resource-by-resource and updates state incrementally — a partial failure is recoverable by re-running, not a reason to panic or manually fix state.
- `-target` is an escape hatch for emergencies, not a routine workflow — it can leave state in a condition that surprises the next full `plan`.
- OpenTofu exists because of a licensing change (MPL → BUSL), not a technical disagreement — know this distinction for interviews.

## Common Mistakes

- Running `terraform apply -auto-approve` as a personal habit to save a keystroke, then one day auto-approving a plan that would have destroyed a production database had a human actually read it.
- Treating `-target` as a normal way to "just fix this one resource," accumulating a config where state and reality have quietly diverged from what a full `plan` would show.
- Not saving/reusing a plan file (`-out`) in CI pipelines, so the plan a human approved in a PR comment isn't necessarily the plan that gets applied minutes later if infrastructure changed in between.
- Running `terraform destroy` in the wrong directory/workspace because of an un-set or wrong `TF_WORKSPACE`/`terraform workspace select`, destroying the wrong environment's resources.
- Assuming OpenTofu and Terraform will always stay drop-in compatible forever — the ecosystems can and likely will diverge over time as each adds features independently; pin and test explicitly rather than assuming.
