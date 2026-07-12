# Day 30 — Terraform Remote State & Modules: Output Dependencies & Terraform Cloud

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** Terraform | **Flag:** ⚡ Interview-critical

## Brief

The last piece of the remote-state/modules picture is understanding how outputs chain together into dependency graphs across modules, and what a **managed backend** like Terraform Cloud buys you over rolling your own S3 + DynamoDB setup. Interviewers ask about Terraform Cloud specifically because "would you build this yourself or use the managed option" is a real, recurring decision every team using Terraform at scale has to make.

## Output dependencies within and across modules

Outputs aren't just "print this value at the end" — they're **graph nodes**. When one resource/module output references another, Terraform adds an edge to its dependency graph, which determines ordering during both `apply` and `destroy`.

```hcl
module "vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
}

module "app" {
  source    = "./modules/app"
  subnet_id = module.vpc.subnet_ids[0]   # app module depends on vpc module
}

output "app_url" {
  value = module.app.load_balancer_dns   # root output depends on the app module
}
```

Terraform automatically knows `module.app` must be created after `module.vpc` because `module.app`'s input references `module.vpc`'s output — this is **implicit dependency via reference**, the same mechanism as within a single module, just crossing module boundaries. On `destroy`, the order reverses: `module.app` is torn down before `module.vpc`, because you can't safely delete a VPC while something is still using its subnet.

### When implicit dependency isn't enough: `depends_on`

Some dependencies aren't visible through data references — e.g., an IAM policy needs to exist before a Lambda function that assumes a role governed by it, but the Lambda resource block might not reference any attribute of the policy directly.

```hcl
resource "aws_iam_role_policy" "lambda_policy" {
  # ...
}

resource "aws_lambda_function" "fn" {
  # ... no direct attribute reference to aws_iam_role_policy.lambda_policy
  depends_on = [aws_iam_role_policy.lambda_policy]
}
```

`depends_on` is the explicit escape hatch — use it sparingly, only when you've confirmed no attribute reference can express the same ordering, since it makes the graph harder to read for the next engineer (they can't tell *why* the dependency exists, just that it does).

### `output` blocks can also declare `depends_on`

Occasionally an output needs to wait on a resource it doesn't directly reference (e.g., waiting for an `aws_eip_association` to settle before reporting `public_ip`, when the IP is technically available slightly earlier in the graph). `output` blocks accept a `depends_on` argument for exactly these edge cases — used rarely, but worth knowing it exists.

## Terraform Cloud — the managed alternative to self-hosted remote state

Terraform Cloud (TFC, HashiCorp's SaaS offering; there's also self-hosted Terraform Enterprise) replaces the S3 + DynamoDB DIY pattern with a managed backend that also runs `plan`/`apply` itself, not just storing state:

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "prod-networking"
    }
  }
}
```

What you get beyond "just state storage":
- **Remote execution** — `plan`/`apply` run on TFC's infrastructure, not your laptop or a bespoke CI runner, giving consistent execution environments and centralized logs.
- **Built-in locking and versioned state** — the S3+DynamoDB problem solved without you assembling it.
- **VCS-driven runs** — a `git push`/PR can automatically trigger a `plan`, with the plan output posted back to the PR (the same outcome Day 31's GitHub Actions setup builds by hand).
- **Sentinel/OPA policy-as-code gating** — block an `apply` if a plan violates an organizational policy (e.g., "no public S3 buckets," "no untagged resources") before it ever executes — covered more in Day 31.
- **Team/role-based access control** over who can approve applies to which workspaces, and a private module registry for internal modules.

**The honest tradeoff**: TFC's free tier covers small teams; paid tiers add cost as you scale. The DIY S3+DynamoDB pattern is "free" (just AWS storage costs) but you own building the CI orchestration, the PR-comment automation, and any policy gating yourself — which is exactly what Day 31 walks through building manually. Many teams start DIY and move to TFC (or an alternative like Spacelift/env0/Atlantis) once the self-built CI plumbing becomes its own maintenance burden.

## Points to Remember

- Referencing one module's output as another module's input creates an implicit dependency edge — this is the primary way Terraform orders cross-module operations, same mechanism as within one module.
- `depends_on` is the explicit fallback for dependencies with no visible attribute reference — use it only when necessary, since it hides the "why" from readers.
- Destroy order is the reverse of create order, derived from the same dependency graph — this is why you can't casually `-target` destroy something without considering what depends on it.
- Terraform Cloud replaces DIY S3+DynamoDB with managed state storage, locking, remote execution, VCS-triggered runs, and policy gating — the same problems Day 31's hand-built CI pipeline solves piece by piece.
- TFC/TFE aren't the only managed option — Atlantis (self-hosted, open source), Spacelift, and env0 solve overlapping problems with different tradeoffs.

## Common Mistakes

- Overusing `depends_on` as a first resort instead of restructuring the config so an actual attribute reference exists — makes dependencies invisible in the code and easy to accidentally remove later.
- Assuming module output references alone guarantee correct *destroy* ordering when a `-target`-scoped destroy is used — targeting can bypass parts of the graph the engineer didn't think to include.
- Choosing to hand-roll all of Terraform Cloud's functionality (locking, PR-triggered plans, policy gating) without first checking whether TFC's free/lower tiers already cover the team's actual scale — reinventing infrastructure that's a checkbox away.
- Treating TFC's remote execution as a substitute for reading the plan output — teams still need a human (or a Sentinel/OPA policy) actually gating the apply, not just moving where `apply` physically runs.
- Forgetting that migrating a config's backend (e.g., S3 → TFC `cloud` block) requires `terraform init` to detect and offer a state migration — just swapping the block without re-running `init` leaves Terraform pointed at the wrong state.
