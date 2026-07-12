# Day 29 — Quiz: Terraform Fundamentals

Try to answer without looking at your notes. Answers are at the bottom.

1. What does "declarative" mean in the context of Terraform, and why does it make `apply` idempotent?
2. What is the difference between a resource's local name (e.g. `aws_vpc.main`) and the actual resource it represents in AWS?
3. What's the difference between a `variable` and a `local` in Terraform?
4. Name three things Terraform's state file is used for.
5. Why should the state file never be committed to version control?
6. What three things does `terraform plan` compare against each other to compute its diff?
7. What happens to already-created resources if `terraform apply` fails partway through (e.g., resource #5 of 10 hits a permissions error)?
8. What do Terraform workspaces actually isolate — configuration, state, or both? Why does this distinction matter for a dev/prod setup?
9. Why does HashiCorp warn against routine use of `terraform apply -target`?
10. What is `.terraform.lock.hcl` for, and why is it committed while `.terraform/` is not?
11. Why does OpenTofu exist? Is it a technical fork or a licensing fork?
12. **Interview question:** What is Terraform state and what problems can arise from it in a team environment?

---

## Answers

1. Declarative means you describe the *desired end state* (what should exist), not the steps to get there. Terraform's engine is responsible for diffing desired state against current state and computing the minimal set of actions to reconcile them. This makes `apply` idempotent: running it again with no config changes produces "no changes" because the desired and actual states already match — unlike an imperative script that would blindly re-run every step.
2. The local name is a reference used only inside that Terraform configuration to address the resource in HCL (`aws_vpc.main.id`) and has no relationship to the cloud object's actual name, ID, or tags. The real AWS resource is identified by state, which maps the local name to the actual resource ID (`vpc-0abc123`).
3. A `variable` is part of a module's external interface — it's set by whoever calls the module (CLI flag, tfvars file, environment variable, or its own default). A `local` is computed internally from an expression and cannot be set from outside — it exists purely to avoid repeating logic within the same module (DRY).
4. Any three of: mapping resource addresses to real-world IDs; caching resource attribute metadata to avoid redundant API calls; recording the dependency graph so destroy order is correct; enabling `plan` to diff config vs. last-known vs. live reality.
5. Because it stores **plaintext** copies of any sensitive attribute a resource has ever had (generated passwords, private keys, sensitive outputs) — committing it to git permanently exposes those secrets in history, and it's also prone to write conflicts since Terraform doesn't merge state like git merges text.
6. Your `.tf` configuration (desired state), the state file (last-known state), and a live refresh of the actual infrastructure via the provider's API (current reality).
7. The resources that were successfully created before the failure remain created in real infrastructure **and** are recorded in state — `apply` updates state incrementally, resource by resource, not as one atomic all-or-nothing transaction. Fixing the underlying error and re-running `apply` resumes from where it left off rather than starting over.
8. Workspaces isolate **state only**, not configuration or variable values. This matters because a naive dev/prod setup using just workspaces will still load the same `.tf` files and the same default variable values in both — you must still supply environment-specific `-var-file`s (or use `terraform.workspace` conditionals) or you'll silently apply the wrong sizing/config to production.
9. Because `-target` only computes and applies a plan for the targeted resource(s) and their dependencies, ignoring the rest of the configuration — this can leave state and reality in a partially-applied condition that a subsequent full `plan` finds surprising, and it's easy to miss a dependent resource that also needed the same targeting.
10. `.terraform.lock.hcl` records the exact provider versions (and their checksums) resolved during `init`, guaranteeing every engineer and CI run uses identical provider code even if the version constraint (`~> 5.0`) would technically allow multiple versions. It's committed for the same reason a `package-lock.json` is. `.terraform/` contains the actual downloaded provider binaries and module caches — large, OS/architecture-specific, and fully reproducible from the lock file, so it's gitignored.
11. OpenTofu exists because HashiCorp changed Terraform's license in 2023 from the open-source MPL 2.0 to the more restrictive Business Source License (BUSL), which limited competing commercial products built on top of Terraform. It is a **licensing fork**, not a technical disagreement — OpenTofu forked the last MPL-licensed codebase and (at least initially) remained command- and syntax-compatible.
12. Strong answer: "Terraform state is the JSON record mapping my configuration's resources to their real-world IDs and last-known attributes — it's how Terraform knows what it manages, since cloud APIs don't expose 'which Terraform run created this.' In a team environment, the main problems are: (1) concurrent applies corrupting a shared state file if there's no locking; (2) the state file containing plaintext secrets, so access to it needs to be as tightly controlled as production credentials; (3) drift, where someone changes a resource by hand and state goes stale until the next plan/refresh; and (4) if state is lost entirely, Terraform doesn't lose the real infrastructure but does lose track of it, requiring painful `terraform import` recovery. This is exactly why teams move to a remote backend with locking (S3 + DynamoDB, or Terraform Cloud) instead of a local state file — which is tomorrow's topic."
