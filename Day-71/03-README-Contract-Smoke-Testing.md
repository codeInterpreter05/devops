# Day 71 — Testing in CI Pipelines: Contract Testing, Smoke Tests, and Test Reports

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** Testing

## Brief

Unit and integration tests (file 1) and infrastructure tests (file 2) still leave two real gaps: **do independently-deployed services actually agree on their API contract with each other**, and **did the thing that just got deployed actually come up healthy in the real environment**? Contract testing (Pact) and post-deploy smoke tests answer those two questions respectively, and both are squarely in a DevOps/platform engineer's territory rather than purely an application developer's — they live at the deployment/pipeline boundary, not inside a single service's codebase.

## Contract testing with Pact — solving the microservices integration problem

In a microservices architecture, a **consumer** service (e.g., a checkout service) calls a **provider** service (e.g., an inventory service). Full end-to-end tests spinning up both real services are slow and require both teams to coordinate deploys just to test integration — exactly the kind of e2e-heavy, brittle testing the pyramid (file 1) warns against. **Consumer-driven contract testing** flips this: the consumer team writes a "contract" describing exactly what it expects from the provider (request shape, expected response shape), and that contract is then verified independently against the *real* provider — without either service needing the other running at test time.

**How it actually works, step by step:**

1. **Consumer test** (checkout service, written by the checkout team): runs against a Pact **mock provider**, asserting the checkout code correctly calls and handles a specific expected request/response shape. This produces a **pact file** (a JSON contract) as a byproduct.
2. The pact file is published to a **Pact Broker** (a shared service both teams' CI pipelines talk to).
3. **Provider verification** (inventory service, run in the inventory team's own CI): pulls the pact file(s) from the broker and replays the consumer's expected requests against the *real* inventory service code, asserting real responses match what the consumer expects.
4. Both sides run entirely independently — neither team needs the other's service actually running to test the integration, and the safety net is "if the provider ever changes its response shape in a way that breaks what any known consumer expects, provider verification fails immediately."

```python
# Consumer-side pact test (checkout service, pytest + pact-python)
from pact import Consumer, Provider

pact = Consumer('CheckoutService').has_pact_with(Provider('InventoryService'))

def test_get_inventory_level():
    expected = {'sku': 'ABC123', 'quantity': 42}
    (pact
     .given('inventory exists for sku ABC123')
     .upon_receiving('a request for inventory level')
     .with_request('GET', '/inventory/ABC123')
     .will_respond_with(200, body=expected))

    with pact:
        result = get_inventory_level('ABC123')   # the actual consumer code under test
        assert result['quantity'] == 42
```

```bash
# Provider-side verification (inventory service's own CI, pulls contracts from the broker)
pact-verifier --provider-base-url=http://localhost:8080 \
  --pact-broker-base-url=https://pact-broker.mycompany.com \
  --provider=InventoryService --publish-verification-results
```

**Why this matters for CI/CD specifically**: this is what lets independent teams deploy independently without a shared, coordinated integration-testing environment — each team's pipeline verifies compatibility against contracts, not against the other team's actually-running service, which is exactly the decoupling that makes independent, frequent deploys possible in a microservices org.

## Smoke tests — the last line of defense, immediately post-deploy

A smoke test is a minimal, fast check run immediately after a deployment completes, confirming the new version came up healthy in the real target environment — before real user traffic is allowed to hit it, or before a canary rollout proceeds further.

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_URL="${1:?Usage: smoke-test.sh <base-url>}"

echo "Checking health endpoint..."
curl -sf "${BASE_URL}/health" | grep -q '"status":"ok"' || { echo "Health check failed"; exit 1; }

echo "Checking a critical read path responds..."
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${BASE_URL}/api/v1/products")
[[ "$STATUS" == "200" ]] || { echo "Products endpoint returned $STATUS"; exit 1; }

echo "Smoke test passed."
```

```yaml
- name: Deploy
  run: ./deploy.sh production

- name: Smoke test
  run: ./smoke-test.sh https://api.myapp.com

- name: Rollback on smoke test failure
  if: failure()
  run: ./rollback.sh production
```

Smoke tests deliberately test only a small number of critical paths (health endpoint, a couple of key read paths) — they are not meant to replace the full test suite that already ran pre-deploy; their entire job is to answer one narrow question fast: **"did this specific deploy, in this specific real environment, come up healthy right now."** This catches environment-specific failures (wrong secret in this environment, a missing environment variable, a database migration that didn't actually run) that no amount of pre-deploy testing in a different environment could have caught, because those failures are specific to *this* deploy, *this* environment, *right now*.

## Publishing test reports in CI

A test that fails silently in a log nobody reads is nearly as bad as no test at all. Every major CI system supports publishing structured test results (typically JUnit XML, a format that originated in Java but became a de facto universal standard nearly every test framework can emit) as first-class, browsable output.

```yaml
- name: Run tests
  run: pytest --junitxml=results.xml

- name: Publish test results
  uses: dorny/test-reporter@v1
  if: always()
  with:
    name: pytest results
    path: results.xml
    reporter: java-junit
```

```yaml
# GitLab CI native JUnit report support
test:
  script:
    - pytest --junitxml=results.xml
  artifacts:
    reports:
      junit: results.xml
```

`if: always()` on the report-publishing step matters: without it, a failed test run also skips *publishing the report explaining the failure*, forcing whoever's debugging to dig through raw console logs instead of a structured, clickable test report.

## Points to Remember

- Contract testing (Pact) solves the specific problem of verifying independently-deployed services agree on their API contract, without either team needing the other's real service running — this is what enables safe, independent deploys in a microservices architecture.
- The Pact flow: consumer test generates a pact file → published to a Pact Broker → provider verification replays it against the real provider in the provider team's own CI. Both sides test independently against the shared contract, not against each other live.
- Smoke tests run immediately post-deploy against the real target environment, checking a small number of critical paths fast — they catch environment-specific failures (bad secret, missed migration) that no pre-deploy test in a different environment could catch.
- Publish structured test reports (JUnit XML is the near-universal format) with `if: always()` so failure details are inspectable in the CI UI, not just buried in raw console logs.
- Each of these three techniques (contract, smoke, reporting) targets a distinct gap: cross-service contract drift, this-specific-deploy health, and failure visibility — none of them substitutes for the other.

## Common Mistakes

- Relying purely on full e2e tests to catch cross-service integration issues instead of contract testing — this couples every team's CI to every other team's service being up and correctly configured, recreating exactly the brittle, slow, tightly-coupled testing the microservices architecture was meant to avoid.
- Writing an exhaustive smoke test suite that essentially re-runs the full integration/e2e suite post-deploy — defeating its purpose (fast, narrow, immediate signal) and slowing down every deploy meaningfully.
- Forgetting to gate a rollback (or halting a progressive/canary rollout) on smoke test failure — running the smoke test but not actually wiring its result into an automated decision means a failed smoke test is just another log line nobody's watching in real time.
- Not publishing structured test reports (`if: always()`), so a failed CI run just shows a red X with no easy way to see *which* test failed and why, without digging through raw logs — slows down every debugging cycle across the team.
- Treating a passing provider-side Pact verification as permanent — if the provider changes its API later without re-running verification against current consumer contracts (e.g., a consumer team hasn't re-published an updated pact after their own change), a real-world break can still slip through; contract verification needs to run continuously in each team's own CI, not as a one-time check.
