# Day 93 — OpenSearch Deep Dive: Shard Sizing & Allocation

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Logging | **Flag:** —

## Brief

This is the file that directly answers the day's interview question: "your OpenSearch cluster is slow and disk is at 85%, walk me through your diagnosis." The overwhelming majority of real OpenSearch performance incidents trace back to shard sizing/allocation problems — too many shards, too few, unevenly distributed, or sitting on nodes that are running out of disk. Understanding what a shard actually is, and how OpenSearch decides where to put one, is what turns "the cluster is slow" into an actual, defensible root-cause diagnosis.

## What a shard actually is

An OpenSearch (and Elasticsearch) index is split into one or more **shards**, and each shard is a complete, independent **Lucene index** — its own inverted index, its own segments, its own resource consumption. Sharding exists for two reasons: it lets an index's total data be **spread across multiple nodes** (horizontal scaling beyond one node's disk/CPU), and it lets a single search **parallelize** across shards (each shard searched concurrently, results merged).

Each **primary shard** can have zero or more **replica shards** — exact copies kept on different nodes, used for (a) high availability (if the node holding the primary dies, a replica is promoted) and (b) read scaling (search requests can be served by primaries or replicas, spreading query load).

## Why shard *count* and *size* are both real problems, in opposite directions

**Too many shards (oversharding)** — a very common mistake, especially from over-eager time-based index creation (a new index every hour "just in case," or leaving default shard counts unchanged as data volume was actually small):
- Every shard has fixed overhead — in-memory data structures, file handles, and per-shard cluster state metadata that every node in the cluster has to track and every cluster-state update has to propagate. Thousands of small shards means thousands of these overheads, even if the total data volume is modest.
- **Cluster state** (which tracks metadata for every index/shard/mapping) has to be broadcast to every node on every change; a cluster with tens of thousands of shards can spend a meaningful amount of time just processing cluster-state updates, becoming sluggish for reasons that have nothing to do with query complexity.
- Symptom: a cluster that "feels slow everywhere," high heap usage on master/cluster-manager nodes, slow cluster state updates, even when individual queries aren't inherently expensive.

**Too few / too large shards (undersharding)**:
- A single shard has a **practical upper size limit** (widely cited guidance: keep individual shards in roughly the **10–50GB** range, sometimes higher for specific well-tested workloads) — beyond that, a single search or a single recovery/relocation operation on that shard takes proportionally longer, and a shard can't be split across more nodes once created (shards are the unit of distribution — an oversized shard can't use more parallelism than one node/CPU core can provide).
- Recovery cost: if a node holding a very large shard fails, recovering (rebuilding from replica, or restoring) that single large shard takes much longer than recovering several smaller ones in parallel across multiple nodes.
- Symptom: specific queries/indices are disproportionately slow, and node/disk recovery after a failure takes a long time.

**The practical target**: choose shard count and rollover thresholds (previous file) together so that individual shards land in that ~10–50GB sweet spot, and periodically re-evaluate — shard count for a new index is typically set at index-creation time and isn't trivially changeable afterward (see `_split`/`_shrink` APIs for the exceptions, which are non-trivial operations themselves).

## Shard allocation — how OpenSearch decides where shards live

The **cluster manager node** (renamed from Elasticsearch's "master node" terminology in OpenSearch) is responsible for shard allocation decisions — deciding which node hosts which primary/replica shard, and rebalancing when nodes join/leave or fail.

Key allocation mechanics worth knowing cold:
- **Disk-based shard allocation watermarks** — OpenSearch won't allocate new shards to a node once its disk usage crosses a **low watermark** (default 85%), will actively try to relocate shards away from a node crossing a **high watermark** (default 90%), and will make the affected indices **read-only** once a node crosses a **flood-stage watermark** (default 95%) — this is precisely the "disk at 85%" scenario in today's interview question: at 85% you're at the low watermark, meaning new shard allocation to that node is already being avoided, and you're one node-fill-up away from read-only indices blocking writes cluster-wide.
- **Shard allocation awareness** (rack/zone awareness) — configuring OpenSearch with awareness attributes (e.g., `node.attr.zone: us-east-1a`) ensures replicas are placed in a *different* zone/rack than their primary, so a single zone outage doesn't take out both a primary and all its replicas simultaneously.
- **Allocation filtering** — explicitly including/excluding nodes from holding certain indices' shards (e.g., `index.routing.allocation.require.box_type: hot` to pin an index's shards only to nodes tagged as "hot" tier hardware) — this is the actual mechanism that implements the hot/warm/cold tiering described in file 1: different node groups are tagged by tier, and ISM's state transitions include allocation-filter actions that move a shard's *required* node attribute from "hot" to "warm" to "cold," physically relocating the shard's data to different hardware as it ages.

