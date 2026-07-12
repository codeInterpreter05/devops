# Day 58 — AWS CDK: Core Concepts (Constructs, Stacks, Apps)

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** AWS | **Flag:** 📌

## Brief

AWS CDK (Cloud Development Kit) lets you define infrastructure using a real programming language — Python, TypeScript, Java, C#, Go — instead of a declarative templating language like HCL or raw CloudFormation YAML/JSON. It matters for DevOps work because it represents a genuinely different philosophy from Terraform: instead of learning a domain-specific configuration language, you write infrastructure with loops, conditionals, functions, classes, and your existing IDE's autocomplete/type-checking — while CDK compiles ("synthesizes") all of it down to plain CloudFormation templates under the hood. Interviewers ask about CDK vs. Terraform specifically because it tests whether you understand *why* a company would pick one over the other, not just that both "provision AWS resources."

This day is split into three focused files:

1. **This file** — CDK's core building blocks: constructs, stacks, apps, and the synth process.
2. **[02-README-Construct-Levels-CDK-vs-Terraform.md](02-README-Construct-Levels-CDK-vs-Terraform.md)** — L1/L2/L3 constructs, and CDK vs. Terraform/CloudFormation head-to-head.
3. **[03-README-CDK-Pipelines-Tooling.md](03-README-CDK-Pipelines-Tooling.md)** — CDK Pipelines (self-mutating), `cdk diff`, `cdk-nag`.

## The three building blocks: App, Stack, Construct

CDK's object model is a strict hierarchy:

- A **Construct** is the base unit — literally any reusable piece of infrastructure definition, from a single S3 bucket to an entire multi-service application pattern. Constructs can contain other constructs (composition all the way down).
- A **Stack** is a construct that maps 1:1 to a CloudFormation stack — it's the deployable unit. Everything inside a Stack gets synthesized into one CloudFormation template and deployed/rolled back together as a unit.
- An **App** is the root construct — the entry point of your CDK program, which can contain multiple Stacks (e.g., one per environment, or one per logical layer of your architecture).

```python
from aws_cdk import App, Stack
from aws_cdk import aws_s3 as s3
from constructs import Construct

class MyBucketStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)
        s3.Bucket(self, "MyBucket", versioned=True)

app = App()
MyBucketStack(app, "MyBucketStack-dev", env={"account": "111111111111", "region": "us-east-1"})
app.synth()
```

This is a real Python program — you can wrap `MyBucketStack` construction in a loop to create it once per environment, add an `if` for conditional resources, write a unit test against the synthesized template with `assertions.Template`, and import shared logic from your own Python packages, exactly like any other application code.

## `cdk synth` — the compilation step

`cdk synth` executes your CDK app (runs your Python/TypeScript/etc. code) and produces plain CloudFormation templates (JSON/YAML) as output, one per Stack, written to `cdk.out/`. This is the critical mechanical fact to internalize: **CDK does not talk to AWS directly to provision resources — CloudFormation does, exactly like it always has.** CDK is a code-generation and orchestration layer on top of CloudFormation; `cdk deploy` runs `synth` internally and then hands the resulting template to CloudFormation to create/update the stack, the same engine that's been running AWS deployments for over a decade.

```bash
cdk synth                       # generate CloudFormation template(s), print to stdout / cdk.out/
cdk synth MyBucketStack-dev > template.yaml   # save a specific stack's template
```

**Why this matters practically:** every CloudFormation guarantee and limitation still applies under CDK — rollback behavior on failed deploys, drift detection, stack size/resource limits (500 resources per stack), nested stacks for working around that limit, and IAM permissions needed by CloudFormation itself to create your resources. CDK does not remove CloudFormation's constraints; it removes the pain of hand-writing verbose YAML/JSON to describe your infrastructure.

## Bootstrapping — CDK's own infrastructure prerequisite

Before deploying any CDK app to an AWS account/region for the first time, you must run:

```bash
cdk bootstrap aws://111111111111/us-east-1
```

This creates a small, dedicated CloudFormation stack (`CDKToolkit`) containing an S3 bucket (for storing synthesized templates and file assets like Lambda code bundles) and IAM roles CDK uses for deployment. Skipping this step is the most common first-time CDK error — `cdk deploy` will fail immediately with an error about a missing S3 bucket or IAM role, because CDK genuinely has no way to stage assets or assume deployment roles until bootstrapping has run once per account/region combination.

## Context and environment-specific configuration

CDK apps read configuration from `cdk.json` (context values), environment variables, and `-c key=value` CLI flags — the equivalent of Terraform's `.tfvars`, but resolved as arbitrary key-value lookups inside your program code rather than a separate typed variable system:

```json
{
  "context": {
    "dev": {"account": "111111111111", "instanceType": "t3.micro"},
    "prod": {"account": "222222222222", "instanceType": "m5.large"}
  }
}
```
```python
env_name = self.node.try_get_context("env") or "dev"
config = self.node.try_get_context(env_name)
```
```bash
cdk deploy -c env=prod
```

## Points to Remember

- Construct → Stack → App is CDK's composition hierarchy; a Stack is the unit that maps 1:1 to a real CloudFormation stack and deploys/rolls back as a whole.
- `cdk synth` compiles your program into plain CloudFormation templates; CDK never talks to AWS to create resources directly — CloudFormation still does all the actual provisioning.
- `cdk bootstrap` must run once per account/region before first deploy — it creates the S3 asset bucket and IAM roles CDK relies on; forgetting it is the most common first-run failure.
- Because CDK apps are real code, environment-specific config, loops, and reusable abstractions are handled with ordinary programming constructs (functions, classes, context lookups) rather than a separate templating/variable language.
- Every CloudFormation limitation (rollback semantics, 500-resources-per-stack limit, drift) still applies — CDK is a layer on top, not a replacement for CloudFormation's engine.

## Common Mistakes

- Forgetting `cdk bootstrap` on a new account/region and being confused by an opaque deployment failure that's actually just a missing prerequisite stack.
- Treating a Stack as an arbitrary organizational folder rather than understanding it's a real deployment/rollback unit — putting unrelated, independently-lifecycled resources in one Stack couples their failure/rollback behavior together unnecessarily.
- Writing CDK code with heavy imperative logic (API calls, random values, non-deterministic behavior) inside construct definitions — since `synth` must produce the same template given the same inputs to be safely diffable/repeatable, and CloudFormation deployments assume declarative, deterministic templates.
- Assuming CDK "replaces" needing to understand CloudFormation — you still need to read CloudFormation error messages, understand stack events, and reason about update/replacement behavior per resource type when a deploy fails.
- Hardcoding account IDs/regions instead of using CDK's `env` context and `cdk.json`, making the same app awkward to redeploy into a second environment.
