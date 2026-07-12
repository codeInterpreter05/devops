# Day 56 — Lab: Terraform Associate Exam Prep

**Goal:** Drill the state-management and CLI-utility commands the exam (and real Terraform debugging) leans on hardest, then sit a full timed practice exam.

**Prerequisites:** Terraform CLI installed (`terraform version`), an AWS (or any provider) sandbox account with a free-tier-safe resource type to experiment on (a couple of `aws_instance` `t3.micro`/`aws_s3_bucket` resources are fine — or use the `local` / `random` providers if you want zero cloud cost for most of this lab).

---

### Lab 1 — State surgery: `mv`, `rm`, `show`, `list`

1. In a scratch directory, write a minimal config with two resources using the `random` provider (no cloud cost):
   ```hcl
   resource "random_pet" "example" { length = 2 }
   resource "random_id" "example"  { byte_length = 4 }
   ```
2. `terraform init && terraform apply -auto-approve`
3. `terraform state list` — confirm both addresses appear.
4. `terraform state show random_pet.example` — inspect its full state.
5. Rename `random_pet.example` to `random_pet.name_tag` in the HCL. Run `terraform plan` first — observe it proposes a destroy+create. Then instead run:
   ```bash
   terraform state mv random_pet.example random_pet.name_tag
   terraform plan     # should now show NO changes
   ```
6. Remove `random_id.example` from state without destroying it: `terraform state rm random_id.example`, then run `terraform plan` — note Terraform now wants to *create* a new `random_id.example` because your HCL still declares it but state no longer tracks it. Explain in one sentence why that happened.

**Success criteria:** You can demonstrate, live, the destroy/recreate that a plain rename causes versus the no-op that `state mv` achieves, and can articulate why `state rm` orphans a HCL-declared resource.

---

### Lab 2 — `import` an existing resource, correctly

1. Create a resource by hand, outside Terraform (e.g., `aws s3api create-bucket --bucket my-manual-bucket-<yourname> --region us-east-1`, or any resource type you have sandbox access to).
2. Write a **minimal/empty** `aws_s3_bucket` resource block in a new Terraform config for it.
3. `terraform import aws_s3_bucket.manual my-manual-bucket-<yourname>`
4. Run `terraform plan` immediately — read the diff carefully; it will show every attribute Terraform thinks needs to change to match your (currently incomplete) HCL.
5. Iterate: add real attributes to your HCL (tags, versioning config, etc.) matching the bucket's actual settings until `terraform plan` shows "No changes."

**Success criteria:** `terraform plan` reports no drift after reconciling your HCL — proving you understand `import` populates state only, and getting HCL to match is your job.

---

### Lab 3 — `-replace`, `graph`, `fmt`, `validate`

1. Using Lab 1's `random_pet` resource, force a recreate without any config change:
   ```bash
   terraform plan -replace="random_pet.name_tag"
   terraform apply -replace="random_pet.name_tag"
   ```
   Confirm the pet name value changed after apply.
2. Generate a dependency graph: `terraform graph | dot -Tpng > graph.png` (install graphviz first if needed: `apt install graphviz` / `brew install graphviz`). Open the PNG and identify the nodes/edges for your two resources.
3. Deliberately misformat a `.tf` file (bad indentation, inconsistent spacing), then run:
   ```bash
   terraform fmt -check -diff     # should exit non-zero and show the diff
   terraform fmt -recursive        # fix it
   terraform fmt -check -diff     # should now exit 0
   ```
4. Introduce a deliberate HCL error (reference a variable that doesn't exist), run `terraform validate`, read the error, then fix it and confirm `validate` passes. Note that you never needed cloud credentials for any part of this step.

**Success criteria:** You can explain why `validate` caught the bad reference but would *not* catch, e.g., an invalid AMI ID — and you have a rendered graph image you can read.

---

### Lab 4 — Terraform Cloud remote state (free tier) + a Sentinel thought exercise

1. Sign up for Terraform Cloud's free tier (app.terraform.io) if you haven't; create an organization and a workspace.
2. Configure your Lab 1 config to use the `remote` backend pointing at that workspace, `terraform init` (it will offer to migrate local state to TFC — accept), then `terraform apply` and watch the run execute remotely in the TFC UI instead of your terminal.
3. Without needing a paid Sentinel-capable tier, write out (as a text file, not something you can execute) a Sentinel policy sketch that would block creating any `aws_instance` with an instance type outside an approved list (`t3.micro`, `t3.small`). Note which enforcement level (advisory / soft-mandatory / hard-mandatory) you'd choose for a *first* rollout of this policy, and justify it in one sentence.

**Success criteria:** A real `apply` you can point to in the TFC UI showing remote execution, plus a written Sentinel policy sketch with a justified enforcement level choice.

---

### Lab 5 — The core hands-on activity: full Terraform Associate practice exam

1. Take the official **HashiCorp Terraform Associate practice exam** (or a reputable third-party question bank), timed.
2. Specifically flag every question touching state management (`mv`/`rm`/`import`/locking/remote backends) — these are meant to be your strongest category after this lab; if you miss any, that's a signal to redo Lab 1-2.
3. Write down your weakest domain (of the 9) and pick one concrete resource to study it further before your next attempt.

**Success criteria:** A completed practice exam with a score, plus a one-line, honest note on your single weakest objective domain.

---

### Cleanup

```bash
terraform destroy -auto-approve   # in each lab directory
aws s3 rb s3://my-manual-bucket-<yourname> --force   # if you created the manual S3 bucket
rm -f graph.png
```

### Stretch challenge

Split your Lab 1 resources into two separate Terraform configurations/state files ("layers"), then wire the second to read an output from the first using `terraform_remote_state` — proving you can operate a multi-layer state architecture, not just a single flat config.
