# Day 58 — Resources: AWS CDK

## Primary (assigned)

- **AWS CDK Workshop** (cdkworkshop.com) — the assigned starting point; a hands-on, step-by-step build (in your language of choice) covering constructs, stacks, testing, and pipelines from scratch. The best single resource to actually get comfortable with CDK's mental model.

## Deepen your understanding

- **AWS CDK API Reference** (docs.aws.amazon.com/cdk/api/v2) — the definitive source for what's available at L1 vs L2 vs L3 for any given AWS service; check here whenever you're unsure if a property is exposed at the L2 level or needs the escape hatch.
- **AWS CDK Developer Guide — Constructs / Stacks / Environments** (docs.aws.amazon.com/cdk/v2/guide) — the conceptual reference behind today's core-concepts file, including bootstrapping internals and environment/context resolution order.
- **CDK Pipelines documentation** (docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html) — covers self-mutation, multi-account/multi-stage patterns, and the `ShellStep`/`CodeBuildStep` synth mechanics in depth.
- **cdk-nag GitHub repo** (github.com/cdklabs/cdk-nag) — full rule-pack documentation (AwsSolutions, HIPAA, NIST, PCI-DSS) with explanations for every individual rule ID and correct suppression syntax.

## Reference / lookup

- **Construct Hub** (constructs.dev) — searchable registry of community and AWS-published L3 pattern constructs; check here before hand-building an architecture pattern someone may have already published.
- **AWS CDK GitHub — Issues & RFCs** (github.com/aws/aws-cdk) — useful when an L2 construct is missing a feature; often there's an open issue or RFC with a documented workaround via the escape hatch.

## Practice

- **Replicate an existing Terraform project in CDK** (exactly what today's lab does with Day 29's VPC) — the single best exercise for building an honest, firsthand opinion on the CDK-vs-Terraform tradeoff instead of repeating secondhand takes.
- **AWS Skill Builder — "AWS Cloud Development Kit (CDK) Primer"** (free, skillbuilder.aws) — a shorter, structured video course as a supplement if you prefer guided video over the workshop's self-paced format.
