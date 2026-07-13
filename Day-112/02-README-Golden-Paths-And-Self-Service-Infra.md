# Day 112 — Platform Engineering: Golden Paths, Paved Roads & Self-Service Infrastructure

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Platform Eng | **Flag:** —

## Brief

If Platform as Product is the *mindset*, golden paths and self-service infrastructure are the *mechanism* — the concrete artifacts that make an IDP something developers actually experience day to day rather than a slide in a strategy deck. This is the part of platform engineering that's most directly buildable and demoable, which is why it dominates the hands-on side of platform engineering work (and today's lab).

## Golden paths (a.k.a. paved roads)

A **golden path** is an opinionated, supported, well-documented way to accomplish a common task — spin up a new service, add a database, deploy to production — that is deliberately the *easiest* path available, so that developers choose it not because they're forced to, but because doing it any other way is more work.

The "paved road" metaphor (from Spotify's engineering culture writing, and widely adopted since) is precise: you *can* walk through the field instead of the paved road, and platform teams generally don't block you from doing so (a permissive "unpaved paths are possible but unsupported" stance rather than a hard technical block) — but the paved road is faster, smoother, and clearly marked, so most people take it by default. This is a deliberate design choice over "guardrails that forbid everything else," because rigid technical enforcement breeds resentment and workarounds, while a genuinely better default breeds voluntary adoption.

Concretely, a golden path for "create a new backend service" typically bundles, pre-wired, in one scaffolded output:

- Repository structure and boilerplate code for the chosen language/framework.
- CI pipeline already configured (build, test, security scan, image push).
- Kubernetes manifests/Helm chart with sane defaults (resource requests/limits, liveness/readiness probes, PodDisruptionBudget).
- Observability wiring (structured logging config, trace exporter, standard dashboards auto-created).
- Access to secrets/config via the org's standard mechanism (Vault, SOPS, cloud secret manager) already plumbed in, not left as a TODO.
- Registration in the service catalog (ownership, on-call, docs) done automatically as part of scaffolding, not a manual follow-up step.

The measure of a good golden path isn't "does it cover every possible use case" — it's "does it get 80% of teams to a safe, production-ready starting point with near-zero manual setup," while still being escapable/customizable for the teams with genuinely different needs.

## Self-service infrastructure

Self-service means a developer can provision what they need (a database, an S3 bucket, a message queue, a new environment) **without opening a ticket and waiting for a human on the platform/infra team to act.** This is the operational core that makes golden paths actually self-service rather than "well-documented but still requires a Slack message to platform-eng."

Two common implementation patterns:

1. **Kubernetes-native self-service** — expose infrastructure as Kubernetes Custom Resources (this is exactly what Crossplane does, covered in depth on Day 113): a developer applies a CRD like `kind: PostgreSQLInstance`, and a controller reconciles that into real cloud resources, with policy (allowed instance sizes, regions, tagging) enforced by the platform team's compositions/webhooks rather than by a human reviewing each request.
2. **Portal-driven self-service** — a UI (Backstage software templates, or a custom internal portal) presents a form; submitting it triggers a pipeline (often Terraform or Crossplane under the hood) that provisions the resource and reports back a connection string/URL, again with no human approval step for anything within the pre-approved guardrails.

The critical design property in both patterns is **guardrails baked into the self-service mechanism itself**, not enforced by manual review: a developer literally *cannot* request an oversized, non-compliant, or unbudgeted resource, because the self-service interface only exposes the parameters the platform team has decided are safe to expose (e.g., an S3 bucket template that always turns on encryption and blocks public access, with no toggle to disable it). This is what "guardrails, not gates" means in practice — the unsafe option isn't behind an approval process, it simply doesn't exist as an option.

## Points to Remember

- A golden path is the *easiest*, most supported way to do something — adoption should come from it genuinely being better, not from blocking every alternative.
- A good golden path bundles CI, deployment config, observability, secrets access, and catalog registration together, pre-wired — "80% of teams, near-zero manual setup" is the target, not 100% coverage of every edge case.
- Self-service means no human-in-the-loop ticket for standard requests — implemented either as Kubernetes-native CRDs (Crossplane-style) or a portal-driven form (Backstage templates) backed by an automated provisioning pipeline.
- Guardrails should be baked into what the self-service interface *exposes* (parameters, allowed values) rather than enforced by after-the-fact manual approval — "can't request it" beats "requested it and got rejected."
- The "paved road, not a walled garden" framing matters in interviews: golden paths are optional-but-obviously-better, not the only technically possible path.

## Common Mistakes

- Building a golden path so rigid it can't accommodate legitimate edge cases, forcing teams with valid different needs to fight the platform instead of extending it — golden paths need a documented, supported escape hatch, not just a locked box.
- Calling something "self-service" when it's actually "self-service to submit a request that a human still manually approves" — that's just a nicer ticket form, not real self-service, and it doesn't remove the bottleneck.
- Putting all the guardrail logic in documentation ("please remember to enable encryption") instead of in the self-service mechanism itself (a template/composition that only ever produces encrypted resources) — documented guardrails get skipped; structural guardrails can't be.
- Treating golden-path adoption as a one-time migration instead of an ongoing product — golden paths need to evolve as the underlying platform changes, or teams that adopted early end up stuck on a stale, unsupported pattern.
- Under-investing in the "escape hatch" story — if going off the golden path means zero support and zero documentation, teams with genuine edge cases either can't ship or quietly become unsupported/unsafe shadow infrastructure.
