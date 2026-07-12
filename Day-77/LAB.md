# Day 77 — Lab: Multi-cloud CI/CD

**Goal:** Extend a CI/CD pipeline to deploy a minimal app to both AWS (Lambda) and GCP (Cloud Run) from a single build, using Terraform with multiple provider blocks, and produce a written cost/benefit assessment of the exercise itself.

**Prerequisites:**
- AWS account + GCP project, both with billing enabled (free-tier eligible for this exercise's scale).
- Terraform >= 1.5, AWS CLI, `gcloud` CLI, Docker.
- GitHub repo with Actions enabled, OIDC trust configured for both AWS (`aws-actions/configure-aws-credentials`) and GCP (`google-github-actions/auth` with Workload Identity Federation) — set these up once, they're reusable beyond this lab.

---

### Lab 1 — Provision both clouds with one Terraform config

1. Write `providers.tf` declaring both `aws` and `google` providers (see `02-README-Terraform-Workspaces-Multicloud.md`).
2. Define a minimal `aws_lambda_function` (container-image-based) and a minimal `google_cloud_run_v2_service`, both deploying the same placeholder image (e.g., a "hello world" HTTP handler).
3. Configure a single remote backend (S3 + DynamoDB, or Terraform Cloud) for state.
4. Run `terraform init && terraform plan` and confirm the plan shows resources for **both** providers in one plan output.
5. `terraform apply` and confirm both endpoints are live (`curl` the Lambda's function URL and the Cloud Run service URL).

**Success criteria:** One `terraform apply` provisions working compute on both AWS and GCP, tracked in one state file.

---

### Lab 2 — Add environment separation with workspaces

1. Create `staging` and `production` workspaces.
2. Parameterize replica count / memory size per workspace (e.g., `terraform.workspace == "production" ? 512 : 128` for Lambda memory).
3. Apply to `staging`, confirm resources, then switch to `production` and apply, confirming **separate** state and resources exist for each.

**Success criteria:** You can explain, and demonstrate via `terraform workspace list` and `terraform state list` per workspace, that workspaces isolate state/environment while the same provider blocks target both clouds in each.

---

### Lab 3 — Build the CI/CD pipeline: build once, deploy to both

1. Add the `build`, `deploy-aws-lambda`, and `deploy-gcp-cloudrun` jobs from `01-README-Multi-Cloud-Pipeline-Design.md` to a GitHub Actions workflow.
2. Push a change to your app. Confirm the Actions UI shows `build` complete, then `deploy-aws-lambda` and `deploy-gcp-cloudrun` running **in parallel** (not sequentially).
3. Verify the deployed digest is identical on both clouds:
   ```bash
   aws lambda get-function --function-name myapp --query 'Code.ImageUri'
   gcloud run services describe myapp --region=us-central1 --format='value(spec.template.spec.containers[0].image)'
   ```

**Success criteria:** Both cloud deployments reference the exact same image digest, built exactly once, deployed in parallel via OIDC federation (no static cloud credentials in secrets).

---

### Lab 4 — Break something and observe the blast radius

1. Deliberately misconfigure the GCP deploy job (e.g., wrong region) and push. Confirm `deploy-aws-lambda` still succeeds independently — the two deploy jobs should have no unintended coupling.
2. Fix it, redeploy, confirm both succeed again.

**Success criteria:** You can demonstrate that a failure in one cloud's deploy job doesn't block or fail the other, proving genuine job independence.

---

### Lab 5 — Write the cost/benefit assessment (today's real deliverable)

1. Based on Labs 1–4, write a half-page assessment answering: how much *additional* engineering time did the GCP-specific pieces (IAM, Workload Identity setup, Cloud Run specifics) take beyond what you already knew from AWS? What would ongoing on-call/maintenance cost look like if this were a real production system? Under what specific circumstance would this dual-cloud setup have been worth that cost for a real team?
2. Reference `03-README-Cost-Repatriation.md`'s framing directly in your answer.

**Success criteria:** A written, specific (not generic) assessment grounded in what you actually experienced setting this up today — not a restatement of the README.

---

### Cleanup

```bash
terraform workspace select staging
terraform destroy
terraform workspace select production
terraform destroy
terraform workspace select default
terraform workspace delete staging production
```

### Stretch challenge

Add a third job, `deploy-gcp-cloudrun-canary`, that deploys the new revision to Cloud Run with 10% traffic split (`gcloud run services update-traffic myapp --to-revisions=LATEST=10`) while AWS Lambda deploys fully — then write one paragraph on why "the same deployment strategy" is often hard to keep genuinely equivalent across two clouds even when the pipeline structure looks symmetric.
