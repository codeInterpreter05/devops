# Day 58 — AWS CDK: Construct Levels (L1/L2/L3) & CDK vs Terraform/CloudFormation

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** AWS | **Flag:** 📌

## Brief

Not all CDK constructs are the same level of abstraction — knowing the difference between L1, L2, and L3 constructs is exam/interview-relevant because it explains both CDK's power (sensible secure-by-default resources with minimal code) and its biggest practical risk (an abstraction hiding exactly what CloudFormation properties are actually being set, which matters a lot when you need one specific property nobody wrapped yet). This file also does the direct comparison against Terraform and raw CloudFormation that interviewers ask for explicitly.

## The three construct levels

- **L1 ("CFN Resources")** — a direct, mechanical, one-to-one mapping to a CloudFormation resource type, auto-generated from the CloudFormation resource specification. Named `Cfn<ResourceName>` (e.g., `CfnBucket`). Every property matches the CloudFormation property exactly, with no defaults, no convenience methods, no validation beyond types. Using an L1 construct is functionally identical to writing raw CloudFormation, just from within a programming language.

  ```python
  from aws_cdk import aws_s3 as s3
  bucket = s3.CfnBucket(self, "RawBucket",
      bucket_encryption=s3.CfnBucket.BucketEncryptionProperty(
          server_side_encryption_configuration=[...]
      )
  )
  ```

- **L2 ("Curated Constructs")** — a hand-written wrapper around one or more L1 resources providing sensible defaults, type-safe helper methods, and automatic wiring of related resources (e.g., IAM policies). This is what you use 90% of the time.

  ```python
  bucket = s3.Bucket(self, "MyBucket",
      versioned=True,
      encryption=s3.BucketEncryption.S3_MANAGED,
      block_public_access=s3.BlockPublicAccess.BLOCK_ALL,
  )
  # .grant_read(some_lambda) automatically creates the correct IAM policy statement —
  # you'd otherwise hand-write an aws_iam_policy_document in Terraform for the same effect
  ```

- **L3 ("Patterns")** — a composition of many L2 constructs implementing a common, higher-level architecture pattern in one call — e.g., `aws-ecs-patterns.ApplicationLoadBalancedFargateService` provisions a Fargate service, an ALB, target groups, security groups, and log groups, wired together correctly, in roughly 10 lines.

  ```python
  from aws_cdk import aws_ecs_patterns as ecs_patterns
  ecs_patterns.ApplicationLoadBalancedFargateService(self, "MyService",
      cluster=cluster, cpu=256, memory_limit_mib=512,
      task_image_options={"image": ecs.ContainerImage.from_registry("nginx")},
  )
  ```

**The tradeoff to understand deeply:** the higher the level, the less code you write and the more "correct by default" the result — but also the less visible and less overridable the exact resource configuration becomes. When an L3 pattern doesn't do quite what you need, you either dig into its "escape hatch" (every L2/L3 construct exposes the underlying L1 resource via `.node.default_child` so you can override specific CloudFormation properties directly) or drop to L1/hand-written CloudFormation for that piece. Knowing this escape hatch exists — and how to use it — is what separates "I used CDK for a tutorial" from "I've actually hit CDK's abstraction limits in a real project."

```python
cfn_bucket = bucket.node.default_child  # drop down to the L1 resource underneath an L2 construct
cfn_bucket.add_property_override("BucketEncryption.ServerSideEncryptionConfiguration.0.BucketKeyEnabled", True)
```

## CDK vs. Terraform vs. raw CloudFormation

| | Raw CloudFormation | Terraform | AWS CDK |
|---|---|---|---|
| Language | YAML/JSON (declarative, AWS's own DSL) | HCL (declarative, provider-agnostic DSL) | Real programming language (TypeScript, Python, Java, C#, Go) |
| Multi-cloud | AWS only | Yes — any provider with a Terraform provider | AWS only (CDK for Terraform / cdktf exists for multi-cloud CDK-style code, but plain AWS CDK targets AWS) |
| State | CloudFormation manages it server-side (stack state) | Terraform's own `.tfstate` file, self-managed or in a backend | Delegates entirely to CloudFormation's stack state — no separate CDK state file |
| Abstraction | None — every property by hand | Modules (reusable HCL, still declarative) | L1/L2/L3 constructs — real inheritance/composition, testable with unit tests |
| Drift handling | Native drift detection | `terraform plan` shows drift vs. state on next run | Inherits CloudFormation's native drift detection |
| Ecosystem maturity | Oldest, most mature for AWS specifically | Largest overall ecosystem, provider-agnostic, dominant industry default | Younger, AWS-specific, growing fast especially among teams already deep in a supported language |
| Testing | Hard — no real test framework | `terraform plan` + tools like Terratest (external, not built-in) | Native unit testing via `assertions.Template` — assert specific resources/properties exist in the synthesized template, in the same test framework as your application code (pytest, Jest, JUnit) |

**Why this comparison is a real interview question, not trivia:** the honest answer is situational. A platform team managing infrastructure across AWS, GCP, and on-prem VMware will pick Terraform for one consistent tool and language across everything. A team that's all-in on AWS, already writes application code in TypeScript/Python, and wants infrastructure definitions that can be unit-tested in the same CI pipeline and language as their application code, gets real leverage from CDK's L2/L3 constructs and native testing story. Organizations already deeply invested in Terraform modules and multi-cloud strategy have little reason to introduce a second IaC tool just for AWS.

## Points to Remember

- L1 = raw 1:1 CloudFormation mapping, no defaults; L2 = curated, secure-by-default, convenience methods (`grant_read`, etc.); L3 = whole architecture patterns in one call.
- Every L2/L3 construct exposes an escape hatch (`.node.default_child` + `add_property_override`) down to its underlying L1/CloudFormation properties — know this exists before assuming "CDK can't do X."
- CDK has no separate state file — it delegates entirely to CloudFormation's own stack state, unlike Terraform's self-managed `.tfstate`.
- Terraform is provider-agnostic/multi-cloud by design; CDK (plain AWS CDK) is AWS-only, though `cdktf` brings CDK's programming-language approach to Terraform's multi-provider ecosystem if that combination is wanted.
- CDK's native unit-testing story (`assertions.Template`, in the same language/test runner as your app) is a genuine differentiator versus Terraform, which relies on external tools (Terratest, `plan`-diffing) for equivalent confidence.

## Common Mistakes

- Assuming CDK stores its own state the way Terraform does — it doesn't; CloudFormation's stack state is authoritative, and there's no CDK-specific state file to back up or lock.
- Reaching for a completely custom L1-only implementation the moment an L2/L3 construct doesn't expose a property, without first checking `.node.default_child`'s escape hatch — often unnecessary extra work.
- Picking CDK for a genuinely multi-cloud organization "because it's newer/nicer to write" and then discovering there's no equivalent first-class multi-cloud story (unless you specifically adopt `cdktf`, which is a different tool with a different maturity profile).
- Believing CDK removes CloudFormation's practical limits (e.g., 500 resources per stack) — large CDK apps still need nested stacks or multiple Stack objects to stay within CloudFormation's real constraints.
- Not writing any unit tests against synthesized templates despite CDK making this easy and idiomatic — losing one of CDK's most concrete advantages over Terraform by not exercising it.
