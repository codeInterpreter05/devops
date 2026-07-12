# Day 47 — Quiz: Terraform on AWS Full Project

Try to answer without looking at your notes. Answers are at the bottom.

1. What is the dependency order between the VPC, EKS cluster, node groups, and IRSA layers in a Terraform EKS project, and why does that order matter?
2. What are the two specific subnet tags the AWS Load Balancer Controller needs to auto-discover subnets, and what happens if they're missing?
3. Mechanically, how does IRSA let a pod assume an IAM role without static credentials?
4. Why is the ALB itself not created directly by Terraform in this architecture — what creates it instead?
5. What are the three fatal problems with local Terraform state once more than one person/process is involved?
6. What does the DynamoDB lock table actually do, and what's the newer alternative that doesn't require it?
7. Why is "one giant state file for everything" considered a structural mistake, even if it technically applies successfully?
8. What's the difference between Terraform workspaces and separate environment directories for isolating dev/staging/prod, and why do many teams prefer the latter for production?
9. What's the core principle both Atlantis and a GitHub Actions apply flow implement, regardless of tooling?
10. What GitHub-native mechanism gates a `terraform apply` job behind human approval in a GitHub Actions workflow?
11. Does marking a Terraform output `sensitive = true` encrypt it? What does it actually do, and what's the real control for protecting secret values?
12. **Interview question:** Walk me through your Terraform project structure for a multi-environment EKS setup.

---

## Answers

1. VPC must exist first (EKS needs subnets to place the cluster/nodes in); the EKS cluster (and its OIDC provider) must exist before IRSA roles can be created (IRSA roles trust that specific OIDC provider); node groups depend on the cluster existing. Terraform mostly infers this from resource/module references (implicit dependency graph), but getting the layering conceptually right matters for reasoning about partial applies and for structuring modules correctly.
2. `kubernetes.io/role/elb = 1` on public subnets and `kubernetes.io/role/internal-elb = 1` on private subnets (plus the shared cluster tag). If missing, the AWS Load Balancer Controller can't auto-discover which subnets to place a new ALB's ENIs in, resulting in an `Ingress` that never gets an address, often with no obviously pointed error message.
3. The EKS cluster's OIDC provider is trusted by an IAM role's trust policy, scoped via a condition matching a specific namespace+ServiceAccount in the OIDC subject claim. The EKS Pod Identity webhook injects a projected service account token and environment variables into the pod, which the AWS SDK automatically uses to assume that IAM role for temporary, scoped credentials — no static access keys involved.
4. Because the AWS Load Balancer Controller (a Kubernetes controller, deployed via Helm) is what actually creates and manages ALB/NLB resources dynamically, in response to Kubernetes `Ingress`/`Service type=LoadBalancer` objects. Terraform's role is limited to provisioning the IAM permissions (via IRSA) and VPC subnet tags the controller needs — not the load balancer itself.
5. No sharing (other users/processes don't see your local state, so Terraform thinks resources you created don't exist), no locking (concurrent applies against the same file can race/corrupt it), and no durability (a single deleted or corrupted local file loses all tracking of what Terraform manages, recoverable only via manual `terraform import`).
6. Before a state-modifying operation, Terraform writes a lock record to the DynamoDB table; a concurrent operation against the same state fails immediately with a clear lock error instead of racing against the first one. Newer Terraform/OpenTofu versions can achieve equivalent locking using S3's native conditional writes (`use_lockfile = true`) without a separate DynamoDB table.
7. Because it maximizes blast radius (an issue with any single resource in that state can block plan/apply for everything else in it) and slows down every `plan`, since Terraform refreshes/checks the status of every resource in the state file regardless of how small the actual change is.
8. Workspaces give multiple state files under one backend configuration, switched via a CLI command (`terraform workspace select`) — which environment you're targeting is invisible ambient session state, easy to get wrong by mistake. Separate directories per environment make the target environment explicit and visible from which directory you're standing in, which is why most production setups prefer directories over workspaces despite workspaces being more convenient for quick throwaway use.
9. Both implement "generate the plan automatically and make it visible in a pull request for human review; apply only happens after explicit approval, typically gated to occur after merge" — turning infrastructure changes into a reviewed, auditable process like application code changes, regardless of which specific tool enforces it.
10. A GitHub `environment` configured with required reviewers — the apply job references that environment, and GitHub blocks the job from running until an authorized reviewer approves it.
11. No — `sensitive = true` only suppresses the value from CLI output and CI logs; the value is still stored in plaintext within the Terraform state file itself. The real control for protecting secret values is state file encryption at rest and tightly scoped access control on the state backend (e.g., the S3 bucket and its bucket policy/KMS key), not the `sensitive` flag alone.
12. Strong answer: "I isolate state per environment and per logical component — separate directories (`environments/dev`, `environments/prod`) each with their own backend `key`, rather than Terraform workspaces, so the target environment is explicit and visible. Each environment composes a shared local module (or directly composes `terraform-aws-modules/vpc` and `.../eks`) with environment-specific variables, keeping the VPC, EKS cluster, and dependent components like RDS in separate state files to limit blast radius and keep plans fast. Cross-component references go through `terraform_remote_state` or explicit output-passing, not hardcoded IDs. Changes go through a PR with an automatic `plan` posted for review — either via Atlantis or a GitHub Actions workflow gated by a GitHub environment approval — with apply happening only after merge, so infrastructure changes get the same review rigor as application code."
