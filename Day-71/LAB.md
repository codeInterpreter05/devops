# Day 71 — Lab: Testing in CI Pipelines

**Goal:** Write real Terratest tests for a Terraform VPC module and run them in CI on PR — the day's assigned hands-on activity — plus hands-on practice across the full test pyramid, contract testing, and smoke tests.

**Prerequisites:**
- Go 1.22+ installed (for Terratest).
- Terraform installed.
- Python 3.11+ and `pytest` installed (`pip install pytest pytest-mock`).
- An AWS account (a free-tier/sandbox account is strongly recommended — Lab 2 creates real, briefly-billable resources) with credentials configured locally.
- `pact-python` installed for Lab 3 (`pip install pact-python`).

---

### Lab 1 — Build the test pyramid for a small Python module

1. Write `deploy_helpers.py` with a `compute_rollout_batches` function (as in README file 1) and a second function `get_running_instances()` that calls `boto3.client("ec2").describe_instances(...)`.
2. Write unit tests (`test_deploy_helpers.py`) for `compute_rollout_batches`, including a `@pytest.mark.parametrize` table covering zero, exact-multiple, and remainder cases.
3. Write a mocked unit test for `get_running_instances` using `unittest.mock.patch` on `boto3.client` — no real AWS call.
4. (Integration layer) Install `moto` (`pip install moto`) and write one integration-style test that spins up a mocked-but-realistic AWS EC2 service via `moto`, actually calling `describe_instances` against it rather than a hand-rolled mock.
5. Run all tests with a marker distinguishing layers:
   ```bash
   pytest tests/unit -v -m "not integration"
   pytest tests/integration -v -m integration
   ```

**Success criteria:** You have working unit tests (fast, fully mocked) and at least one integration test (using `moto`, no hand-rolled mock) for the same underlying AWS-calling function, and can explain what each layer would and wouldn't catch.

---

### Lab 2 — The core hands-on activity: Terratest for a Terraform VPC module

1. Write a minimal Terraform VPC module at `modules/vpc/` (a VPC, 2 public subnets, 2 private subnets, an internet gateway attached only to the public route table) with outputs `vpc_id`, `public_subnet_ids`, `private_subnet_ids`.
2. Write `test/vpc_test.go` using Terratest (as in README file 2): apply the module with `defer terraform.Destroy(...)` immediately after building `terraformOptions`, then assert `vpc_id` is non-empty and both subnet ID lists have length 2.
3. Add a real-behavior assertion using the AWS SDK for Go: fetch the private subnets' route tables and assert none of them have a route to an internet gateway (proving they're genuinely private, not just labeled that way).
4. Run it:
   ```bash
   cd test && go mod init day71test && go mod tidy
   go test -v -timeout 30m -run TestVpcModule
   ```
5. Wire it into a GitHub Actions workflow triggered `on: pull_request`, using a scoped IAM role via OIDC (`aws-actions/configure-aws-credentials`) pointed at a sandbox account.

**Success criteria:** Terratest actually creates the VPC module's real resources in a sandbox AWS account, asserts real routing behavior via the AWS SDK (not just Terraform outputs), and tears everything down whether the test passes or fails.

---

### Lab 3 — Contract testing with Pact

1. Write a tiny "consumer" function `get_inventory_level(sku)` that calls `GET http://inventory-service/inventory/{sku}` and returns the parsed JSON.
2. Write a Pact consumer test (as in README file 3) asserting the function correctly parses a `{"sku": "ABC123", "quantity": 42}` response, using Pact's mock provider — confirm it generates a pact JSON file on disk.
3. Write a minimal Flask/FastAPI "provider" stub implementing `GET /inventory/{sku}` returning that exact shape.
4. Run `pact-verifier` (or the Python pact verifier equivalent) against your local provider stub using the generated pact file, and confirm verification passes.
5. Break the provider on purpose (rename the `quantity` field to `qty`) and re-run verification — confirm it now fails, catching the contract violation.

**Success criteria:** You have a working consumer test generating a pact file, and a provider verification step that fails when the provider's real response shape no longer matches what the consumer expects.

---

### Lab 4 — Smoke tests and test report publishing

1. Write `smoke-test.sh` (as in README file 3) checking a health endpoint and one critical read path against a locally-running stub service (`python -m http.server` with a fake `/health` JSON file is enough for this exercise).
2. Wire it into a GitHub Actions job that runs after a (simulated) `deploy` step, with a follow-up `rollback` step gated on `if: failure()`.
3. Add `pytest --junitxml=results.xml` to your Lab 1 tests and publish the report using `dorny/test-reporter` (or GitLab's native `artifacts: reports: junit:` if you're using GitLab CI) with `if: always()`.
4. Deliberately break one test and confirm the published report clearly shows which test failed, without needing to read raw console logs.

**Success criteria:** A failed smoke test triggers the rollback step automatically, and a failed pytest run produces a browsable, structured report in the CI UI rather than only raw log output.

---

### Cleanup

```bash
cd test && go test -v -timeout 30m -run TestVpcModule   # re-run if `destroy` didn't complete, to confirm no leaked resources
rm -rf tests/ modules/ test/ smoke-test.sh results.xml
```

### Stretch challenge

Extend the Terratest suite to run destructively against a *chaos scenario*: after the VPC applies successfully, manually delete one subnet via the AWS SDK mid-test (simulating drift), then re-run `terraform plan` programmatically from within the Go test and assert it detects the drift — bridging today's IaC-testing material directly into tomorrow's chaos-engineering topic.
