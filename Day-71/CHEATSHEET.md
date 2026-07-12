# Day 71 — Cheatsheet: Testing in CI Pipelines

## pytest essentials

```bash
pytest                              # run all tests discovered (test_*.py / *_test.py)
pytest tests/unit -v                 # verbose, specific directory
pytest -k "batch"                    # run tests matching name substring
pytest -m integration                # run only tests marked @pytest.mark.integration
pytest --junitxml=results.xml        # emit JUnit XML report
pytest --lf                          # re-run only last-failed tests
```

```python
import pytest
from unittest.mock import patch, MagicMock

@pytest.mark.parametrize("input,expected", [(1, 2), (2, 4)])
def test_double(input, expected):
    assert double(input) == expected

def test_raises():
    with pytest.raises(ValueError):
        risky_function(-1)

@patch("mymodule.boto3.client")
def test_mocked(mock_client):
    mock_client.return_value.describe_instances.return_value = {"Reservations": []}
    ...
```

## Terratest

```go
terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
    TerraformDir: "../modules/vpc",
    Vars: map[string]interface{}{"cidr_block": "10.0.0.0/16"},
})
defer terraform.Destroy(t, terraformOptions)   // ALWAYS immediately after building options

terraform.InitAndApply(t, terraformOptions)
output := terraform.Output(t, terraformOptions, "vpc_id")
```

```bash
go test -v -timeout 30m -run TestVpcModule
go mod tidy
```

## Pact contract testing

```python
# Consumer side
from pact import Consumer, Provider
pact = Consumer('CheckoutService').has_pact_with(Provider('InventoryService'))

(pact.given('state').upon_receiving('description')
     .with_request('GET', '/inventory/ABC123')
     .will_respond_with(200, body={'quantity': 42}))

with pact:
    result = get_inventory_level('ABC123')
```

```bash
# Provider side (in provider's own CI)
pact-verifier --provider-base-url=http://localhost:8080 \
  --pact-broker-base-url=https://pact-broker.example.com \
  --provider=InventoryService --publish-verification-results
```

## Smoke test skeleton

```bash
#!/usr/bin/env bash
set -euo pipefail
BASE_URL="${1:?usage: smoke-test.sh <url>}"
curl -sf "${BASE_URL}/health" | grep -q '"status":"ok"' || exit 1
```

## CI test-report publishing

```yaml
# GitHub Actions
- run: pytest --junitxml=results.xml
- uses: dorny/test-reporter@v1
  if: always()
  with: { name: pytest, path: results.xml, reporter: java-junit }
```

```yaml
# GitLab CI
test:
  script: pytest --junitxml=results.xml
  artifacts:
    reports:
      junit: results.xml
```

## Test pyramid gating pattern

```yaml
jobs:
  unit:        { steps: [pytest tests/unit] }                          # every PR, fast
  integration: { needs: unit, steps: [pytest tests/integration] }        # every PR, slower
  e2e:         { needs: integration, if: "github.ref == 'refs/heads/main'", steps: [...] }  # post-merge only
```
