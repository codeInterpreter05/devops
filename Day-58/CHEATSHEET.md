# Day 58 — Cheatsheet: AWS CDK

## CLI basics

```bash
npm install -g aws-cdk              # install CDK CLI (Node-based, even for Python apps)
cdk init app --language python      # scaffold a new app
cdk bootstrap aws://<account>/<region>   # one-time per account/region, creates CDKToolkit stack

cdk synth                            # compile app -> CloudFormation template(s) in cdk.out/
cdk synth <StackName>                # synth one stack
cdk diff <StackName>                 # compare local synth vs deployed stack (like terraform plan)
cdk deploy <StackName>               # synth + CloudFormation create/update
cdk deploy --all                     # deploy every stack in the app
cdk deploy --require-approval broadening   # pause only on IAM/SG-widening changes (default behavior)
cdk destroy <StackName>              # tear down
cdk ls                               # list stacks in the app
cdk context                          # view cached context values
```

## Core object model

```python
from aws_cdk import App, Stack
from constructs import Construct

class MyStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)
        # resources go here

app = App()
MyStack(app, "MyStack-dev", env={"account": "111111111111", "region": "us-east-1"})
app.synth()
```
`App` (root) → `Stack` (1:1 with a CloudFormation stack, unit of deploy/rollback) → `Construct` (composable building block).

## Construct levels

```
L1  Cfn<Resource>          direct 1:1 CloudFormation mapping, no defaults
L2  <Resource>              curated: secure defaults, helper methods (.grant_read(), etc.)
L3  <Pattern>                whole architectures in one call (e.g. ApplicationLoadBalancedFargateService)
```
```python
# escape hatch: drop from L2 down to the underlying L1 resource
cfn_resource = my_l2_construct.node.default_child
cfn_resource.add_property_override("SomeProperty.Nested", value)
```

## Context / environment config

```json
// cdk.json
{ "context": { "dev": {"account": "111...", "instanceType": "t3.micro"} } }
```
```python
config = self.node.try_get_context(self.node.try_get_context("env") or "dev")
```
```bash
cdk deploy -c env=prod
```

## CDK Pipelines

```python
from aws_cdk import pipelines as pipelines

pipeline = pipelines.CodePipeline(self, "Pipeline",
    synth=pipelines.ShellStep("Synth",
        input=pipelines.CodePipelineSource.git_hub("org/repo", "main"),
        commands=["npm ci", "npx cdk synth"]))

pipeline.add_stage(MyAppStage(self, "Staging", env=staging_env))
pipeline.add_stage(MyAppStage(self, "Prod", env=prod_env),
                    pre=[pipelines.ManualApprovalStep("PromoteToProd")])
```
First stage always self-mutates: pipeline redeploys its own CDK-defined structure before running the rest.

## `cdk-nag`

```python
from cdk_nag import AwsSolutionsChecks, NagSuppressions
from aws_cdk import Aspects

Aspects.of(app).add(AwsSolutionsChecks(verbose=True))

NagSuppressions.add_resource_suppressions(resource, [
    {"id": "AwsSolutions-S1", "reason": "documented justification, never blank"}
])
```
Rule packs: AwsSolutionsChecks, HIPAASecurityChecks, NIST80053R5Checks, PCIDSS321Checks.

## CDK vs Terraform vs raw CloudFormation (quick recall)

| | CloudFormation | Terraform | CDK |
|---|---|---|---|
| Language | YAML/JSON | HCL | Real language (Py/TS/Java/C#/Go) |
| Multi-cloud | No | Yes | No (AWS only; cdktf for multi-cloud+CDK style) |
| State | CFN stack state | own `.tfstate` | delegates to CFN stack state |
| Testing | none native | plan-diff / Terratest | native unit tests (`assertions.Template`) |
