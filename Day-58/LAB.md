# Day 58 — Lab: AWS CDK

**Goal:** Replicate your Day 29 VPC using AWS CDK (Python) — the assigned hands-on activity — then compare the experience directly against Terraform, and explore construct levels, diffing, and security linting hands-on.

**Prerequisites:** Node.js (CDK CLI is Node-based even for Python apps), Python 3.9+, AWS CLI configured with a sandbox account/profile, `npm install -g aws-cdk`, and your Day 29 Terraform VPC code available for side-by-side comparison.

---

### Lab 1 — Bootstrap and scaffold a CDK Python app

1. `mkdir cdk-vpc-lab && cd cdk-vpc-lab && cdk init app --language python`
2. Activate the generated virtualenv and install deps: `source .venv/bin/activate && pip install -r requirements.txt`
3. Bootstrap your account/region (skip if already bootstrapped): `cdk bootstrap aws://<account-id>/<region>`
4. `cdk synth` on the default scaffolded (empty) stack — confirm it produces a minimal valid CloudFormation template in `cdk.out/`.

**Success criteria:** `cdk synth` succeeds with no errors, and you can find the generated template file in `cdk.out/`.

---

### Lab 2 — The core hands-on activity: replicate your Day 29 VPC

1. Open your Day 29 Terraform VPC configuration side by side with your new CDK stack file.
2. Using the `aws_cdk.aws_ec2` module's L2 `Vpc` construct, replicate the same topology: CIDR block, number of AZs, public + private subnets, NAT gateway(s), route tables:
   ```python
   from aws_cdk import aws_ec2 as ec2

   vpc = ec2.Vpc(self, "MyVpc",
       ip_addresses=ec2.IpAddresses.cidr("10.0.0.0/16"),
       max_azs=2,
       nat_gateways=1,
       subnet_configuration=[
           ec2.SubnetConfiguration(name="public", subnet_type=ec2.SubnetType.PUBLIC, cidr_mask=24),
           ec2.SubnetConfiguration(name="private", subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS, cidr_mask=24),
       ],
   )
   ```
3. `cdk synth` and inspect the generated template — count how many CloudFormation resources this single `Vpc` L2 construct expanded into (VPC, subnets, route tables, IGW, NAT gateway, EIP, route table associations). Compare that count against how many `resource` blocks your Terraform version needed to write by hand.
4. `cdk deploy` into your sandbox account. Confirm in the AWS Console that the VPC topology matches your Day 29 Terraform version exactly (same AZ count, same subnet split).

**Success criteria:** A deployed VPC in your sandbox account matching Day 29's topology, plus a written one-paragraph comparison: lines of code written, resources auto-wired for you, and anything CDK's L2 construct did differently by default than your hand-written Terraform.

---

### Lab 3 — Construct levels and the escape hatch

1. Add an S3 VPC Gateway Endpoint to your VPC using the L2 API: `vpc.add_gateway_endpoint("S3Endpoint", service=ec2.GatewayVpcEndpointAwsService.S3)`.
2. Now find one property the L2 `Vpc` construct doesn't expose directly (check the CDK API docs for `CfnVPC` vs `Vpc`) and use the escape hatch to set it:
   ```python
   cfn_vpc = vpc.node.default_child
   cfn_vpc.add_property_override("EnableDnsSupport", True)
   ```
3. `cdk diff` before deploying this change — read the diff output and confirm it shows exactly the property-level change you expect, nothing more.

**Success criteria:** You've used `.node.default_child` successfully at least once and can explain, from firsthand use, when you'd need to drop from L2 to this escape hatch versus writing a raw L1 `CfnVPC` from scratch.

---

### Lab 4 — `cdk-nag` security linting

1. `pip install cdk-nag` and wire `AwsSolutionsChecks` into your app's entry point (`app.py`), applied via `Aspects.of(app).add(...)`.
2. `cdk synth` — read through the findings. Your VPC stack should be mostly clean, but add a deliberately insecure resource (e.g., a security group open to `0.0.0.0/0` on port 22) and re-run `cdk synth` to see `cdk-nag` fail the synth with a specific rule ID.
3. Fix the security group to restrict the CIDR, confirm the finding clears.
4. Add one deliberate, justified suppression using `NagSuppressions` for something you've decided is genuinely fine for a dev sandbox (e.g., missing access logging on a throwaway bucket), with a real reason string.

**Success criteria:** You've seen `cdk-nag` block a real bad config, fixed it, and used a suppression correctly with a documented justification — not blanket-disabled the checker.

---

### Lab 5 — `cdk diff` as your safety net

1. Make three different kinds of changes to your VPC stack one at a time, running `cdk diff` before each `cdk deploy`: (a) a purely additive change (add a new subnet CIDR range), (b) a property update that CloudFormation can apply in-place (a tag), (c) a change that forces resource replacement (changing the VPC's primary CIDR block).
2. For each, read the `cdk diff` output and identify which resources are marked for replacement (`[-]`/`[+]` pairs) versus in-place update (`[~]`).
3. Write one sentence on why (c) is dangerous to run against a live environment without careful review, referencing what "replacement" means for a VPC specifically (everything inside it would need to be recreated).

**Success criteria:** You can read a `cdk diff` output and correctly distinguish additive, in-place, and replacement changes before ever running `deploy`.

---

### Cleanup

```bash
cdk destroy --all
# confirm in the AWS Console that the VPC, NAT gateway, and EIP are gone (NAT gateways incur hourly cost even idle)
```

### Stretch challenge

Wire this VPC stack into a minimal CDK Pipeline (`pipelines.CodePipeline`) sourced from a GitHub repo, deploying to a single `Staging` stage — then make a trivial code change (add a tag), push it, and observe the pipeline's self-mutation stage run before your actual VPC stage deploys. Explain in writing what you observed happen in that self-mutation step.
