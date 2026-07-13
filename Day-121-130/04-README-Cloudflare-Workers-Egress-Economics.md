# Day 121-130 — Multi-Cloud & Edge IV: Cloudflare Workers and the Real Economics of Egress

**Phase:** 4 – Advanced/Specialization | **Week:** W21-W22 | **Domain:** Advanced | **Flag:** —

## Brief

This closing file covers two things that pair together more than they first appear to: **Cloudflare Workers** as a genuinely different edge-compute model from "a smaller VM closer to the user," and the **real, dollar-denominated cost of moving data between and out of clouds** — the single most concrete, quotable topic in this entire block for an interview, because most candidates can describe multi-cloud architecture in the abstract but very few can put real numbers on why it's expensive.

## Cloudflare Workers — not a smaller container, a different execution model

Workers do **not** run in containers or VMs. Each Worker executes inside a **V8 isolate** — the same lightweight JavaScript execution sandbox Chrome uses to isolate browser tabs from each other. Isolates share a running V8 process and start in **single-digit milliseconds**, because there's no OS boot, no container runtime, no VM hypervisor involved — this is the actual mechanism behind Workers having no meaningful "cold start" the way Lambda or Cloud Run can. Requests are routed to whichever of Cloudflare's 300+ global points of presence is network-closest to the requester, and the same Worker script runs identically at every one of them — there's no "which region did I deploy to" decision to make, because the answer is "all of them, simultaneously."

```bash
# wrangler is the CLI for the whole Workers lifecycle
npm create cloudflare@latest my-worker
cd my-worker

wrangler dev          # local dev server with a simulated edge runtime
wrangler deploy        # ships to every Cloudflare PoP globally, no region selection
wrangler tail          # live-stream logs from production
```

A minimal `wrangler.toml`:

```toml
name = "edge-router"
main = "src/index.ts"
compatibility_date = "2024-09-01"

[[kv_namespaces]]
binding = "SESSION_KV"
id = "..."

[[r2_buckets]]
binding = "ASSETS"
bucket_name = "edge-assets"
```

### The storage/state bindings, and when to use which

Workers are stateless by default (each request can hit a different PoP) — real applications need at least one of these bindings:

- **Workers KV** — a global, **eventually consistent** key-value store, optimized for read-heavy data that tolerates propagation lag (feature flags, config, cached auth lookups). Writes propagate globally over seconds, not instantly — do not use it for anything requiring strong consistency.
- **Durable Objects** — a single, strongly consistent, stateful actor per unique ID, pinned to one location, with in-memory state and its own attached storage. This is the mechanism for anything needing coordination (WebSocket connection state, rate limiting, a single-writer counter) — the tradeoff for consistency is that all requests for a given object ID funnel through that one instance, so it's not a horizontally-scaled read path.
- **R2** — S3-API-compatible object storage. Its headline feature, and the reason it's relevant to this file specifically: **R2 charges zero egress fees.** This is a deliberate, explicit competitive move against the AWS/GCP/Azure egress-pricing model described below, and it's common for companies to adopt R2 purely as an egress-cost bypass for high-download-volume object storage, independent of whether they use Workers for anything else.
- **D1** — a managed SQLite database at the edge, for relational data with moderate write volume; not a replacement for a full OLTP database at high write concurrency.

### Where Workers genuinely fit in an architecture

Edge compute earns its keep specifically where **latency to the nearest PoP** and **avoiding a round trip to a distant origin** matter: JWT/session validation before a request ever reaches the origin, A/B test bucket assignment, request routing/rewriting, image resizing/transformation, and caching logic with custom invalidation rules. It is a poor fit for anything CPU-heavy or requiring a large runtime/library surface — Workers run a restricted JS/Wasm runtime (no arbitrary native binaries, execution-time and memory limits per invocation) and are not a general-purpose compute replacement for your origin services.

## Cross-cloud data egress — the hidden cost that breaks multi-cloud pitches

Every major cloud's egress pricing follows the same shape: **ingress is free, intra-cloud (mostly) is cheap or free, egress to the internet is metered and tiered, and cross-cloud traffic is billed as ordinary internet egress with no special discount** — there is no "AWS-to-GCP direct peering discount" in the standard pricing model. This last point is the crux of the "hidden cost" answer to the interview question this block is building toward.

### Approximate 2020s-era tiered pricing shape (know the *shape*, not memorized exact digits — providers revise rates)

| Egress path | AWS | GCP | Azure |
|---|---|---|---|
| To internet, first tier | ~$0.09/GB (up to ~10TB/mo) | ~$0.12/GB (varies by destination continent, "Premium" network tier) | ~$0.087/GB (up to ~10TB/mo) |
| Higher-volume tiers | Steps down to ~$0.05/GB at very high volume | Steps down at higher volume | Steps down at higher volume |
| Inter-AZ, same region | ~$0.01/GB **each direction** (often the most-forgotten line item) | Generally free within a zone, low cost cross-zone | Low or no charge within a region, varies by tier |
| Inter-region, same cloud | ~$0.02/GB (varies by region pair) | ~$0.01–0.08/GB | Varies by region pair |
| Ingress (any cloud, any direction) | Free | Free | Free |
| Cloudflare R2 egress | N/A | N/A | N/A — **$0/GB, by design** |

