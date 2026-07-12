# Day 58 — Quiz: AWS CDK

Try to answer without looking at your notes. Answers are at the bottom.

1. What are the three levels of CDK's App → Stack → Construct hierarchy, and which one maps 1:1 to a real CloudFormation stack?
2. What does `cdk synth` actually produce, and does CDK ever talk directly to AWS APIs to create resources?
3. What is `cdk bootstrap` for, and what happens if you skip it on a brand-new account/region?
4. Explain the difference between an L1, L2, and L3 construct, with an example of each.
5. What is the "escape hatch" pattern, and when would you need it?
6. Does CDK maintain its own state file the way Terraform does? If not, what tracks deployed resource state?
7. What does `cdk diff` show you, and what mechanism does it use under the hood to compute that diff?
8. What does `cdk deploy --require-approval broadening` protect against, and is it opt-in or default behavior?
9. What is "self-mutation" in a CDK Pipeline, and what problem does it solve?
10. What does `cdk-nag` check, and at what point in the workflow does it run?
11. Terraform is provider-agnostic and multi-cloud; what is CDK's equivalent scope, and what tool exists if you want CDK's programming-language approach with Terraform's multi-provider reach?
12. **Interview question:** Compare AWS CDK with Terraform. In what situations would you choose CDK?

---

## Answers

1. App (root, holds one or more Stacks) → Stack (maps 1:1 to a CloudFormation stack — the deployable/rollback unit) → Construct (any composable building block, including Stacks and Apps themselves in CDK's object model). Stack is the one that corresponds 1:1 to a real CloudFormation stack.
2. `cdk synth` runs your CDK program and produces plain CloudFormation templates (JSON/YAML), written to `cdk.out/`. CDK never talks directly to AWS to provision resources — CloudFormation does, exactly as it always has; CDK is a code-generation/orchestration layer that hands its output to CloudFormation via `cdk deploy`.
3. `cdk bootstrap` creates a one-time-per-account/region `CDKToolkit` CloudFormation stack containing an S3 bucket (for staging synthesized templates and file assets like Lambda bundles) and IAM roles CDK deployments assume. Skipping it causes the first `cdk deploy` to fail immediately, typically with an error about a missing S3 bucket or IAM role.
4. L1 (`Cfn<Resource>`) is a direct, auto-generated 1:1 mapping to a CloudFormation resource type with no defaults — e.g., `CfnBucket`. L2 (`<Resource>`) is a hand-written, curated wrapper with sensible secure defaults and convenience methods — e.g., `s3.Bucket` with `.grant_read()` auto-generating the correct IAM policy. L3 (patterns) composes many L2 constructs into a full architecture in one call — e.g., `ApplicationLoadBalancedFargateService` provisioning a Fargate service, ALB, target groups, and security groups together.
5. Every L2/L3 construct exposes its underlying L1 resource via `.node.default_child`, letting you call `.add_property_override(...)` to set a specific CloudFormation property the higher-level construct doesn't expose directly. You need it when an L2/L3 abstraction is missing a property or behavior you require, without having to drop the whole resource down to a hand-written L1/raw CloudFormation definition.
6. No — CDK has no separate state file of its own. It delegates entirely to CloudFormation's own stack state (tracked server-side by AWS), the same mechanism CloudFormation has always used, including its native drift detection.
7. `cdk diff` synthesizes your current code and compares the resulting template against the currently deployed CloudFormation stack, printing additions/changes/removals — it's CDK's version of `terraform plan`. It relies on the same underlying mechanism CloudFormation itself uses for change sets, so the diff reliably previews what an actual deploy would do.
8. It protects against a code change accidentally widening IAM permissions or security group rules without anyone noticing — `cdk deploy` will pause and require manual confirmation any time the diff would broaden access in those two categories. This is CDK's default behavior (the approval prompt appears automatically for these specific kinds of changes), not something you must opt into separately, though the flag lets you control/adjust it.
9. Self-mutation means a CDK Pipeline's first stage checks whether the pipeline's own CDK-defined structure has changed and, if so, deploys that updated structure to itself via CloudFormation before continuing the rest of the pipeline run. It solves the chicken-and-egg problem where changing a pipeline (new stage, new approval gate, new target account) would otherwise require someone to manually update the CodePipeline resource out-of-band before the new definition could even execute.
10. `cdk-nag` checks synthesized CloudFormation templates against configurable rule packs (AWS Solutions, HIPAA Security, NIST 800-53, PCI-DSS, etc.) for security/compliance misconfigurations — e.g., unencrypted S3 buckets, overly-permissive IAM wildcards, open security groups. It runs at `synth` time (via an `Aspect` applied to the app), catching issues before deployment rather than after the resource already exists in the account.
11. Plain AWS CDK is AWS-only — there is no native multi-cloud target. `cdktf` (CDK for Terraform) brings CDK's real-programming-language approach (constructs, testing, composition) to Terraform's provider ecosystem, giving you both the programming-language ergonomics and Terraform's multi-cloud provider reach, as a distinct, separately-maturing tool.
12. Strong answer: "Both provision infrastructure as code, but Terraform is a declarative, provider-agnostic DSL (HCL) with its own self-managed state file, while CDK lets you define AWS infrastructure in a real programming language that compiles down to CloudFormation templates, delegating all actual state tracking to CloudFormation itself. I'd choose CDK when the team is AWS-only, already writes application code in a CDK-supported language (TypeScript/Python/Java/C#/Go), and wants infrastructure that can be unit-tested in the same test framework and CI pipeline as the application — CDK's L2/L3 constructs and native `assertions.Template` testing give real leverage there. I'd choose Terraform when the org needs one consistent tool across multiple cloud providers or on-prem systems, or is already heavily invested in a mature Terraform module ecosystem — introducing CDK just for the AWS portion would mean maintaining two separate IaC toolchains for no clear benefit."
