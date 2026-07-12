# Day 71 — Resources: Testing in CI Pipelines

## Primary (assigned)

- **Terratest documentation** (terratest.gruntwork.io) — free, the assigned starting point for this day. Covers the full module library (Terraform, AWS, Docker, Kubernetes helpers) and testing patterns for real infrastructure.

## Deepen your understanding

- **Martin Fowler — "TestPyramid"** (martinfowler.com/bliki/TestPyramid.html) — the canonical short article defining the test pyramid concept and the "ice cream cone" anti-pattern by name.
- **pytest documentation** (docs.pytest.org) — official docs on fixtures, parametrization, markers, and plugin ecosystem (`pytest-mock`, `pytest-cov`).
- **Pact documentation** (docs.pact.io) — the official guide to consumer-driven contract testing, the Pact Broker, and provider verification workflows across multiple languages.
- **moto documentation** (docs.getmoto.org) — the AWS service mocking library referenced in this day's lab, useful for integration-layer tests that need realistic AWS behavior without real API calls or costs.

## Reference / lookup

- **JUnit XML format reference** — useful when configuring any CI system's test-report publishing, since nearly every test framework across every language can emit this same format.
- **GitHub Actions: `dorny/test-reporter`** (github.com/dorny/test-reporter) — Marketplace action reference for publishing structured test results in the Actions UI.

## Practice

- Take a Terraform module you've already built earlier in this course (the VPC module referenced in this day's lab is ideal) and write a full Terratest suite for it in a disposable AWS sandbox account — the single highest-value exercise for this day.
- Set up a two-service Pact example locally (a tiny consumer + tiny provider, both stubs) and deliberately break the provider's response shape to watch verification fail — seeing contract testing catch a real break firsthand is far more convincing than reading about it.
