# Day 71 — Testing in CI Pipelines: Infrastructure Testing with Terratest

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** Testing

## Brief

Terraform's own tooling (`terraform validate`, `terraform plan`) checks that your HCL is syntactically valid and shows you what *would* change — it does not tell you whether the infrastructure it creates actually *works* the way you intend (a security group that's supposed to block port 22 externally, a VPC whose subnets are actually routable the way you designed). **Terratest** closes that gap: it's a Go testing library that actually applies your Terraform, makes real assertions against the real resulting infrastructure, and tears it down afterward. This is a distinctly senior-level DevOps skill — most engineers write Terraform; comparatively few write automated tests *for* their Terraform, which is exactly why "how do you test Infrastructure as Code" is a strong differentiating interview question.

## What can and can't be tested, and at what layer

- **Static/syntax validation** (`terraform validate`, `terraform fmt -check`) — catches HCL syntax errors and basic internal consistency. Fast, free, no real infrastructure involved. This is the "unit test" layer for Terraform in spirit, though it's not really testing *logic*, just structural validity.
- **Policy/config validation** (Conftest against `terraform plan -json`, from Day 68) — catches known-bad configuration patterns (public S3 buckets, open security groups) without ever creating real infrastructure. Fast, still no real cloud resources touched.
- **Terratest (real apply + assert + destroy)** — the only layer that proves the infrastructure *actually behaves correctly once created*: that an EC2 instance in a private subnet truly has no route to the internet, that a load balancer actually distributes traffic, that an IAM role genuinely can't do more than intended. This requires real cloud API calls (usually against a dedicated, disposable test account/project), costs real (small) money, and is slow (minutes, since it's a real `terraform apply`).

**What genuinely can't be unit-tested about IaC**: emergent, cross-resource runtime behavior. You can statically verify a security group rule's *declared* CIDR block, but you cannot know from the HCL alone whether two resources' actual runtime interaction (e.g., a Lambda's VPC configuration actually allowing it to reach a downstate RDS instance through the exact subnet/route-table/NACL combination) works as intended without truly creating them and testing the live behavior. This is the crux of the "what can and can't be unit-tested" interview question — the answer isn't "nothing can be tested," it's "structural correctness is cheap to check statically, but actual runtime behavior requires real infrastructure."

## A real Terratest example

```go
package test

import (
	"testing"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../modules/vpc",
		Vars: map[string]interface{}{
			"cidr_block": "10.0.0.0/16",
			"environment": "test",
		},
	})

	// Ensures resources are destroyed even if assertions fail or the test panics
	defer terraform.Destroy(t, terraformOptions)

	terraform.InitAndApply(t, terraformOptions)

	vpcId := terraform.Output(t, terraformOptions, "vpc_id")
	assert.NotEmpty(t, vpcId)

	privateSubnetIds := terraform.OutputList(t, terraformOptions, "private_subnet_ids")
	assert.Equal(t, 2, len(privateSubnetIds))

	// Real AWS SDK call to verify actual runtime behavior, not just declared config
	// e.g., confirm the private subnets have no route to an internet gateway
}
```

```bash
cd test/
go test -v -timeout 30m -run TestVpcModule
```

Key structural points about how Terratest is used:
- **`defer terraform.Destroy(...)`** immediately after creating the options, before `InitAndApply` even runs — this guarantees cleanup happens even if the test panics or an assertion fails partway through, which matters enormously once you're spinning up real, billable cloud resources in CI.
- **`terraform.Output(...)`** reads Terraform outputs the same way a consumer of your module would — testing the actual public interface of the module, not its internals.
- Terratest tests typically go further than reading Terraform outputs — they use the cloud provider's own SDK (AWS SDK for Go, in this example) to independently verify real runtime behavior against the actual created resources, which is the whole point versus just re-checking Terraform's own state.

## CI integration — running on PR against a real (disposable) account

```yaml
jobs:
  terratest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/terratest-ci   # dedicated, scoped test-account role
          aws-region: us-east-1
      - run: cd test && go mod download
      - run: cd test && go test -v -timeout 30m ./...
```

Running Terratest against a **dedicated test/sandbox AWS account**, not the same account as production, is standard practice — it isolates the (real) cost, blast radius, and risk of a test run gone wrong (e.g., a failed `terraform destroy` leaving orphaned resources) away from anything that matters.

## Points to Remember

- `terraform validate`/`fmt` and Conftest policy checks are fast, free, static checks — they catch structural/config-pattern issues without ever creating real infrastructure, but can't verify actual runtime behavior.
- Terratest genuinely applies Terraform, asserts against real created resources (often via the cloud provider's own SDK, not just Terraform outputs), and destroys everything afterward — it's the only layer that proves infrastructure actually *behaves* as intended at runtime.
- `defer terraform.Destroy(...)` placed immediately after creating `terraformOptions` (before `InitAndApply`) is the standard pattern to guarantee cleanup even on test failure/panic — critical since real, billable cloud resources are involved.
- What can't be reliably unit-tested about IaC is emergent, cross-resource runtime behavior (actual network reachability, actual IAM effective permissions) — only creating the real resources and observing them proves that.
- Run Terratest suites against a dedicated, disposable test/sandbox cloud account, never production — isolating cost and blast radius from anything that matters if a test or its cleanup fails.

## Common Mistakes

- Placing `terraform.Destroy` at the *end* of the test function instead of immediately via `defer` — if an assertion fails or the test panics partway through, cleanup never runs, silently leaking real, billable cloud resources.
- Running Terratest suites against the production AWS account/project "because it's just a test" — a bug in the test itself, or a failed cleanup, can create (or worse, tear down) real production-adjacent resources.
- Treating `terraform validate` or Conftest's static checks as sufficient proof that infrastructure "works," when they only prove HCL syntax validity and known-bad-pattern absence — neither actually creates or observes real runtime behavior.
- Writing Terratest suites so slow (minutes per test, real cloud resources) that they run on every single commit/PR — the same "gate expensive tests appropriately" lesson from the general test pyramid applies directly to infrastructure testing; these often belong on a schedule or gated to merges into a release branch, not every push.
- Forgetting to scope the CI role used for Terratest tightly (least privilege) — an overly broad IAM role for "just running some tests" is itself a security risk if the CI pipeline or its secrets are ever compromised.
