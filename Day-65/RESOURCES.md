# Day 65 — Resources: Deployment Strategies

## Primary (assigned)

- **Argo Rollouts documentation** (argo-rollouts.readthedocs.io) — free, the assigned starting point. Start with "Concepts" and then "Canary" and "BlueGreen" strategy pages directly relevant to today.

## Deepen your understanding

- **Kubernetes documentation — "Deployments" (Strategy section)** (kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) — the authoritative reference on Rolling Update/Recreate mechanics, `maxSurge`/`maxUnavailable` semantics.
- **Argo Rollouts "Analysis" documentation** (argo-rollouts.readthedocs.io/en/stable/features/analysis/) — the full `AnalysisTemplate` reference, including Prometheus, Datadog, and web-based metric providers.
- **Flagger documentation** (docs.flagger.app) — an alternative (CNCF) progressive-delivery controller to Argo Rollouts, useful for seeing how another tool models the same canary/blue-green concepts slightly differently.
- **Martin Fowler — "BlueGreenDeployment" and "CanaryRelease"** (martinfowler.com/bliki/) — short, foundational articles that predate/inform the Kubernetes-specific tooling; good for the underlying concepts independent of any one tool.
- **Unleash — "Feature Flag Best Practices"** (docs.getunleash.io) and **LaunchDarkly documentation** (docs.launchdarkly.com) — both cover targeting rules, percentage rollouts, and kill-switch patterns referenced in file 3.

## Reference / lookup

- **Argo Rollouts `kubectl` plugin reference** (argo-rollouts.readthedocs.io/en/stable/generated/kubectl-argo-rollouts/) — every CLI subcommand (`promote`, `abort`, `retry`, `set image`, etc.).
- **"Expand and Contract" migration pattern** (multiple sources reference this under the term "parallel change" — e.g., martinfowler.com/bliki/ParallelChange.html) — the canonical write-up of the schema-migration pattern in file 3.

## Practice

- Run today's lab's Argo Rollouts canary end-to-end, then deliberately introduce a failing `AnalysisTemplate` condition and watch the automatic rollback — the single best way to internalize "automated" rollback versus "gradual."
- Sign up for a free Unleash Cloud (or self-host via Docker) or a LaunchDarkly free trial and wire a simple boolean flag into a toy app — flipping a flag live and watching behavior change with zero redeploy is the fastest way to viscerally understand the deploy/release distinction.
