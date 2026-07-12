# Day 31 — Quiz: Terraform CI & Advanced Patterns

Try to answer without looking at your notes. Answers are at the bottom.

1. What specific problem does Terragrunt's `remote_state` + `include` pattern solve that plain Terraform modules don't?
2. What does Terragrunt's `run-all` command do that plain Terraform has no built-in equivalent for?
3. In the standard "plan on PR, apply on merge" pattern, why should CI be the only thing with `apply`-level credentials?
4. Why is OIDC federation preferred over static AWS access keys stored as CI secrets?
5. What does `terraform plan -detailed-exitcode` return, and why is that specifically necessary for automated drift detection?
6. What is the difference between what Checkov scans and what Sentinel/OPA scan?
7. Why can a policy like "no instance type larger than m5.xlarge in dev" be hard to enforce correctly against source HCL, but straightforward against the plan JSON?
8. What's the practical difference between Sentinel and OPA/Rego, and when would you pick one over the other?
9. What does Infracost add to a PR pipeline that Checkov/OPA don't cover?
10. Why should a saved plan artifact (not a freshly re-run `plan`) be what actually gets applied on merge?
11. What is Atlantis, and what problem does it solve relative to hand-rolling the same pipeline in GitHub Actions?
12. **Interview question:** How do you prevent engineers from running `terraform apply` locally against production?

---

## Answers

1. It removes the need to repeat near-identical `backend`/`provider` boilerplate across every environment/module folder — the root `terragrunt.hcl` defines it once, and `include` + `path_relative_to_include()` lets every child unit inherit it with just its unique state `key` computed automatically from its file path.
2. `run-all` orchestrates `plan`/`apply`/`destroy` across an entire tree of separate Terragrunt units (each with its own state file), applying them in dependency order derived from `dependency` blocks. Plain Terraform's dependency graph only spans a single root module's state — it has no native way to sequence operations across multiple independent root modules/state files.
3. Because it centralizes where production changes originate: a merged, reviewed git commit with an author and approval trail — instead of an engineer's laptop with ambient credentials and no review step. It converts "who has apply access" into "what's in the audited CI pipeline," which is auditable and revocable in one place.
4. Static access keys are long-lived and have standing access until manually rotated/revoked — if leaked, they remain valid indefinitely. OIDC federation issues short-lived, workflow-scoped temporary credentials via AWS STS for each run, so there's no persistent secret to leak, and access is automatically scoped to the specific repo/branch trust policy.
5. It returns exit code `0` for "no changes," `1` for an error, and `2` for "changes detected" (drift). This is necessary because plain `terraform plan` always exits `0` on a successful run regardless of whether any changes were found — without `-detailed-exitcode`, a CI script can't distinguish "everything matches" from "drift found" using the exit code alone.
6. Checkov performs static analysis directly on the source HCL (or a plan JSON) against a library of built-in security/compliance rules, with no need to fully resolve variables — fast and runs on every PR. Sentinel/OPA evaluate the **fully-resolved plan JSON** — the actual set of resource changes after all variables, data sources, and `for_each`/`count` expansion are evaluated — which lets them enforce policies that depend on values only knowable after that resolution.
7. Source HCL might reference a variable (`instance_type = var.size`) whose actual value depends on a `.tfvars` file, environment variable, or module default not visible by reading that one line — a static rule over source text can't reliably determine the final value. The plan JSON contains the fully resolved value after all of that interpolation, so a policy checking it sees the true, final instance type regardless of how many layers of variables produced it.
8. Sentinel is HashiCorp's proprietary policy language, tightly integrated as a required gate specifically within Terraform Cloud/Enterprise. OPA/Rego is open-source and tool-agnostic — used for Terraform plans, but also Kubernetes admission control, API authorization, and general CI policy checks. Pick Sentinel if already standardized on Terraform Cloud/Enterprise; pick OPA if you want one policy language reusable across many parts of the stack.
9. Infracost estimates the dollar cost impact of a plan and posts it as a PR comment — a cost-visibility gate, distinct from Checkov/OPA's security/compliance gate. It surfaces "this PR increases monthly spend by $340" during review instead of as a surprise on the next cloud bill.
10. Because infrastructure (or the state backing it) can change between when a plan is reviewed/approved and when the merge actually triggers `apply` — re-running `plan` at apply time could compute a different diff than what a human approved. Applying the exact saved plan artifact guarantees the reviewed plan and the executed plan are identical.
11. Atlantis is a self-hosted, open-source server that automates the plan-on-PR/apply-on-comment workflow by listening to VCS webhooks directly, rather than requiring a hand-written CI workflow file per repository. It solves the "we keep reinventing the same GitHub Actions YAML for every repo" problem by centralizing the plan/apply/comment logic in one purpose-built tool.
12. Strong answer: "Structurally remove standing apply access from individuals — CI is the only identity with permission to run `terraform apply` against production, typically via a short-lived OIDC-federated role rather than static keys. Engineers get PR review authority: every change goes through `plan` on PR (posted as a comment for review), and `apply` only fires automatically after merge to main, applying the exact reviewed plan rather than a freshly recomputed one. On top of that, I'd add required status checks — Checkov/OPA policy gates — so even an approved PR can't merge if it violates a hard policy like a public S3 bucket, and I'd pair this with IAM restricting who can even assume a role capable of apply, so there's no informal side-channel around the pipeline."
