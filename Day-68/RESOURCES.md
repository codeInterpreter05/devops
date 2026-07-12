# Day 68 — Resources: Policy as Code

## Primary (assigned)

- **Kyverno documentation** (kyverno.io/docs) — free, the assigned starting point for this day. Covers policy types (validate/mutate/generate), enforcement modes, and a large library of ready-made policies to adapt.

## Deepen your understanding

- **Open Policy Agent documentation + "Rego Playground"** (openpolicyagent.org, play.openpolicyagent.org) — the official Rego language reference, and a browser-based sandbox to test Rego without installing anything — the fastest way to build intuition for iteration/unification.
- **Gatekeeper library** (github.com/open-policy-agent/gatekeeper-library) — a large collection of production-ready `ConstraintTemplate`s maintained by the OPA community; a great way to see idiomatic Rego for common policies instead of writing everything from scratch.
- **Conftest documentation** (conftest.dev) — official docs covering the Terraform/Kubernetes/Dockerfile input flows and CI integration examples.
- **"Policy-based Control for Cloud Native Environments" (CNCF blog/talks)** — good conceptual framing for why Policy as Code exists as a category and how Gatekeeper/Kyverno fit into the broader Kubernetes admission-control picture.

## Reference / lookup

- **Kyverno policy library** (kyverno.io/policies) — searchable, ready-to-adapt policies (the exact five common ones covered today, and many more) with copy-paste YAML.
- **Rego built-in function reference** (openpolicyagent.org/docs/latest/policy-reference) — the full list of built-ins (`sprintf`, `count`, `sum`, string/regex functions) you'll reach for constantly once writing real policies.

## Practice

- **Rego Playground** — work through the "Kubernetes admission control" example templates directly in the browser; it's the fastest iteration loop for building Rego fluency without any local setup.
- Take one existing Helm chart's rendered output (`helm template .`) and run it through Conftest with the five common policies from this day — a realistic exercise in catching real-world misconfigurations in a chart you didn't write yourself.
