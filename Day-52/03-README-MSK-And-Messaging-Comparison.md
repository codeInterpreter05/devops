# Day 52 — AWS Messaging Deep Dive: MSK & Comparing Messaging Systems

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** AWS | **Flag:** —

## Brief

This file closes today's topic with MSK (Managed Kafka) and directly tackles today's interview question: comparing RabbitMQ, SQS, and Kafka. Interviewers ask this not to test trivia recall, but to see if you reason about messaging systems in terms of their actual architectural trade-offs — delivery model, ordering guarantees, operational burden, and the kind of consumer access pattern each is built for — rather than just listing product names.

## MSK — Managed Streaming for Apache Kafka

MSK is AWS running and managing **actual, open-source Apache Kafka** (or MSK Serverless, a fully-managed variant with no broker/capacity management at all) — you get standard Kafka APIs and ecosystem compatibility (Kafka Connect, Kafka Streams, ksqlDB, existing client libraries), but AWS handles broker provisioning, patching, and (for the provisioned variant) some scaling operations.

- **MSK Provisioned**: you choose broker instance types, count, and storage per broker; you're responsible for cluster sizing/partition planning, though AWS handles the underlying infrastructure (patching, replacement of failed brokers, etc.).
- **MSK Serverless**: no broker management or capacity planning at all — scales automatically with throughput, billed per-throughput/storage rather than per-broker-hour. Trade-off: fewer configuration knobs (some Kafka broker-level configs aren't exposed), and it's a newer offering with narrower feature parity than a self-managed or MSK Provisioned cluster.

**Why choose MSK over Kinesis when both are "streaming"**: MSK gives you Kafka's actual ecosystem — if your organization already has Kafka expertise, existing Kafka Connect connectors, Kafka Streams topology code, or you need Kafka-specific semantics (consumer groups with fine-grained partition assignment strategies, exactly-once stream processing via transactions, log compaction for a changelog-style topic), MSK lets you keep all of that while offloading operations to AWS. Kinesis is the "native AWS-first" choice — tighter IAM integration, simpler mental model, less operational surface area, but a narrower ecosystem and API that's AWS-specific rather than portable.

## RabbitMQ vs. SQS vs. Kafka — the actual comparison

The three sit on genuinely different points of the messaging design space, not just "different vendors for the same thing":

| | RabbitMQ | SQS | Kafka (/MSK) |
|---|---|---|---|
| **Model** | Message broker (queue-based, smart broker/dumb consumer) | Managed message queue (queue-based) | Distributed commit log (dumb broker/smart consumer) |
| **Delivery semantics** | At-least-once (or exactly-once with plugins); broker tracks consumer ack | At-least-once (standard) / exactly-once-ish (FIFO) | At-least-once by default; exactly-once achievable with idempotent producers + transactions |
| **Message removal** | Deleted from queue once acked | Deleted once acked (`DeleteMessage`) | **Not deleted on read** — retained per a time/size policy regardless of consumption |
| **Ordering** | Per-queue FIFO (with caveats under multiple consumers) | Best-effort (standard) / per-group strict (FIFO) | Strict, per-partition |
| **Multi-consumer replay** | Not really — once acked/consumed, it's gone (unless using separate queues per consumer, similar to SNS fan-out) | Same limitation as RabbitMQ | **Native** — consumer groups independently track offsets; replay is a first-class, cheap operation |
| **Throughput ceiling** | Moderate (broker-bound, vertical scaling matters) | Near-unlimited (standard), moderate (FIFO) | Very high, horizontally scales via partitions |
| **Routing flexibility** | Very rich (direct/topic/fanout/headers exchanges, complex routing logic in the broker) | None built-in (SNS provides fan-out on top) | Simple (topic + partition key); complex routing typically built at the application/stream-processing layer |
| **Operational model** | Self-hosted or managed (Amazon MQ for RabbitMQ) — you manage broker sizing | Fully managed, zero infrastructure to think about | Self-hosted, or MSK — broker/partition/replication concepts to reason about even when managed |
| **Best fit** | Complex routing needs (RPC-style request/reply, priority queues, flexible exchange topologies), moderate throughput, teams already invested in AMQP | Simple decoupling/work-queue needs, want zero operational burden, don't need replay | High-throughput event streaming, multiple independent consumers, need replay/reprocessing, event sourcing / CDC pipelines |

**The one-sentence mental model interviewers are listening for**: *RabbitMQ is a feature-rich traditional message broker best for complex routing and moderate throughput; SQS is the simplest possible fully-managed queue for basic decoupling with no replay needs; Kafka is a distributed log built for high-throughput, multi-consumer, replayable event streaming.* Everything else (specific numbers, exact delivery guarantees, ecosystem tooling) is supporting detail for that core distinction.

## Concrete "which would you choose" scenarios (rehearse these)

- **"Process each uploaded file exactly once, retry on failure, don't need anyone to replay old uploads"** → SQS. Simple work-queue, no ecosystem needed, zero ops.
- **"Fan out a single order-placed event to 6 different downstream teams' systems, and new teams will be added over time without changing the producer"** → SNS (or EventBridge if the routing needs to be content-based rather than simple topic subscription) fanning into SQS per consumer.
- **"Ingest clickstream/telemetry at very high volume; a real-time fraud model and a batch analytics job both need to read it independently, and analytics needs to reprocess the last few days after a pipeline bug"** → Kafka/MSK (or Kinesis if you want to stay AWS-native and don't need Kafka's specific ecosystem).
- **"Need complex routing — e.g., route messages to different queues based on multiple header attributes with AND/OR logic, or implement an RPC-style request/reply pattern"** → RabbitMQ, because its exchange types (direct, topic, fanout, headers) support this natively in the broker, where SQS/Kafka would require you to build that routing logic yourself in application code.
- **"Team already runs Kafka on-prem and wants to lift-and-shift to AWS without rewriting consumers"** → MSK, for API/ecosystem compatibility.

## Points to Remember

- MSK Provisioned gives you control over broker sizing but you own partition/capacity planning; MSK Serverless removes broker management entirely but with a narrower configuration surface.
- The core mental model: RabbitMQ = rich routing, traditional broker. SQS = simplest fully-managed queue, no replay. Kafka = distributed log, high-throughput, replayable, multi-consumer-native.
- Kafka's defining structural property is that **reading doesn't delete** — retention is time/size-based, independent of how many consumer groups have read the data, which is what makes native replay possible.
- SQS/RabbitMQ's defining structural property is that a message is **claimed and removed** once processed — great for simple work distribution, weak for "multiple independent applications need to read the same data."
- Choosing between Kinesis and MSK when you need AWS-native streaming: MSK for Kafka ecosystem/portability needs, Kinesis for tighter AWS integration and a simpler operational model.

## Common Mistakes

- Choosing Kafka/MSK for a simple point-to-point task queue because "Kafka is powerful," and taking on partition/consumer-group operational complexity that SQS would have handled with zero infrastructure thinking.
- Choosing SQS for a use case that actually needs multiple independent consumers to replay historical data, then building an increasingly complicated workaround (separate queues per consumer, manual replay via re-publishing from a database) that Kafka/Kinesis would have given natively.
- Assuming RabbitMQ and Kafka are interchangeable "just pick whichever" message brokers — RabbitMQ's smart-broker/dumb-consumer model and rich exchange routing solve a different problem than Kafka's dumb-broker/smart-consumer log model; picking based on team familiarity alone without considering the actual access pattern needed.
- Forgetting that Kafka topics need thoughtful **partition count and key design** up front — under-partitioning caps throughput and consumer parallelism, and repartitioning later is disruptive (changes key-to-partition mapping, breaking ordering guarantees for existing keys).
- Treating MSK Serverless as a drop-in replacement for every self-managed Kafka use case without checking feature parity — some broker-level configs and older client behaviors assumed by existing tooling may not be available.
