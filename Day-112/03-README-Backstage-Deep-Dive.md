# Day 112 — Platform Engineering: Backstage — Catalog, TechDocs & Scaffolder

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Platform Eng | **Flag:** —

## Brief

Backstage (open-sourced by Spotify, now a CNCF graduated project) is the de facto reference implementation of an IDP's portal layer, and it's what most companies reach for first when starting a platform engineering initiative — which is exactly why today's hands-on activity has you standing one up. Knowing its three pillars (Software Catalog, TechDocs, Scaffolder) concretely, not just as buzzwords, is directly useful both for the lab and for describing real platform work in an interview.

## Software Catalog — the single source of truth for "what exists and who owns it"

The catalog is a queryable inventory of every **entity** in your organization — services, APIs, resources (databases, queues), and systems (logical groupings) — each described by a YAML file conventionally named `catalog-info.yaml` living in the entity's own repo:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payments-service
  description: Handles payment processing and reconciliation
  annotations:
    github.com/project-slug: myorg/payments-service
    backstage.io/techdocs-ref: dir:.
spec:
  type: service
  lifecycle: production
  owner: team-payments
  system: checkout
  providesApis:
  - payments-api
```

Backstage discovers these via a configured **catalog processor** (typically scanning GitHub/GitLab org repos for `catalog-info.yaml` files) and builds a live graph of ownership, dependencies (`dependsOn`, `providesApis`/`consumesApis`), and lifecycle state. The practical payoff: "who owns this service, and what does it depend on" becomes a UI query instead of a Slack archaeology exercise — which is exactly the kind of organizational-scale problem that only bites once you have more than a handful of services, but bites hard once you do.

## TechDocs — docs-as-code, rendered where the catalog lives

TechDocs solves the "docs live somewhere nobody looks" problem by co-locating documentation with the entity that owns it and rendering it inside the same portal as the catalog entry. Mechanically: you write Markdown under `docs/` in the service's own repo, alongside an `mkdocs.yml`, reference it from `catalog-info.yaml` via the `backstage.io/techdocs-ref` annotation shown above, and Backstage's TechDocs plugin builds it (using MkDocs under the hood) into static HTML displayed directly on that service's catalog page.

Why this design (docs-as-code, built from the same repo, rendered in-portal) beats a separate wiki: docs live in the same PR review/version-control flow as the code they describe (a code change and its doc update ship together, reviewed together), and there's no separate system to keep in sync or that silently rots because updating it isn't part of anyone's normal workflow.

## Scaffolder — golden paths made concrete

The Scaffolder is Backstage's implementation of the "golden path" concept from the previous file: a **software template** (a YAML spec plus a set of file templates) that a developer fills out via a form in the portal, producing a fully wired new repository.

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: python-service-template
  title: New Python Microservice
spec:
  parameters:
  - title: Service details
    required: [name, owner]
    properties:
      name:
        type: string
      owner:
        type: string
  steps:
  - id: fetch
    name: Fetch skeleton
    action: fetch:template
    input:
      url: ./skeleton
      values:
        name: ${{ parameters.name }}
        owner: ${{ parameters.owner }}
  - id: publish
    name: Publish to GitHub
    action: publish:github
    input:
      repoUrl: github.com?owner=myorg&repo=${{ parameters.name }}
  - id: register
    name: Register in catalog
    action: catalog:register
    input:
      repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
      catalogInfoPath: /catalog-info.yaml
```

Three-step pattern to note: **fetch** (template a skeleton repo with the parameters the developer supplied), **publish** (create the real repo on GitHub/GitLab), **register** (automatically add the new service to the catalog). This is exactly why the earlier file's claim — "catalog registration should happen automatically as part of scaffolding, not a manual follow-up" — is achievable in practice: `catalog:register` is a first-class, standard Scaffolder action, not a custom hack.

## Points to Remember

- Catalog entities are described declaratively (`catalog-info.yaml`) and discovered automatically from your repos — ownership and dependency graphs come from this metadata, not a separately maintained spreadsheet.
- TechDocs is docs-as-code: Markdown plus MkDocs, stored in the same repo as the code, rendered inside the portal on the entity's own page — this is what keeps docs from rotting compared to a disconnected wiki.
- The Scaffolder's three-step golden-path pattern is fetch → publish → register — a new service should land in the catalog automatically as part of creation, not as a manual afterthought.
- Backstage itself is a frontend/aggregation layer — it doesn't provision infrastructure on its own; Scaffolder template actions typically call out to your actual provisioning system (Terraform, Crossplane, a CI pipeline) to do the real work.
- Backstage plugins are how it extends into your specific stack (Kubernetes plugin, cost-insights plugin, custom internal plugins) — a bare Backstage install is a shell; real value comes from wiring it into your org's actual tools.

## Common Mistakes

- Standing up Backstage's catalog but never enforcing that new repos include a `catalog-info.yaml` — the catalog silently drifts out of sync with reality (services exist that aren't in it) and loses trust as a "source of truth."
- Writing Scaffolder templates that only create the repo (fetch and publish) and skip the `catalog:register` step — every new service then needs a manual catalog PR, quietly recreating the exact toil the Scaffolder was meant to remove.
- Letting TechDocs content go stale because it's treated as a one-time setup task rather than a docs-as-code practice enforced the same way code review is — the "docs live next to code" design only pays off if PRs are actually expected to update docs.
- Underestimating the backend setup: Backstage's catalog/TechDocs/Scaffolder need a real database (Postgres in production, not the default SQLite used for local dev) and auth integration — treating a quick local demo as production-ready is a common early misstep.
- Building custom one-off internal portals before trying Backstage's plugin ecosystem — a large fraction of "we need custom platform UI" needs are already covered by existing community plugins.
