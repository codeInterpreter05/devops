# Day 113 — Crossplane & Self-Service Infra: Beyond Crossplane — Kratix & Port

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Platform Eng | **Flag:** —

## Brief

Crossplane solves "provision infrastructure via Kubernetes CRDs," but a real platform needs two more things it doesn't provide by itself: a way to deliver *entire platform capabilities* consistently across many clusters, and a human-friendly UI layer so developers don't have to write YAML at all. Kratix and Port are two answers to those adjacent problems, and knowing where they fit relative to Crossplane (and to Backstage from yesterday) rounds out a realistic picture of a production IDP stack.

## Kratix — platform-as-a-product delivery across multiple clusters

Kratix (from Syntasso) addresses a problem Crossplane alone doesn't: **how do you package and deliver a repeatable "platform capability"** — e.g., "a fully configured Postgres-as-a-service offering, including monitoring, backups, and access policy" — **as a versioned, installable unit across many different Kubernetes clusters**, rather than hand-wiring Compositions cluster by cluster?

Its core abstraction is the **Promise** — a bundle that declares: the API schema a developer-facing request should look like (similar in spirit to a Crossplane XRD); a **pipeline** (a sequence of container-based steps, not a fixed template) that turns a request into whatever needs to happen — and which can itself call Crossplane, Terraform, Helm, or arbitrary scripts, since Kratix doesn't care what's inside the pipeline; and **scheduling** logic for which of potentially many worker clusters should actually fulfill a given request.

The multi-cluster angle is the differentiator: Kratix explicitly models a split between a **platform cluster** (where Promises are registered and requests land) and one or more **worker clusters** (where the actual work happens, and which can be selected dynamically per-request based on labels, capacity, or region). This matters for organizations running many clusters (per-team, per-region, per-environment) who want one consistent "request a database" experience regardless of which cluster ultimately serves it — a problem Crossplane's single-cluster-centric model doesn't directly solve without extra plumbing.

## Port — the UI/UX layer for self-service, as a managed alternative to building your own

Port is a commercial internal developer portal — conceptually competing with (and often integrated alongside) Backstage, but delivered as a managed SaaS product rather than something you self-host and operate. It provides the same core pillars covered on Day 112 (software catalog, self-service actions/forms, scorecards for maturity/compliance tracking) without a platform team needing to run and maintain a Backstage instance's backend, plugins, and upgrades themselves.

Where Port is commonly used in a Crossplane-based stack: Port's **self-service actions** feature lets a platform team expose a form (e.g., "Request a new database — pick size and region") that, on submission, triggers a webhook/GitHub Action/pipeline which applies a Crossplane `Claim` YAML on the developer's behalf — Port becomes the friendly UI in front of the same Claim/XRD/Composition machinery from the earlier files, so a developer never needs to write or even see YAML directly.

## How these fit together as a stack

A realistic modern platform stack layers roughly like this:

- **Provisioning engine**: Crossplane (Claims/XRDs/Compositions) and/or Terraform, doing the actual cloud API work.
- **Multi-cluster capability delivery** (if you operate many clusters): Kratix Promises orchestrating which cluster fulfills which request, potentially calling Crossplane underneath.
- **Developer-facing portal/UX**: Backstage (self-hosted, deeply customizable, free/open-source) or Port (managed, faster to stand up, subscription cost) — either can trigger the provisioning layer via forms/templates.
- **Catalog and docs**: usually Backstage's Software Catalog/TechDocs, or Port's equivalent catalog features, giving ownership/discovery on top of everything else.

None of these tools are strict substitutes for each other despite surface-level similarity (both Backstage and Port do "catalog plus self-service forms"; both Crossplane and Kratix do "turn a request into infrastructure") — the practical decision is usually build-vs-buy (Backstage vs. Port) for the UI layer, and single-cluster-vs-multi-cluster (Crossplane alone vs. Crossplane-under-Kratix) for the delivery layer.

## Points to Remember

- Kratix's unit of delivery is a **Promise**: API schema + pipeline + scheduling — designed for delivering a platform capability consistently across *multiple* clusters, a problem single-cluster Crossplane Compositions don't directly address.
- Kratix doesn't replace Crossplane — a Kratix Promise's pipeline commonly *calls* Crossplane (or Terraform, or Helm) to do the actual provisioning; Kratix is an orchestration/delivery layer above it.
- Port is a managed SaaS alternative to self-hosting Backstage, covering catalog, self-service actions, and scorecards, trading operational ownership for subscription cost and faster time-to-value.
- A common real integration: Port (or Backstage) presents a form; submitting it applies a Crossplane `Claim` (possibly via a Kratix Promise if multi-cluster) behind the scenes — the UI layer and provisioning layer are separate, composable concerns.
- None of these tools are 1:1 substitutes despite overlapping surface features — the real decision axes are build-vs-buy (portal layer) and single-cluster-vs-multi-cluster (delivery layer).

## Common Mistakes

- Assuming Kratix is "Crossplane but better" or a competitor — it operates one layer up (multi-cluster capability delivery/orchestration) and typically has Crossplane running underneath it, not instead of it.
- Assuming Port and Backstage are interchangeable with zero tradeoffs — Port trades self-hosting effort and customization depth for speed and a subscription cost; Backstage trades setup/ops burden for full control and no licensing cost, and the "right" choice depends on team size and how much bespoke customization is actually needed.
- Introducing Kratix's multi-cluster orchestration complexity for a platform that only runs one or two clusters — it solves a real problem, but only once you have the multi-cluster scale problem to solve; adding it prematurely is unneeded operational overhead.
- Treating the UI/portal layer as the hard part and the provisioning layer as an afterthought — in practice the guardrails and reliability of the underlying Crossplane Compositions/Kratix Promises matter far more to safety and adoption than the frontend's look and feel.
- Forgetting that whichever UI layer you pick, the actual security boundary is still enforced at the provisioning layer (Crossplane RBAC/Composition schema, or Kratix Promise pipeline validation) — a pretty form in Port/Backstage doesn't add safety by itself if the backend Claim schema allows unsafe parameters.