## Diagnosing "cluster is slow, disk at 85%" — the actual walkthrough

1. **Check disk watermarks first** — 85% is exactly the low watermark default; check `GET _cluster/allocation/explain` to see if shards are stuck unassigned or if allocation is being throttled due to watermark pressure on specific nodes (not necessarily uniformly across the cluster — a single hot node can trip this while others have headroom).
2. **Check shard count and size distribution** — `GET _cat/shards?v` and `GET _cat/indices?v&s=store.size:desc` to find oversized shards or an excessive shard count relative to data volume and node count.
3. **Check for unassigned shards** — `GET _cluster/health` showing `unassigned_shards > 0` often directly explains both "slow" (reduced effective replica/parallelism) and is frequently *caused by* the same disk pressure from step 1 (OpenSearch refusing to allocate replicas to nodes near watermark).
4. **Check hot/warm tiering and ISM policy health** — confirm cold/older data has actually transitioned out of hot-tier nodes as expected; a stuck or misconfigured ISM policy is a common reason hot nodes fill up faster than intended (data that should have moved to warm/cold nodes never did).
5. **Remediate**: free space (force-merge + fix/accelerate ISM transitions, delete indices past retention that failed to auto-delete), add capacity (more nodes/disk), or rebalance (adjust shard allocation filters/rack awareness if data is unevenly distributed across nodes rather than uniformly full).

## Points to Remember

- A shard is a complete, independent Lucene index; sharding enables both horizontal scaling and search parallelism. Replicas provide HA and read-scaling.
- Oversharding causes cluster-state overhead and sluggishness that has nothing to do with query complexity; undersharding causes disproportionately slow queries/recovery on oversized shards. Target roughly 10–50GB per shard.
- Disk watermarks: 85% (low, stop allocating new shards here) → 90% (high, actively relocate away) → 95% (flood-stage, indices become read-only). "Disk at 85%" in the interview question is precisely the low-watermark threshold — a warning sign the cluster is already reacting to, not a coincidental number.
- Allocation awareness (zone/rack attributes) keeps replicas physically separated from their primaries; allocation filtering (`box_type`/tier attributes) is the actual mechanism implementing hot/warm/cold tiering from ISM.
- `GET _cluster/allocation/explain`, `GET _cat/shards?v`, `GET _cat/indices?v`, and `GET _cluster/health` are the four commands that make a real diagnosis possible instead of guessing.

## Common Mistakes

- Leaving default shard counts unchanged as data volume shrinks or grows dramatically over a project's lifetime, without periodically re-evaluating against the ~10–50GB per-shard target — shard sizing is a decision that needs revisiting, not a one-time setting.
- Treating "disk at 85%" as "still fine, we have 15% left" without knowing that's the exact default low watermark — by the time it's visibly a problem to end users, allocation behavior has already started changing.
- Not configuring shard allocation awareness (zone/rack attributes) — a full availability-zone outage can take out a primary and all its replicas simultaneously if they were never guaranteed to be spread across zones.
- Assuming a slow cluster must have an expensive-query problem and tuning queries first, without first checking shard count/size and disk watermark status — often the actual bottleneck is allocation/cluster-state overhead, not query execution.
- Force-merging or reindexing under disk pressure without first freeing space — some remediation operations themselves temporarily need free disk headroom (e.g., reindex needs room for both old and new copies), and attempting them on an already-tight cluster can make things worse before they get better.
