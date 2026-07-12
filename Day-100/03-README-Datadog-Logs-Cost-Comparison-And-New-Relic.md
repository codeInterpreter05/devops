# Day 100 — Commercial Observability III: Log Management, Cost vs. Prometheus/Grafana, and New Relic

**Phase:** 3 – Observability | **Week:** W17 | **Domain:** Observability | **Flag:** ⚡ Interview-critical

## Brief

This is the file that directly targets today's interview question — "unlimited budget, Datadog or self-managed Prometheus, justify it" — because it's really asking whether you understand *what you're actually paying for* with a commercial platform, beyond "it has nicer dashboards." Log management economics, the real cost/capability delta versus OSS, and New Relic as a comparison point all feed into giving a genuinely defensible answer instead of a preference.

## Log management in Datadog

Datadog Log Management follows a two-tier model that exists specifically to control cost at scale:

- **Indexed logs** — logs that are fully parsed, tagged, and searchable/facetable in the Log Explorer, and count toward retention-based billing. You choose which logs get indexed (via **indexing filters/exclusion filters**), because indexing 100% of raw log volume at high traffic is prohibitively expensive.
- **Logs without limits / log rehydration** — Datadog can ingest and archive *all* logs cheaply to cold storage (e.g., your own S3 bucket) without indexing them, and later **rehydrate** a specific time range/query back into the indexed, searchable tier on demand — e.g., "we don't need last month's debug logs searchable right now, but if there's an incident review, we can pull that specific hour back in."

Datadog also supports **log pipelines**: a processing chain (grok parser, date remapper, status remapper, trace-ID correlation) that structures raw log lines into faceted attributes, similar in spirit to CloudWatch Logs Insights' `parse` but computed at ingest time rather than query time — meaning search/aggregation on parsed fields in Datadog is fast (pre-processed), whereas Logs Insights `parse` patterns are evaluated per-query on raw data (Day 99). This ingest-time vs. query-time processing distinction is a real architectural difference, not just a UI preference.

A particularly powerful feature: **trace-to-log correlation** — because both traces and logs go through the same Agent and carry the same trace ID, you can jump from a slow span in a flame graph directly to the exact log lines emitted during that span, with zero manual correlation work. This is a concrete example of a commercial platform's value coming from *integration* between pillars (metrics/traces/logs), not from any single pillar being individually superior to a good OSS equivalent.

## Datadog vs. Prometheus/Grafana: the honest cost/capability comparison

| Dimension | Datadog | Self-managed Prometheus + Grafana (+ Loki/Elasticsearch for logs) |
|---|---|---|
| **Pricing model** | Per-host for infra monitoring, per-million-spans for APM, per-GB ingested + retention tier for logs — usage-based, scales directly with your footprint | Infrastructure cost (compute/storage for Prometheus, Loki/ELK, Grafana) — scales with *your* hardware choices, not a per-metric/per-span meter |
| **Time to value** | Agent install + a few annotations gets you dashboards, APM, and log correlation within hours | Requires assembling and integrating multiple separate OSS projects (Prometheus + Alertmanager + Grafana + Loki/Tempo/Mimir) yourself — real integration engineering effort |
| **Cross-pillar correlation** | Native — trace-to-log, trace-to-metric correlation built in via shared tagging/trace IDs | Possible but requires deliberate design (consistent labels across Prometheus/Loki/Tempo, e.g., via the Grafana LGTM stack) — not automatic out of the box |
| **Operational burden** | Effectively zero — Datadog runs and scales its own backend | You own HA, storage retention/compaction, upgrades, and scaling for every component in the stack |
| **Vendor lock-in** | High — dashboards, monitors, log pipelines, and APM configuration are Datadog-proprietary; migrating away means rebuilding all of it | Low — PromQL, Grafana dashboards-as-JSON, and OTel-based instrumentation are portable across backends |
| **Cost trajectory at scale** | Grows roughly linearly (or worse, with high-cardinality tags) with hosts/spans/log volume — can become the single largest line item in an infra budget at real scale | Grows with your infrastructure spend, which historically scales sub-linearly with heavy engineering investment in retention/compaction/cardinality control |

