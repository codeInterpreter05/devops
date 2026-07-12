# Day 31 — Lab: Terraform CI & Advanced Patterns

**Goal:** Build a real GitHub Actions pipeline that runs `terraform fmt` → `validate` → `plan` on every PR and posts the plan as a PR comment — this day's assigned hands-on activity — then layer in Checkov scanning and drift detection.

**Prerequisites:**
- A GitHub repository (can be a new throwaway repo) containing the Terraform config from Day 29/30.
- An AWS account with permission to set up an IAM OIDC identity provider and an IAM role (or, if you want to skip OIDC setup for speed, a scoped IAM user's access keys stored as repo secrets — note in your own words why OIDC is preferred).
- `pip install checkov` locally for the pre-check step.

---

### Lab 1 — The core hands-on activity: PR plan pipeline

1. In your repo, create `.github/workflows/terraform.yml` with the `pull_request`-triggered job from `01-README-Terraform-In-CI.md` (fmt check, init, validate, plan, post-comment step). Start with static AWS credentials as GitHub Actions secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) to get it working first.
2. Create a branch, make a small Terraform change (e.g., add a tag to the VPC), and open a PR.
3. Watch the Actions tab — confirm `fmt`, `validate`, and `plan` all run, and that a comment with the plan output appears on the PR.
4. Intentionally break formatting (extra spaces) and push again — confirm `terraform fmt -check` fails the job before `plan` even runs.

**Success criteria:** A real PR in your repo has an automatically-posted comment showing the exact `terraform plan` diff for your change.

---

### Lab 2 — Switch to OIDC federation

1. Create an IAM OIDC identity provider for GitHub Actions:
   ```bash
   aws iam create-open-id-connect-provider \
     --url https://token.actions.githubusercontent.com \
     --client-id-list sts.amazonaws.com \
     --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
   ```
2. Create an IAM role with a trust policy scoped to your specific repo/branch, and attach a policy allowing the Terraform actions you need (start broad for the lab, e.g. `AmazonVPCFullAccess` + `AmazonEC2FullAccess`; note in your own words why you'd scope this far tighter in real production).
3. Update the workflow to use `aws-actions/configure-aws-credentials@v4` with `role-to-assume` instead of static secrets, and remove the static secrets from the repo.
4. Re-run the PR pipeline — confirm it authenticates successfully with no long-lived keys involved.

**Success criteria:** The pipeline runs with zero static AWS credentials stored anywhere in GitHub.

---

### Lab 3 — Add Checkov as a required gate

1. Add a Checkov step before `terraform plan` in the workflow:
   ```yaml
   - name: Checkov scan
     run: |
       pip install checkov
       checkov -d infra/ --framework terraform --compact
   ```
2. Deliberately introduce a finding — add a security group rule with `cidr_blocks = ["0.0.0.0/0"]` on port 22 — and push it in a new PR. Confirm Checkov fails the job with a specific `CKV_AWS_24` (or similar) finding.
3. Fix it (restrict to your IP), confirm the job goes green.
4. In your repo settings, mark the Checkov job as a **required status check** on the main branch so the PR literally cannot merge while it's red.

**Success criteria:** A deliberately inserted security misconfiguration is caught and blocks merge before you fix it.

---

### Lab 4 — Apply on merge

1. Extend the workflow with the `push`-to-`main`-triggered `apply` job, using a saved plan artifact passed between the `plan` job (on PR) and the `apply` job (on merge) via `actions/upload-artifact` / `download-artifact`, so the exact reviewed plan is what gets applied.
2. Merge a small, safe PR (e.g., add another tag) and watch the `apply` job run automatically on `main`, using the same plan that was posted on the PR.
3. Confirm in the AWS console that the change took effect, with zero manual `terraform apply` commands run from your own machine.

**Success criteria:** A merge to `main` triggers an automatic apply of the exact previously-reviewed plan, with no human running Terraform locally.

---

### Lab 5 — Scheduled drift detection

1. Add a second workflow, `.github/workflows/drift-check.yml`, on a `schedule` cron trigger, running `terraform plan -detailed-exitcode` and treating exit code `2` as "drift found."
2. Manually change something in the AWS console that Terraform manages (e.g., add a tag to the VPC by hand).
3. Manually trigger the drift-check workflow (`workflow_dispatch` for testing) and confirm it detects and reports the drift.
4. Revert the manual change (or run `terraform apply` to reconcile it back).

**Success criteria:** You've built and personally triggered a working drift detector that distinguishes "no changes" from "changes found" using the exit code, not by parsing text output.

---

### Cleanup

```bash
terraform destroy
# In GitHub: delete the OIDC-based IAM role and identity provider if this was a throwaway account
aws iam delete-role --role-name gha-terraform-role
aws iam delete-open-id-connect-provider --open-id-connect-provider-arn <ARN>
```

### Stretch challenge

Add an Infracost step to the PR pipeline that posts the monthly cost delta of the plan as a second PR comment, and add one Sentinel-style (or OPA/Rego, if using a local `conftest` setup against the plan JSON) policy that blocks any `aws_db_instance` resource with `publicly_accessible = true`.
