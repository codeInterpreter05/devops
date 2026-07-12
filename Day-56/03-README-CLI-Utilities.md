# Day 56 — Terraform Associate Exam Prep: CLI Utilities (taint, import, graph, fmt)

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Terraform | **Flag:** 📌

## Brief

Beyond the core `init`/`plan`/`apply`/`destroy` loop, the exam (objective domain 4, "Use the Terraform CLI outside of core workflow") tests a handful of utility commands that come up constantly in real operations: forcing a resource to be recreated, bringing existing infrastructure under management, visualizing the dependency graph, and keeping formatting/syntax consistent across a team. These are small commands individually but each solves a specific, recurring real-world problem.

## `taint` (and its modern replacement, `-replace`)

`terraform taint aws_instance.web` marks a resource in state as "tainted" — the next `apply` will destroy and recreate it, even though nothing in its HCL configuration changed. This is the tool for "this specific instance is in a bad state (corrupted disk, stuck in a weird AWS console-modified condition) and I want Terraform to replace it cleanly" without touching any other resource.

```bash
# legacy (still works, but deprecated since Terraform 0.15.2):
terraform taint aws_instance.web
terraform apply

# modern equivalent — preferred, shows up explicitly in the plan output:
terraform apply -replace="aws_instance.web"
```

**Why `-replace` replaced `taint`:** `taint` mutated state *immediately*, outside of any plan — you couldn't preview the blast radius (e.g., did that instance have dependents that would also need replacing?) before it was already marked. `-replace` is a plan-time flag: you see the replacement as part of a normal `terraform plan -replace=...` output, review it like any other change, and only then apply — consistent with Terraform's core "always show a plan before touching infrastructure" philosophy.

## `import` — bringing existing infrastructure under management

`terraform import` links a real-world resource that already exists (created by hand in a console, by a previous tool, or by another team) to a resource block in your HCL, without recreating it.

```bash
# 1. Write the resource block in HCL first (attributes can be minimal/wrong initially)
resource "aws_instance" "web" {
  # will be filled in / corrected after import
}

# 2. Import the real object's ID into that address
terraform import aws_instance.web i-0abc123456789

# 3. Run `terraform plan` — it will show a diff between your (probably incomplete) HCL
#    and the real resource's actual attributes. Fix your HCL until `plan` shows no changes.
```

**The critical gotcha:** `import` populates *state* from the real object, but does **not** generate or update your HCL configuration automatically (pre-Terraform 1.5's `import` blocks + `-generate-config-out` narrow this gap — worth knowing that newer Terraform versions can generate a starting HCL block for you, but the classic CLI-only `import` command never did). If your HCL doesn't match the real resource's attributes after import, the very next `plan`/`apply` will try to "correct" the real resource to match your (wrong) HCL — potentially destructively. Always run `plan` immediately after any `import` and treat a non-empty diff as a signal to fix your HCL, not your infrastructure.

## `graph` — visualizing the dependency graph

```bash
terraform graph | dot -Tpng > graph.png     # requires Graphviz's `dot` installed
```

Terraform builds an internal DAG (directed acyclic graph) of every resource and its dependencies (explicit, via reference, or implicit, via `depends_on`) to determine safe creation/destruction order and what can be parallelized. `terraform graph` dumps that DAG in Graphviz DOT format. This is mostly a debugging/understanding tool — useful when you're trying to figure out *why* Terraform wants to touch resources in an order that seems surprising, or to visually confirm a `depends_on` you added actually created the edge you intended.

## `fmt` and `validate`

```bash
terraform fmt                # rewrites files in the current dir to canonical style (idempotent)
terraform fmt -recursive     # apply across all subdirectories (modules, environments/)
terraform fmt -check         # exits non-zero if formatting would change anything — CI-friendly, no writes
terraform fmt -diff          # show what would change without writing

terraform validate           # checks HCL syntax + internal consistency (types, required args) — no provider calls, no credentials needed
```

The distinction that matters operationally: **`validate` is purely local and syntactic/semantic** — it catches "you forgot a required argument" or "this reference doesn't exist," but it does **not** catch things that require talking to the provider, like "this AMI ID doesn't exist" or "you don't have permission to create this resource" (those only surface at `plan`, which does refresh/query the provider). This makes `terraform validate` extremely cheap to run in CI on every commit (no cloud credentials required at all), while `terraform plan` is more expensive and requires real provider auth.

```bash
# a typical CI pre-check sequence, cheapest-first:
terraform fmt -check -recursive
terraform validate
terraform plan -out=tfplan     # only this step needs cloud credentials
```

## Points to Remember

- `taint`/`-replace` force destroy+recreate of one resource without any HCL change; prefer `-replace` (plan-time, reviewable) over legacy `taint` (immediate state mutation, no preview).
- `import` populates state only — it does not write or fix your HCL; always follow with `plan` and reconcile any diff by editing HCL, not by re-applying blindly.
- `graph` renders the internal dependency DAG (via Graphviz) — useful for debugging surprising ordering or confirming `depends_on` edges.
- `fmt` is style-only and safe/idempotent; `fmt -check` is the CI-safe variant that never writes files.
- `validate` is local-only (no provider/credentials needed) and catches syntax/type/reference errors; it cannot catch anything that requires querying the actual cloud provider — that's `plan`'s job.

## Common Mistakes

- Running `terraform import` and considering the job done without immediately running `plan` — the next real `apply` can silently "correct" the live resource to match incomplete/wrong HCL.
- Using legacy `taint` in a team setting where nobody reviews the resulting plan before applying — since `taint` mutates state immediately, a surprising destroy can happen with no review step.
- Expecting `terraform validate` to catch invalid AMI IDs, missing IAM permissions, or "resource already exists" conflicts — none of that is checked without contacting the provider, which only happens at `plan`/`apply`.
- Running `terraform fmt` (without `-check`) as a CI gate — it will happily rewrite files and "pass," silently masking the fact that a contributor's PR wasn't actually formatted correctly when they submitted it.
- Forgetting `-recursive` on `fmt` and only formatting the top-level directory, leaving nested module directories inconsistently styled.
