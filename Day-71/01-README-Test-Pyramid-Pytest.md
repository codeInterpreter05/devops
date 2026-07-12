# Day 71 — Testing in CI Pipelines: The Test Pyramid and pytest

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** Testing

## Brief

A pipeline that builds and deploys without meaningful automated testing is just a fast way to ship bugs to production. But "add tests" isn't a flat instruction — different kinds of tests catch different kinds of bugs, run at wildly different speeds, and belong at different layers of a CI pipeline. The **test pyramid** is the mental model for getting this balance right, and it's directly relevant to DevOps work because *you* are usually the one designing the CI stages that run these tests, deciding what blocks a merge versus what runs post-deploy, and choosing tools like `pytest` for testing the actual scripts/automation that make up infrastructure tooling.

This day is split into three focused files:

1. **This file** — the test pyramid concept and `pytest` for Python DevOps scripts.
2. **[02-README-Terratest-IaC-Testing.md](02-README-Terratest-IaC-Testing.md)** — testing Infrastructure as Code itself with Terratest.
3. **[03-README-Contract-Smoke-Testing.md](03-README-Contract-Smoke-Testing.md)** — contract testing with Pact, smoke tests, and publishing test reports in CI.

## The test pyramid — why shape matters, not just presence

The pyramid describes the *proportion* of each test type you want, not just that all three should exist:

```
        /\
       /  \      E2E (few) — slow, brittle, expensive, but tests the real system end-to-end
      /----\
     /      \    Integration (some) — medium speed, tests real interactions between components
    /--------\
   /          \  Unit (many) — fast, cheap, isolated, tests one function/module in isolation
  /------------\
```