The exact numbers move over time and vary by destination region — the thing worth internalizing cold, not the specific digits, is: **egress is the metered, asymmetric cost every cloud shares, and it applies in full to inter-cloud traffic** because there is no "we recognize this destination as another cloud, here's a discount" tier.

### The concrete hidden costs beyond the headline egress-per-GB rate

- **NAT Gateway processing fees.** Private-subnet nodes routing egress through a NAT Gateway (the standard secure pattern on both AWS and Azure — see file 1) pay a **per-GB-processed** fee **in addition to** the standard internet egress rate, plus an hourly per-NAT-Gateway charge. A workload egressing 50TB/month through NAT Gateways can rack up NAT processing charges that rival or exceed the underlying internet-egress bill itself — this is consistently the line item that surprises teams who only budgeted for the advertised per-GB internet rate.
- **Load balancer data-processing charges.** Both cloud-native L4/L7 load balancers meter GB processed, separate from and in addition to compute and egress — easy to omit from a back-of-envelope multi-cloud cost estimate.
- **Cross-region replication for DR/BC.** A multi-cloud DR strategy that continuously replicates data to a standby in another cloud is paying egress on every byte replicated, continuously, as a standing cost — not a one-time migration cost, which is how it's often mentally modeled during the initial pitch.
- **Centralized observability/log shipping.** Shipping logs/metrics/traces from workloads in Cloud A to a centralized stack in Cloud B (a common ask once teams want "one pane of glass" across a multi-cloud estate) is itself a standing egress cost that scales with log volume — worth sizing before committing to a single centralized observability backend across clouds.

### A napkin-math example worth having ready for an interview

A data pipeline moving **10TB/month** from AWS to GCP and back (10TB out, 10TB back): AWS-side egress on the outbound 10TB at a blended ~$0.07–0.09/GB tier is roughly **$700–900/month**; the return trip incurs GCP-side egress at a similar blended rate for another **$700–1,200/month** (GCP's tiering varies more by destination). Over a year, a *single, moderate-volume* pipeline like this can run **$15,000–25,000/year** in egress alone — before NAT/LB processing fees are added, and before accounting for the fact that this number scales linearly with data volume, which tends to grow, not shrink, as a pipeline matures. This is the concrete number that turns "multi-cloud avoids lock-in" from a slide bullet into a budget line item a CFO will ask about.

## Points to Remember

- Cloudflare Workers run in V8 isolates, not containers/VMs — this is *why* they have near-zero cold start and deploy globally by default rather than to a chosen region.
- Pick the right Workers storage binding for the consistency need: KV for eventually-consistent read-heavy data, Durable Objects for single-writer strongly-consistent coordination, R2 for object storage (with the standout feature of zero egress fees), D1 for light relational needs.
- Every major cloud's egress pricing is tiered and metered for internet-bound traffic, and **cross-cloud traffic gets no special discount** — it's billed as ordinary internet egress on both sides of the hop.
- NAT Gateway and load-balancer data-processing fees are billed **in addition to** the headline egress-per-GB rate and are the most commonly underestimated cost in a multi-cloud or even single-cloud egress-heavy architecture.
- Cross-cloud replication for DR and centralized cross-cloud observability are *standing, recurring* egress costs, not one-time costs — model them as an ongoing monthly line item, not a migration expense.

## Common Mistakes

- Treating Workers as "just a smaller Lambda" and expecting the same deployment model (per-region, container-based) — the whole point is a different execution primitive (isolates) with different scaling and latency properties.
- Reaching for Workers KV for data that needs strong consistency (e.g., a counter used for rate-limiting or billing) — KV's eventual consistency will produce visibly wrong behavior under concurrent writes; that's what Durable Objects are for.
- Pricing a multi-cloud architecture by adding up compute and storage costs only, and omitting egress, NAT processing, and load-balancer data-processing fees entirely — this is the single most common reason a multi-cloud cost estimate is wrong by a large margin once it's actually running.
- Assuming a "direct connection" product (AWS Direct Connect, Google Cloud Interconnect, Azure ExpressRoute) eliminates cross-cloud data costs — these reduce latency/improve reliability for dedicated connectivity, but data transferred over them is still metered and billed, just potentially at a different (not necessarily zero) rate than public internet egress.
- Forgetting that R2's zero-egress model is a deliberate strategic differentiator, not an oversight — when comparing total cost of ownership for an egress-heavy object-storage workload, R2 is a legitimate cost lever to name even independent of adopting Workers for compute.
