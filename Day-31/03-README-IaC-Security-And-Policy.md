# Day 31 — Terraform CI & Advanced Patterns: IaC Security Scanning & Policy-as-Code

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** Terraform | **Flag:** —

## Brief

A PR review catches "does this Terraform code do what the author intended." It does **not** reliably catch "does this Terraform code violate a security or cost policy the reviewer didn't happen to think of" — a human skimming a 200-line diff will miss an unencrypted S3 bucket or an overly permissive security group far more often than a tool built to check for exactly that. **Checkov** (static security scanning) and **Sentinel/OPA** (policy-as-code gating) automate that check as a required CI step, turning "please remember to check for public buckets" into "the pipeline physically cannot merge/apply a policy violation."

## Checkov — static analysis for IaC

```bash
pip install checkov
checkov -d infra/ --framework terraform
```

Checkov parses your `.tf` files (or a `terraform plan` JSON output) **without ever touching a real cloud API** and checks them against hundreds of built-in rules mapped to real security frameworks (CIS Benchmarks, NIST, PCI-DSS):

```
Check: CKV_AWS_18: "Ensure the S3 bucket has access logging configured"
        FAILED for resource: aws_s3_bucket.data
        File: /main.tf:12-18

Check: CKV_AWS_24: "Ensure no security groups allow ingress from 0.0.0.0/0 to port 22"
        PASSED for resource: aws_security_group.web_sg
```

Because it's purely static analysis, Checkov runs in milliseconds against source code — cheap enough to run on every single PR, long before any `terraform plan` touches a cloud API. This is the practical answer to *catching* the kind of mistake in this day's interview question (an S3 bucket accidentally made public) before it's ever applied, not just after the fact.

```yaml
# in the GitHub Actions workflow, before terraform plan
- name: Checkov scan
  run: checkov -d infra/ --framework terraform --compact --quiet
```

A failed Checkov check should **block the PR from merging** (via a required status check), the same way a failing unit test would — treating IaC security posture as a build gate rather than a suggestion. Real-world nuance: Checkov (and similar tools like `tfsec`/`trivy`) will have false positives for intentional exceptions (a genuinely public static-website bucket); use inline suppression comments (`#checkov:skip=CKV_AWS_20:Public website bucket, intentional`) sparingly and with a comment explaining *why*, so suppressions are visible in review rather than silent.

## Sentinel / OPA — policy-as-code, evaluated against the *plan*

Checkov looks at your source code. **Sentinel** (HashiCorp's policy language, used in Terraform Cloud/Enterprise) and **OPA/Rego** (Open Policy Agent, the open-source, ecosystem-wide equivalent) instead evaluate the **generated plan JSON** — meaning they see the *actual, fully-resolved* set of changes Terraform is about to make, including values only known after variable interpolation, data source lookups, and `for_each`/`count` expansion. This distinction matters: a rule like "no instance type larger than `m5.xlarge` in the `dev` account" is much easier to enforce correctly against the fully-resolved plan than against source HCL, which may reference a variable whose actual value isn't visible without evaluating the whole config.

```python
# Sentinel policy example (conceptual)
import "tfplan/v2" as tfplan

no_public_s3 = rule {
    all tfplan.resource_changes as _, rc {
        rc.type is "aws_s3_bucket_public_access_block" or
        rc.type is not "aws_s3_bucket"
    }
}

main = rule { no_public_s3 }
```

```rego
# OPA/Rego equivalent (conceptual)
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_security_group_rule"
    resource.change.after.cidr_blocks[_] == "0.0.0.0/0"
    resource.change.after.from_port == 22
    msg := sprintf("SSH open to the world: %s", [resource.address])
}
```

Sentinel is tightly integrated with Terraform Cloud/Enterprise (runs automatically as a required gate between `plan` and `apply` in that platform). OPA is tool-agnostic and shows up far beyond Terraform — Kubernetes admission control, API gateways, CI pipelines generally — so if you're only going to deeply learn one policy language for a broad DevOps career, OPA/Rego has the wider applicability; Sentinel is the right choice specifically if your org is already committed to Terraform Cloud/Enterprise.

## Infracost — the cost dimension of the same PR gate

```bash
infracost breakdown --path infra/
infracost diff --path infra/ --compare-to infracost-base.json
```

Infracost estimates the dollar cost of a Terraform plan **before** apply, and (like Checkov) is commonly wired into the same PR pipeline to post "this PR increases monthly infra spend by $340" as a comment. It doesn't block by default the way a security policy might, but it turns cost from an invisible, discovered-on-the-monthly-bill surprise into a visible part of code review — the same "shift left" idea applied to cost instead of security.

## Putting it together — the full gate

```
PR opened
  → terraform fmt -check, validate
  → Checkov (static security scan of source)
  → terraform plan (generates plan JSON)
  → OPA/Sentinel (policy check against the resolved plan)
  → Infracost (cost delta posted as comment)
  → all required checks green → human review of the diff + plan → merge
  → CI applies on main
```

## Points to Remember

- Checkov scans your **source HCL** statically — fast, cheap, runs on every PR without touching cloud APIs or needing a real plan.
- Sentinel/OPA evaluate the **generated plan JSON** — the fully-resolved set of changes, which is necessary for policies that depend on values only known after variable/data-source resolution.
- OPA/Rego is the broadly-applicable choice (Kubernetes, APIs, general CI); Sentinel is the natural choice only if already standardized on Terraform Cloud/Enterprise.
- Failed security/policy checks should be **required, blocking status checks** — the same seriousness as a failing test suite, not an optional lint warning.
- Infracost applies the same "shift the visibility left" idea to cost — surfacing spend impact in the PR instead of on next month's invoice.

## Common Mistakes

- Running Checkov/OPA as an informational step that doesn't actually block merges — teams see the failures, get used to ignoring them, and the gate becomes theater.
- Suppressing a Checkov finding with no comment explaining why, so six months later nobody can tell if the suppression is still valid or was just someone unblocking a PR under deadline pressure.
- Writing policy rules against source HCL patterns (regex over `.tf` files) instead of the resolved plan JSON, missing violations that only become visible after variables/data sources are evaluated.
- Treating a passing security scan as equivalent to "this is secure" rather than "this passes the specific checks we've encoded" — static scanners can't catch architectural or business-logic security issues, only known patterns.
- Adding Infracost/Checkov/OPA all at once to an existing large codebase and getting hundreds of pre-existing findings that block every future PR — better to baseline existing violations and gate only on *new* findings during rollout.