- **Unit tests** — test one function or class in isolation, with all dependencies mocked/stubbed. Fast (milliseconds each), so you can run thousands of them on every commit. Catch logic bugs precisely, but tell you nothing about whether the pieces actually work *together*.
- **Integration tests** — test real interaction between two or more components (your code talking to a real database, a real message queue, a real API). Slower, need real infrastructure (or realistic test doubles like `testcontainers`), but catch the class of bugs unit tests structurally cannot (wrong SQL, serialization mismatches, wrong assumptions about a third-party API's actual behavior).
- **End-to-end (e2e) tests** — drive the entire deployed system as a user/client would, hitting real (or near-real) infrastructure. Slowest, most brittle (fail for environmental reasons unrelated to actual bugs — flaky network, timing), but the only layer that validates the whole system genuinely works.

**Why the pyramid shape, not an inverted pyramid or a flat distribution:** e2e tests are expensive to write, slow to run, and prone to flakiness unrelated to real bugs — if most of your test suite is e2e, CI becomes slow and unreliable, and developers start ignoring "flaky" failures, which is how real regressions slip through. An "ice cream cone" anti-pattern (mostly e2e, few unit tests) is a very common real-world failure mode, especially in orgs that added testing late and reached first for the most "realistic-feeling" tests instead of the fastest, most precise ones.

## Where each layer runs in a CI pipeline

```yaml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pytest tests/unit/ -v            # runs on every PR, must be fast (<2 min ideally)

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - run: pytest tests/integration/ -v      # runs on every PR, slower, needs real service containers

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.ref == 'refs/heads/main'         # e2e often gated to main/staging only, not every PR
    steps:
      - run: pytest tests/e2e/ -v --base-url=https://staging.myapp.com
```

Gating e2e to `main` (post-merge, against a real staging environment) rather than every PR is a common, deliberate tradeoff: e2e suites are too slow and too environment-dependent to run per-commit across every branch without destroying developer velocity, but you still want them run *somewhere* before code reaches production.

## `pytest` for Python DevOps scripts

DevOps work generates a lot of Python: deployment scripts, custom Terraform/Ansible plugins, Lambda functions, internal CLI tools. These deserve real tests just as much as application code — an untested deploy script is exactly the kind of thing that fails silently in production at 2am.

```python
# deploy_helpers.py
def compute_rollout_batches(total_instances: int, batch_size: int) -> list[int]:
    if batch_size <= 0:
        raise ValueError("batch_size must be positive")
    batches = []
    remaining = total_instances
    while remaining > 0:
        batches.append(min(batch_size, remaining))
        remaining -= batch_size
    return batches
```

```python
# test_deploy_helpers.py
import pytest
from deploy_helpers import compute_rollout_batches

def test_even_split():
    assert compute_rollout_batches(10, 5) == [5, 5]

def test_uneven_split():
    assert compute_rollout_batches(11, 5) == [5, 5, 1]

def test_zero_batch_size_raises():
    with pytest.raises(ValueError):
        compute_rollout_batches(10, 0)

@pytest.mark.parametrize("total,batch,expected", [
    (0, 5, []),
    (5, 5, [5]),
    (100, 30, [30, 30, 30, 10]),
])
def test_parametrized_batches(total, batch, expected):
    assert compute_rollout_batches(total, batch) == expected
```

`@pytest.mark.parametrize` is the single highest-value pytest feature for DevOps-style testing: instead of writing four nearly-identical test functions, you write the assertion logic once and supply a table of input/expected-output pairs — encouraging you to actually think through edge cases (zero, exact multiples, remainders) as data rather than skipping them because writing another test function feels tedious.

### Mocking external calls (AWS, subprocess, HTTP)

```python
from unittest.mock import patch, MagicMock

@patch("deploy_helpers.boto3.client")
def test_get_running_instances(mock_boto_client):
    mock_ec2 = MagicMock()
    mock_ec2.describe_instances.return_value = {
        "Reservations": [{"Instances": [{"InstanceId": "i-123", "State": {"Name": "running"}}]}]
    }
    mock_boto_client.return_value = mock_ec2

    result = get_running_instances()
    assert result == ["i-123"]
```

Mocking `boto3.client` here means the unit test runs in milliseconds with zero real AWS calls, zero AWS credentials needed, and zero risk of accidentally touching real infrastructure — this is precisely what keeps unit tests at the *base* of the pyramid: fast and dependency-free. An integration test, one layer up, would instead run against **moto** (an AWS service mock) or a real sandboxed AWS account to catch anything the mock's assumptions got wrong.

## Points to Remember

- The pyramid's shape (many unit, some integration, few e2e) is about balancing speed/reliability against realism — not "have some of each and call it done."
- Unit tests mock all external dependencies and should be fast enough to run on every single commit; integration tests use real (or realistic containerized) dependencies and are slower; e2e tests drive the real deployed system and are slowest/most brittle.
- Gating expensive e2e suites to run only post-merge (on `main`, against staging) rather than on every PR is a deliberate, common tradeoff to protect developer velocity.
- `pytest.mark.parametrize` turns "write four similar test functions" into "write the assertion once, supply a data table" — and nudges you toward actually covering edge cases.
- Mock external dependencies (`boto3`, `subprocess`, HTTP clients) in unit tests specifically so they run without real credentials, real network calls, or risk of touching real infrastructure.

## Common Mistakes

- Building an "ice cream cone" test suite (mostly e2e/manual, few unit tests) because e2e "feels more realistic" — resulting in a slow, flaky CI pipeline that developers learn to ignore or re-run blindly on failure, defeating the entire purpose of automated testing.
- Running expensive e2e suites on every single PR/commit instead of gating them to post-merge — burning CI minutes and slowing down every contributor's feedback loop for tests that don't need per-commit granularity.
- Writing unit tests that don't actually mock external dependencies (a "unit test" that makes a real AWS API call or hits a real database) — these are integration tests mislabeled as unit tests, and they inherit all the integration layer's slowness/flakiness while providing none of the pyramid's intended speed benefit.
- Testing DevOps/infrastructure scripts manually ("I ran it once locally and it worked") instead of writing real `pytest` tests — exactly the scripts most likely to run unattended in production automation (deploy hooks, cron jobs, Lambda functions) are often the least tested in practice, which is backwards.
- Skipping parametrized edge cases (zero, negative numbers, empty lists, exact boundary values) because writing them "felt unnecessary" — these are disproportionately where real bugs in rollout/batching/scaling logic actually live.
