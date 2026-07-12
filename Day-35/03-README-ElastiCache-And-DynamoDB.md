# Day 35 — AWS Storage & Databases: ElastiCache Redis Cluster Mode & DynamoDB

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** —

## Brief

Rounding out today's storage/database survey: ElastiCache (managed in-memory caching) and DynamoDB (AWS's managed NoSQL database) are both built around a core idea relational databases aren't — **partitioning data across many nodes to scale far beyond what a single machine can hold or serve**. Understanding *how* each partitions data (Redis's hash slots, DynamoDB's partition key hashing) is what separates "I've used DynamoDB" from "I understand why my DynamoDB table has a hot partition problem."

## ElastiCache Redis — cluster mode disabled vs. enabled

```bash
aws elasticache create-replication-group \
  --replication-group-id my-redis --replication-group-description "cache" \
  --engine redis --cache-node-type cache.r6g.large \
  --num-node-groups 1 --replicas-per-node-group 2   # cluster mode disabled: 1 shard, 2 replicas
```

**Cluster mode disabled** gives you exactly **one shard** (one primary + up to 5 read replicas) — all your data lives on a single primary node's memory, and replicas exist purely for read scaling and failover, not for spreading data volume across more memory. Your maximum dataset size is capped at whatever one node's memory holds.

```bash
aws elasticache create-replication-group \
  --replication-group-id my-redis-cluster --replication-group-description "cache" \
  --engine redis --cache-node-type cache.r6g.large \
  --num-node-groups 3 --replicas-per-node-group 1   # cluster mode enabled: 3 shards, sharded data
```

**Cluster mode enabled** partitions your keyspace across **multiple shards** (each shard has its own primary + replicas), using Redis's **hash slot** model — Redis divides the entire keyspace into 16,384 hash slots, and each shard owns a contiguous range of them; a key's slot is computed via `CRC16(key) mod 16384`. This is what actually lets total data volume scale horizontally — adding a shard means redistributing hash slot ownership across more nodes, increasing both total memory capacity and write throughput (each shard's primary handles its own slots' writes independently).

**The operational implication that matters**: multi-key operations (`MGET`, transactions, Lua scripts touching multiple keys) only work atomically if all involved keys hash to the **same slot** — with cluster mode enabled, an `MGET` across keys on different shards either fails or requires client-side fan-out, depending on your client library. Redis's **hash tags** (`{user:123}:profile` and `{user:123}:sessions` — the `{...}` portion is what's hashed, so both keys land on the same slot) are the standard technique for deliberately co-locating related keys when you need multi-key atomicity under cluster mode.

## DynamoDB — partition keys, GSIs, and streams

### Partition keys — the mechanism that determines EVERYTHING about scaling

```bash
aws dynamodb create-table --table-name Orders \
  --attribute-definitions AttributeName=CustomerId,AttributeType=S AttributeName=OrderId,AttributeType=S \
  --key-schema AttributeName=CustomerId,KeyType=HASH AttributeName=OrderId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
```

DynamoDB hashes the **partition key** value to determine which physical partition stores an item — items with the same partition key always land on the same partition (which is what makes range queries on the sort key within one partition key fast and consistent). DynamoDB automatically splits partitions as data/throughput grows, but a **poorly chosen partition key creates a "hot partition"** — if a huge share of your traffic hits one partition key value (e.g., using a single fixed value, or a low-cardinality attribute like `status: "active"` shared by millions of items), that one physical partition absorbs a disproportionate share of read/write throughput, and you hit that partition's throughput ceiling regardless of how much overall table capacity you've provisioned. This is *the* most common real-world DynamoDB performance problem, and it's purely a data-modeling issue — no amount of extra provisioned capacity fixes a hot-partition design.

### Global Secondary Indexes (GSIs) — querying by something other than the primary key

```bash
aws dynamodb update-table --table-name Orders \
  --attribute-definitions AttributeName=Status,AttributeType=S \
  --global-secondary-index-updates '[{
    "Create": {
      "IndexName": "StatusIndex",
      "KeySchema": [{"AttributeName":"Status","KeyType":"HASH"}],
      "Projection": {"ProjectionType":"ALL"}
    }
  }]'
```

DynamoDB's base table can only be efficiently queried by its partition key (and sort key, if defined) — a **GSI** lets you define an alternate partition/sort key pair over the same data, enabling queries by a different attribute (e.g., querying `Orders` by `Status` instead of `CustomerId`) without a full table scan. GSIs are **eventually consistent** (unlike the base table's option of strongly consistent reads) and maintain their own separate write capacity — a GSI with a poorly chosen key can itself become a hot-partition bottleneck independent of the base table's health, since it's genuinely a separate physical structure being kept in sync asynchronously.

### DynamoDB Streams — a change log for reacting to writes

```bash
aws dynamodb update-table --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

**Streams** capture an ordered, near-real-time log of every item-level modification (insert/update/delete) in a table, retained for 24 hours, consumable by a Lambda trigger or a Kinesis Client Library-based consumer. This is the standard mechanism for event-driven reactions to data changes — e.g., a Lambda triggered on every new `Orders` item to update an aggregate, replicate to another table/region, or emit a domain event to EventBridge (Day 34) — without the writer needing to know about every downstream consumer. `StreamViewType` controls whether you get just the new image, just the old image, or both (`NEW_AND_OLD_IMAGES`) — needed if a consumer must compute a diff (e.g., "what specifically changed").

## Points to Remember

- ElastiCache cluster mode disabled = one shard (data capped by one node's memory, replicas only for HA/reads); cluster mode enabled = multiple shards partitioned by hash slot, which is what actually scales total data volume and write throughput.
- Redis hash tags (`{...}` in a key name) let you deliberately force related keys onto the same hash slot for multi-key atomic operations under cluster mode.
- A DynamoDB partition key choice directly determines scaling behavior — a low-cardinality or hot key creates a throughput ceiling no amount of extra provisioned capacity fixes, since it's a single physical partition problem.
- GSIs let you query by a non-primary attribute but are eventually consistent and maintain independent write capacity — they can develop their own hot-partition problems separate from the base table.
- DynamoDB Streams provide an ordered, 24-hour change log for event-driven reactions to writes (commonly via Lambda), decoupling the writer from downstream consumers.

## Common Mistakes

- Enabling ElastiCache cluster mode and assuming existing single-key client code "just works" — multi-key operations across different hash slots need application-level awareness (hash tags, or client-side fan-out) that isn't needed under cluster mode disabled.
- Choosing a DynamoDB partition key based on what's convenient to query rather than what actually distributes access evenly, creating a hot partition that throttles under load despite ample overall provisioned/on-demand capacity.
- Assuming a GSI's write capacity issues are the base table's fault — a GSI is a genuinely separate structure with its own throughput characteristics and can bottleneck independently.
- Relying on strongly consistent reads against a GSI, not realizing GSIs are eventually consistent by design — only the base table supports the strongly-consistent-read option.
- Forgetting DynamoDB Streams only retain 24 hours of change events — a consumer that's down/broken for longer than that permanently misses those changes, with no built-in replay beyond the retention window.
