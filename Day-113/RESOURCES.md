# Day 113 — Resources: Crossplane & Self-Service Infra

## Primary (assigned)

- **Crossplane documentation** (docs.crossplane.io) — the assigned starting point, free. The "Composition" and "Getting Started" sections map directly onto today's core concepts and today's lab.

## Deepen your understanding

- **Crossplane's official "Compositions" guide** — a deeper, example-heavy walkthrough of XRDs and Compositions than the quickstart docs, worth reading in full once the basic S3/RDS lab clicks.
- **Upbound's blog** (upbound.io/blog) — Upbound is the primary commercial steward of Crossplane; their posts frequently cover real-world Composition patterns and the reasoning behind newer features (like Composition Functions, which replace the older patch-and-transform model).
- **Kratix documentation** (kratix.io/docs) — the authoritative source on Promises, pipelines, and the platform/worker-cluster split covered in file 3.
- **Port's own documentation** (docs.getport.io) — useful for seeing the self-service-actions feature described in file 3 from the vendor's own perspective, alongside Backstage's docs from yesterday for comparison.

## Reference / lookup

- **Crossplane API reference** (docs.crossplane.io/latest/concepts) — exact schema reference for `Provider`, `ProviderConfig`, `CompositeResourceDefinition`, and `Composition` objects.
- **Upbound Marketplace** (marketplace.upbound.io) — browse available Providers and pre-built Compositions before writing your own from scratch.

## Practice

- **Crossplane's official "Configure Providers" and "Compose Infrastructure" tutorials** — short, guided, hands-on exercises that closely mirror today's lab structure (install provider, provision a resource, then build a Composition).
- **crossplane.io's community Slack #office-hours channel archives** — real troubleshooting threads covering exactly the kind of drift/deletion-policy edge cases called out in today's Common Mistakes sections.