**The genuinely honest answer to "unlimited budget, which do you pick":** with unlimited budget, the deciding factor stops being cost and becomes **speed of insight during an incident** and **engineering time saved**. Datadog's cross-pillar correlation (flame graph → exact log lines → related metric spike, one click each) and out-of-the-box dashboards mean your engineers spend less time building/maintaining observability tooling and more time on the product — and with unlimited budget, that engineering time is the actual scarce resource being optimized for, not the subscription invoice. The strong interview answer names this explicitly: *unlimited budget changes the optimization target from "minimize cost" to "minimize engineer-hours spent building/maintaining tooling and mean-time-to-resolution during incidents," and Datadog wins decisively on both under that framing* — while still acknowledging that in the real world (finite budget), the calculus shifts hard back toward OSS + engineering time as budgets tighten.

## New Relic as an alternative

**New Relic** occupies similar territory to Datadog — a full-stack commercial observability platform (APM, infrastructure monitoring, logs, synthetics, browser/mobile RUM) — with a few notable differences worth naming if asked to compare:

- **Pricing model difference**: New Relic historically emphasized a consumption-based model centered on **data ingested (GB) plus per-user seats**, versus Datadog's more modular per-product (per-host, per-span, per-log-GB) pricing — meaning the "cheaper option" answer actually depends heavily on your specific traffic shape (host count vs. data volume vs. team size), not a blanket statement either way.
- **Unified data platform**: New Relic markets a single underlying datastore (NRDB) that all signal types (metrics/events/logs/traces) land in, queried via **NRQL** (a SQL-like query language) — architecturally, this is closer to "one telemetry database with one query language for everything" versus Datadog's more product-segmented backends.
- **Free tier**: New Relic has historically offered a more generous perpetual free tier (a fixed amount of free data ingest per month) aimed at smaller teams/individual engineers, which matters for someone building a portfolio project or a small startup evaluating options before committing budget.
- **Market position**: both are credible, enterprise-adopted platforms; the real decision in practice usually comes down to existing team familiarity, specific integration needs (does your stack's most awkward component have a first-class integration in one but not the other), and negotiated enterprise pricing rather than a fundamental capability gap.

## Points to Remember

- Datadog's log tiering (indexed vs. archived-with-rehydration) exists specifically because indexing 100% of raw log volume at scale is cost-prohibitive — you deliberately choose what's searchable now vs. cheaply archived for later.
- Trace-to-log correlation (jumping from a flame graph span directly to its log lines) is a concrete example of commercial-platform value coming from cross-pillar *integration*, not any single pillar being better than a good OSS equivalent.
- The Datadog-vs-Prometheus/Grafana decision genuinely trades cost/lock-in against time-to-value/operational burden/cross-pillar correlation — there's no universally correct side.
- With unlimited budget, the optimization target shifts from cost to engineer-hours and incident-resolution speed — which is the framing a strong interview answer should state explicitly.
- New Relic's NRQL/NRDB unified-datastore architecture and its pricing model (ingest + seats) are the concrete points of comparison to name versus Datadog, not a vague "they're basically the same."

## Common Mistakes

- Indexing 100% of log volume in Datadog by default and being surprised by the bill — exclusion/indexing filters exist precisely to prevent this and should be configured deliberately, not left at defaults.
- Answering "Datadog or Prometheus?" with a flat preference instead of naming the actual trade-off dimensions (cost model, lock-in, time-to-value, cross-pillar correlation, operational burden) — the flat-preference answer is the weak signal in an interview.
- Assuming self-managed Prometheus/Grafana/Loki automatically gives you Datadog-equivalent trace-to-log correlation for free — it requires deliberate, consistent labeling/tagging discipline across every component (the Grafana LGTM stack architecture), which is real engineering work, not a checkbox.
- Comparing Datadog and New Relic purely on sticker price without normalizing for your actual traffic shape (host count vs. GB ingested vs. seats) — the "cheaper" platform depends entirely on which resource your workload consumes most.
- Forgetting that high-cardinality custom tags (e.g., tagging metrics/logs with unique request IDs or user IDs) can blow up commercial platform costs just as badly as high-cardinality metrics blow up Prometheus/CloudWatch costs — the cardinality problem isn't unique to OSS tooling.
