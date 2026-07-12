# Day 77 — Resources: Multi-cloud CI/CD

## Primary (assigned)

- **HashiCorp: Multi-cloud blog posts** (hashicorp.com/blog, search "multi-cloud") — the assigned starting point. HashiCorp's own material is a natural fit here since Terraform is the primary tool that makes multi-cloud infrastructure-as-code tractable; their posts cover both the technical patterns and the organizational reasoning.

## Deepen your understanding

- **Terraform docs — "State: Workspaces"** (developer.hashicorp.com/terraform/language/state/workspaces): the authoritative clarification of what workspaces do and don't do — read this directly if the "workspaces isolate state, not providers" distinction in file 2 still feels fuzzy.
- **Terraform docs — "Provider Configuration"** (developer.hashicorp.com/terraform/language/providers/configuration): reference for declaring and aliasing multiple provider instances (including multiple instances of the *same* provider, e.g., two AWS regions, which is a related but distinct pattern from multi-cloud).
- **"Multi-Cloud: How to Approach it Correctly" and similar posts from the CNCF blog** (cncf.io/blog): CNCF-adjacent perspective on where Kubernetes genuinely helps with cross-cloud portability and where the abstraction leaks.
- **Andreessen Horowitz — "The Cost of Cloud, a Trillion Dollar Paradox"** (a16z.com): the widely cited analysis behind the cloud-repatriation conversation — makes the steady-state-cost argument in file 3 with real industry figures and case studies.

## Reference

- **`aws-actions/configure-aws-credentials` README** (github.com/aws-actions/configure-aws-credentials): OIDC role-assumption configuration reference used in this day's pipeline examples.
- **`google-github-actions/auth` README** (github.com/google-github-actions/auth): Workload Identity Federation setup reference for GCP, the GCP-side equivalent of the AWS OIDC pattern.

## Practice

- **Deploy the same "hello world" container to AWS Lambda and GCP Cloud Run manually first** (before automating it in CI) — doing the manual `aws lambda create-function` / `gcloud run deploy` once each surfaces the real IAM/networking differences between the two clouds before you also have to debug a pipeline on top of it.
- **Terraform Registry — browse both the `hashicorp/aws` and `hashicorp/google` provider docs side by side** for the same resource type (e.g., compare `aws_lambda_function` to `google_cloudfunctions2_function`) to get a concrete feel for how much provider-specific detail doesn't map cleanly one-to-one, which is the practical root of most multi-cloud engineering cost.
