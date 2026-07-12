# Day 93 — OpenSearch Deep Dive: ILM, Rollover Strategy & ISM

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Logging | **Flag:** —

## Brief

Yesterday you learned why Loki is cheap because it barely indexes anything. Today's flip side: when you *do* run a full-text-indexing system like OpenSearch (or you inherited one, which is extremely common — OpenSearch/Elasticsearch remain the dominant choice for security/audit logs, SIEM, and rich ad-hoc search), the entire operational challenge becomes **managing the lifecycle of indices** so that a system designed for fast search doesn't slowly suffocate under its own accumulated data. Index lifecycle management is the single most common thing that goes wrong in unmanaged OpenSearch clusters — "disk full" and "cluster red" incidents trace back to this far more often than to query complexity.

This day is split into three focused files:

1. **This file** — Index Lifecycle Management (ILM) policies, rollover strategy, and ISM (OpenSearch's ILM implementation).
2. **[02-README-Shard-Sizing-And-Allocation.md](02-README-Shard-Sizing-And-Allocation.md)** — shard sizing and allocation.
3. **[03-README-Cross-Cluster-Replication-And-Alerting.md](03-README-Cross-Cluster-Replication-And-Alerting.md)** — cross-cluster replication and alerting in OpenSearch.

## Why indices need a lifecycle at all

Elasticsearch/OpenSearch indices are not designed to grow forever — a single index that accumulates data indefinitely eventually becomes too large to search efficiently, too large to reindex/repair if corrupted, and too expensive to keep entirely on fast (hot) storage. The standard pattern is **time-based indices** (e.g., `logs-2026.07.12`, one index per day) combined with an automated policy that moves each index through a sequence of **states** as it ages, changing its storage tier, replica count, and eventually deleting it — without a human manually running these operations every day.

## ISM — Index State Management (OpenSearch's ILM)

OpenSearch's native feature for this is called **ISM** (Elasticsearch calls the equivalent feature ILM — Index Lifecycle Management; OpenSearch forked from Elasticsearch 7.10 and renamed/reimplemented several X-Pack features as open-source equivalents, ISM being one of them). An ISM policy defines a set of **states**, the **actions** performed on entering each state, and the **transitions** (conditions) that move an index from one state to the next:

```json
PUT _plugins/_ism/policies/logs_policy
{
  "policy": {
    "description": "Hot-warm-cold-delete lifecycle for application logs",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [{ "rollover": { "min_size": "30gb", "min_index_age": "1d" } }],
        "transitions": [{ "state_name": "warm", "conditions": { "min_index_age": "7d" } }]
      },
      {
        "name": "warm",
        "actions": [
          { "replica_count": { "number_of_replicas": 1 } },
          { "force_merge": { "max_num_segments": 1 } }
        ],
        "transitions": [{ "state_name": "cold", "conditions": { "min_index_age": "30d" } }]
      },
      {
        "name": "cold",
        "actions": [{ "read_only": {} }],
        "transitions": [{ "state_name": "delete", "conditions": { "min_index_age": "90d" } }]
      },
      {
        "name": "delete",
        "actions": [{ "delete": {} }]
      }
    ]
  }
}
```

This is the "hot (7d) → warm (30d) → cold (90d) → delete" pattern named in today's hands-on activity, and it maps directly onto how you'd expect to actually query this data:
- **Hot**: newest data, on the fastest storage (often local NVMe), actively being written to and most frequently queried (dashboards, active investigations) — kept small/rotated frequently via rollover so no single index gets too large to manage efficiently.
- **Warm**: no longer being written to, queried less frequently, replica count often reduced (fewer replicas = less redundant storage cost, acceptable since it's not actively serving hot-path traffic) and **force-merged** to fewer segments (fewer, larger Lucene segments compress better and search faster for read-mostly data — see file 2 for what a segment actually is).
- **Cold**: rarely queried, often moved to cheaper/slower storage tiers, made **read-only** to guarantee nothing corrupts settled historical data and to allow further storage optimizations that assume immutability.
- **Delete**: data past its retention requirement is removed entirely, keeping total cluster size bounded — this is usually driven by actual compliance/retention requirements (e.g., "keep audit logs 90 days"), not just cost.

## Rollover — how "hot" stays small

**Rollover** is the mechanism that keeps any single index from growing unbounded within the hot state: instead of writing forever into one index, you write into an **alias** (e.g., `logs-write`) that points at the *current* backing index, and a rollover action creates a **new** backing index (and repoints the alias to it) once a condition is met — `min_size` (index has grown past N GB), `min_doc_count` (too many documents), or `min_index_age` (simply too old), whichever triggers first.

```json
PUT _plugins/_ism/policies/logs_policy/some-index-000001
```
Applications always write to the alias (`logs-write`), never to a specific dated/numbered index directly — this is what makes rollover transparent to producers: they never need to know or care that the underlying index changed underneath them.

**Why rollover thresholds matter operationally**: too small a `min_size` (e.g., 1GB) creates an excessive number of tiny indices, each with its own overhead (shard/segment/metadata cost per index adds up — see file 2 on shard sizing); too large a threshold means a single index takes a long time to become eligible for the warm-state optimizations (force-merge, replica reduction), and a single oversized index is slower to search and riskier to recover if something goes wrong with it. The commonly cited sweet spot for a single shard's size is in the tens of GB (commonly cited as 10–50GB per shard) — rollover thresholds should be chosen with that shard-sizing target in mind, not picked arbitrarily.

## Points to Remember

- ISM is OpenSearch's fork/reimplementation of Elasticsearch's ILM — same core concept (states, actions, transitions), different product name and open-source licensing lineage.
- The hot → warm → cold → delete pattern maps directly to query frequency and storage cost: hot = fast storage + frequent access, warm = reduced replicas + force-merged, cold = read-only + cheap storage, delete = past retention requirement.
- Rollover keeps individual indices from growing unbounded by writing through an alias and cutting over to a new backing index once a size/age/doc-count threshold is hit — applications never write to a specific dated index directly.
- Rollover thresholds should be chosen with target shard size in mind (commonly 10–50GB per shard), not set arbitrarily — too small creates index-count overhead, too large delays lifecycle optimizations and slows recovery.
- Force-merge (in the warm-state example) reduces segment count for read-mostly data, improving both compression and search speed — but it's an expensive, one-time operation that should only run on indices no longer being actively written to.

## Common Mistakes

- Writing directly to a specific dated index (`logs-2026.07.12`) from application code instead of through a rollover alias — this defeats the entire rollover mechanism and typically means someone has to manually intervene once that index grows too large.
- Setting rollover `min_size` far too small, producing hundreds of tiny indices — each index carries its own cluster-state/shard/segment overhead, and cluster state itself becomes bloated and slow to propagate once index count grows into the thousands.
- Force-merging an index that's still receiving writes — force-merge is meant for read-mostly, no-longer-written indices; running it against an actively-written index is expensive and counterproductive, since new writes will just create new segments again immediately.
- Setting an ISM policy's delete transition based purely on convenience rather than actual compliance/retention requirements — either deleting data before a legal/audit retention window expires, or (more commonly) never setting a delete stage at all and letting cold data accumulate forever, quietly consuming disk until the cluster runs out of space.
- Forgetting that changing an ISM/ILM policy doesn't retroactively apply to already-existing indices unless you explicitly re-associate them — a policy edit only affects newly rolled-over indices going forward unless applied deliberately to existing ones.
