# Day 58 — AWS CDK: CDK Pipelines & Tooling (cdk diff, cdk-nag)

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** AWS | **Flag:** 📌

## Brief

Beyond writing and synthesizing CDK apps, real teams need a safe deployment pipeline and a way to catch security misconfigurations before they ship — CDK Pipelines and `cdk-nag` are the ecosystem's answers to both. CDK Pipelines is notable for a genuinely distinctive feature among IaC deployment tools: **self-mutation** — the pipeline can safely redeploy its own definition when you change it, without anyone manually re-running a bootstrap command in CodePipeline's console. This file covers that, plus the day-to-day diffing and security-linting tools you'd actually use before every deploy.

## `cdk diff` — see the CloudFormation change set before you deploy

```bash
cdk diff MyStack-prod
```

`cdk diff` synthesizes your current code and compares the resulting template against the *currently deployed* CloudFormation stack, printing a human-readable diff of resources that would be added/changed/removed — the direct CDK analog of `terraform plan`. Under the hood it's using the same mechanism CloudFormation itself uses for change sets, so what `cdk diff` shows you is a reliable preview of what `cdk deploy` (or a CDK Pipelines stage) will actually do.

```bash
cdk diff --context env=prod          # diff against a specific context/environment
cdk deploy --require-approval broadening   # only prompt for manual approval if IAM/security-group changes broaden access
```

`--require-approval` matters operationally: by default, `cdk deploy` will pause and ask for confirmation any time a diff would widen IAM permissions or security group rules — a built-in safety net against a code change accidentally granting more access than intended, without needing a separate policy tool to catch it.

## CDK Pipelines — self-mutating continuous delivery

A CDK Pipeline is itself defined as CDK code (a construct, `pipelines.CodePipeline`), which means the pipeline's own definition — its stages, its stacks, its approval gates — lives in the same repository and language as your infrastructure. The distinctive property: **the first stage of every CDK Pipeline is a self-mutation stage** that runs `cdk deploy` against the *pipeline stack itself* before deploying anything else.

```python
from aws_cdk import pipelines as pipelines

pipeline = pipelines.CodePipeline(self, "Pipeline",
    synth=pipelines.ShellStep("Synth",
        input=pipelines.CodePipelineSource.git_hub("myorg/myrepo", "main"),
        commands=["npm ci", "npm run build", "npx cdk synth"],
    ),
)
pipeline.add_stage(MyAppStage(self, "Prod", env={"account": "222222222222", "region": "us-east-1"}))
```

**Why self-mutation matters:** without it, changing the pipeline's own structure (adding a new stage, a new manual approval gate, a new target account) would require someone to manually update the CodePipeline resource out-of-band before the new pipeline definition could even run — a chicken-and-egg problem common to other CI/CD-as-code setups. CDK Pipelines solves this by having the pipeline detect, on every run, whether its own CDK stack definition has changed; if so, it updates itself first via CloudFormation, then restarts the pipeline execution using the newly updated structure — entirely automatically, no manual console clicks.

## Multi-stage, multi-account deployment

CDK Pipelines' `Stage` construct groups one or more Stacks that should be deployed together as an atomic unit, typically per environment:

```python
class MyAppStage(Stage):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)
        MyAppStack(self, "AppStack")
        MyDatabaseStack(self, "DbStack")

pipeline.add_stage(MyAppStage(self, "Staging", env=staging_env))
pipeline.add_stage(MyAppStage(self, "Prod", env=prod_env), pre=[pipelines.ManualApprovalStep("PromoteToProd")])
```

This is the standard pattern for promoting the same, tested application/infrastructure definition through Staging → (manual approval) → Prod, across different AWS accounts, from a single pipeline defined once.

## `cdk-nag` — automated security/compliance linting of synthesized templates

`cdk-nag` runs a battery of rule packs (AWS Solutions, HIPAA Security, NIST 800-53, PCI-DSS, and more) against your synthesized CloudFormation templates during `synth`, failing the build if a resource violates a rule — e.g., an S3 bucket without encryption, an IAM policy with a wildcard `Resource: "*"`, a security group open to `0.0.0.0/0` on a sensitive port.

```python
from cdk_nag import AwsSolutionsChecks, NagSuppressions
from aws_cdk import Aspects

Aspects.of(app).add(AwsSolutionsChecks(verbose=True))

# suppress a specific finding with a documented justification (never silently)
NagSuppressions.add_resource_suppressions(bucket, [
    {"id": "AwsSolutions-S1", "reason": "Access logging intentionally disabled for this throwaway dev bucket"}
])
```

**Why this belongs in a CI pipeline, not just a developer's local run:** `cdk-nag` catches classes of misconfiguration that a human code reviewer skimming a pull request easily misses (a wildcard buried three levels deep in a synthesized IAM policy JSON), and it does so automatically, on every `synth`, without requiring a separate scanning tool bolted on after deployment (like AWS Config rules, which catch the same issues but only *after* the resource already exists in the account).

## Points to Remember

- `cdk diff` compares your locally synthesized template against the live CloudFormation stack — the direct equivalent of `terraform plan`, and it uses CloudFormation's own change-set mechanism under the hood.
- `cdk deploy --require-approval broadening` (the default behavior) pauses for confirmation specifically when IAM or security-group changes would widen access — a built-in guardrail, not something you have to configure separately.
- CDK Pipelines are self-mutating: the pipeline's first stage always checks whether its own definition changed and redeploys itself via CloudFormation before continuing — solving the chicken-and-egg problem of updating a pipeline from within the pipeline.
- `Stage` groups Stacks that deploy together as one atomic promotion unit — the mechanism for Staging → manual approval → Prod, potentially across separate AWS accounts, from a single pipeline definition.
- `cdk-nag` lints synthesized templates against rule packs (AWS Solutions, PCI-DSS, HIPAA, NIST) at `synth` time, catching misconfigurations before deploy rather than after — the shift-left complement to post-deployment tools like AWS Config.

## Common Mistakes

- Manually editing a deployed CodePipeline in the AWS Console to "quickly fix" something, instead of changing the CDK Pipeline code — the next self-mutation run will silently revert the manual console change back to whatever the code says, confusing whoever made the manual edit.
- Not running `cdk diff` before `cdk deploy` in a script/habit, and being surprised by a change set that does more than expected — especially resource replacements (which CloudFormation shows in the diff, but are easy to miss if you don't look).
- Suppressing a `cdk-nag` finding without a documented reason (`NagSuppressions` requires a `reason` string) or, worse, disabling the entire check pack because one rule was noisy, instead of suppressing the specific finding with justification.
- Assuming `cdk-nag` (a synth-time, template-level check) replaces the need for runtime security tooling (AWS Config, GuardDuty, etc.) — it only catches what's expressible in the template; it can't catch drift or runtime behavior.
- Putting unrelated Stacks that shouldn't share a deployment/rollback lifecycle into the same `Stage`, coupling their promotion timing unnecessarily.
