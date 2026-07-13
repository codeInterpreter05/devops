# Day 112 — Platform Engineering: Internal Developer Platforms & Platform as Product

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Platform Eng | **Flag:** —

## Brief

"What is an Internal Developer Platform? How does it reduce cognitive load on developers?" is this day's flagged interview question, and it's become one of the most common senior DevOps/Platform Engineer interview questions in the last few years because platform engineering itself has become the industry's answer to "DevOps didn't scale the way we hoped." Understanding IDPs as a discipline — not just "we have a Backstage instance" — is what separates someone who's read the platform-engineering hype from someone who can actually build one.

This day is split into three files:
1. **This file** — what an IDP is, why it exists, and the "Platform as Product" mindset plus the Team Topologies platform-team model.
2. **[02-README-Golden-Paths-And-Self-Service-Infra.md](02-README-Golden-Paths-And-Self-Service-Infra.md)** — golden paths/paved roads and self-service infrastructure mechanics.
3. **[03-README-Backstage-Deep-Dive.md](03-README-Backstage-Deep-Dive.md)** — Backstage's catalog, TechDocs, and Scaffolder.

## Why platform engineering exists: the DevOps cognitive load problem

Classic "you build it, you run it" DevOps pushed infrastructure ownership onto every application team: each team learns Terraform, Kubernetes, CI/CD pipeline authoring, observability setup, and security scanning config — on top of actually building product features. At small scale this works. At the scale of dozens of teams, it produces:

- **Duplicated effort** — every team reinvents its own CI pipeline, its own way of provisioning a database, its own logging setup, often slightly incompatible with everyone else's.
- **Inconsistent security/compliance posture** — thirty teams each configuring their own S3 bucket policies means thirty chances to get it wrong.
- **Cognitive overload** — a backend engineer now needs to be competent in Kubernetes YAML, IAM policy, Terraform modules, and observability dashboards just to ship a feature. This is the actual, named problem platform engineering targets: **reducing the total cognitive load required for a developer to ship safely to production**, by having a dedicated team absorb and abstract away the infrastructure complexity.

## What an Internal Developer Platform (IDP) actually is

An IDP is **the layer of self-service tooling, APIs, and golden-path templates that a dedicated platform team builds and operates so that application teams can provision infrastructure, deploy, and observe their services without needing deep expertise in the underlying systems** (Kubernetes, cloud APIs, CI/CD internals).

Concretely, an IDP usually bundles:

- A **catalog/portal** (e.g., Backstage) — a single place to discover services, ownership, docs, and templates.
- **Self-service provisioning** (e.g., Crossplane, Terraform modules exposed via a portal, or a custom API) — a developer requests and gets it, without filing a ticket to a human and waiting.
- **Golden-path templates/scaffolding** — pre-built, opinionated project skeletons (CI pipeline, Dockerfile, k8s manifests, observability wiring already included).
- **Standardized CI/CD pipelines** — shared, reusable pipeline definitions rather than each team hand-rolling YAML.
- **Built-in observability/security defaults** — logging, tracing, and baseline security scanning wired in by default, not opt-in per team.

The critical distinction from "just a set of scripts" or "just Kubernetes": an IDP is a *product*, with a defined interface (APIs, UI, templates) that app teams consume, and it hides the underlying implementation (which could change — swap Terraform for Crossplane underneath — without app teams noticing).

## Platform as Product: the mindset shift

The single most important idea in modern platform engineering is treating the platform itself as a product, with the platform team as the "vendor" and application developers as **internal customers**:

- The platform team does **product discovery** — talking to internal dev teams about their actual pain points, not just building what infra people think is elegant.
- Adoption is **earned, not mandated** — if golden paths are good enough (faster, safer, less painful) developers choose them voluntarily; if teams keep bypassing the platform and hand-rolling their own infra, that's a signal the product isn't good enough yet, not a discipline problem to be solved by a mandate.
- The platform has **SLAs/reliability expectations** like any product — if the self-service provisioning API is flaky, developers lose trust and route around it.
- Success is measured with product metrics: adoption rate of golden paths, time-to-first-deploy for a new service, developer satisfaction surveys — not just "did we build the thing."

This is a real cultural shift from the "central infra team as gatekeeper" model many organizations still run, where infra is a cost center that says no to tickets rather than a product team that says yes by default through self-service.

## Team Topologies: where the platform team fits

Matthew Skelton and Manuel Pais' *Team Topologies* names four fundamental team types; the one relevant here is the **platform team**: a team that provides internal services (via APIs, tooling, documentation) to reduce the cognitive load of the **stream-aligned teams** (teams that own end-to-end delivery of a specific product/feature area) who consume it. Two other relevant types: **enabling teams** (temporarily help a stream-aligned team adopt a new capability, then leave — different from platform teams, who provide an ongoing service) and **complicated-subsystem teams** (own something that genuinely needs deep specialist knowledge, e.g., a video-encoding engine).

The key Team Topologies insight that maps directly onto IDPs: a platform team's interface to its consumers must be treated as an **API with a documented contract** — Skelton and Pais explicitly call this "platform as internal product," and warn against platform teams becoming an unbounded, ticket-driven bottleneck (which just re-creates the old ops-gatekeeper model with extra buzzwords). If every request to the platform team requires a human in the loop, you haven't actually reduced cognitive load — you've just relocated the bottleneck.

## Points to Remember

- IDPs exist to solve *cognitive load*, not just "automate infra" — the concrete goal is letting a stream-aligned team ship safely without becoming Kubernetes/Terraform/IAM experts themselves.
- An IDP is a bundle: catalog/portal, self-service provisioning, golden-path templates, standardized CI/CD, and built-in observability/security defaults — not any single tool alone.
- "Platform as Product" means internal developers are customers with choice (ideally), the platform team does discovery and has SLAs, and success is measured by adoption/time-to-first-deploy, not by ticket-closure counts.
- In Team Topologies terms, a platform team's job is to reduce cognitive load for stream-aligned teams via a well-defined, self-service API/interface — not to become a ticket queue.
- Distinguish platform teams (ongoing service, self-service interface) from enabling teams (temporary, hands-on help adopting something new) — conflating the two is a common Team Topologies misreading.

## Common Mistakes

- Building an internal platform nobody asked for based on what the infra team thinks is "correct," then being surprised at low adoption — skipping the product-discovery step is the single most common platform engineering failure mode.
- Equating "we stood up Backstage" with "we have a platform" — the catalog/portal is a UI layer; without actual self-service provisioning and golden paths behind it, it's just a directory, not a platform.
- Mandating golden-path adoption top-down instead of making the golden path clearly better (faster, safer, less toil) than the alternative — mandates without genuine value breed workarounds and shadow infrastructure.
- Turning the platform team into a de facto ticket queue (every provisioning request needs a human platform engineer to act) — this recreates the exact ops bottleneck platform engineering was supposed to eliminate, just with a nicer brand.
- Measuring the platform team's success by infrastructure metrics (uptime, cost) alone instead of developer-experience metrics (time-to-first-deploy, golden-path adoption percentage, support ticket volume trend) — you can have a technically excellent platform nobody wants to use.
