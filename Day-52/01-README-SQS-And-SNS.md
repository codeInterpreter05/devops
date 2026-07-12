# Day 52 — AWS Messaging Deep Dive: SQS & SNS

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** AWS | **Flag:** —

## Brief

Almost every non-trivial distributed system needs to decouple producers from consumers, and AWS gives you a spectrum of managed messaging primitives to do it — SQS for point-to-point queuing, SNS for pub/sub fan-out, EventBridge for event routing with rich filtering, and Kinesis/MSK for high-throughput streaming. Today's interview question ("compare RabbitMQ, SQS, and Kafka — when would you choose each?") is a staple because it tests whether you understand *why* these systems differ, not just their feature lists. This file covers the two most foundational building blocks: SQS and SNS.

This day is split into three files:

1. **This file** — SQS (standard vs. FIFO, visibility timeout, DLQ) and SNS (fan-out).
2. **[02-README-EventBridge-And-Kinesis.md](02-README-EventBridge-And-Kinesis.md)** — EventBridge and Kinesis Data Streams.
3. **[03-README-MSK-And-Messaging-Comparison.md](03-README-MSK-And-Messaging-Comparison.md)** — MSK basics and RabbitMQ vs. SQS vs. Kafka.

## SQS — Simple Queue Service

SQS is a **point-to-point message queue**: a producer sends a message, it sits in the queue, and (ideally) exactly one consumer picks it up and processes it. It's fully managed — no brokers to run, no capacity planning, scales transparently.

### Standard vs. FIFO queues

| | Standard | FIFO |
|---|---|---|
| Ordering | **Best-effort**, not guaranteed | **Strict order** within a message group |
| Delivery | **At-least-once** — duplicates possible | **Exactly-once processing** (within a 5-minute dedup window) |
| Throughput | Nearly unlimited | 300 msg/s per API call without batching, 3,000 msg/s with batching (per queue, or higher with high-throughput mode) |
| Naming | Any name | Must end in `.fifo` |

**Why standard queues allow duplicates and reordering**: SQS standard queues replicate messages across multiple servers for redundancy/availability; that distributed architecture means it cannot cheaply guarantee a strict global order or exactly-once delivery without sacrificing throughput — so it explicitly documents "at-least-once, best-effort ordering" and pushes the responsibility of **idempotent processing** onto the consumer. This is the single most important operational fact about standard SQS: **your consumer code must tolerate processing the same message twice** (e.g., using an idempotency key, a database `UPSERT`, or a dedup table) — assuming exactly-once semantics on a standard queue is a bug waiting to happen.

**FIFO queues** guarantee order *within a message group* (via `MessageGroupId` — messages with the same group ID are strictly ordered relative to each other; different group IDs can be processed in parallel by different consumers) and dedupe within a 5-minute window (via `MessageDeduplicationId`, either explicit or auto-derived from a content hash if you enable content-based deduplication). This is why FIFO is used for things like per-account financial transaction ordering (`MessageGroupId = accountId`) where cross-account parallelism is fine but within-account order matters — but the throughput ceiling means it's the wrong choice for raw high-volume ingestion.

### Visibility timeout — the mechanism behind "at-least-once"

When a consumer receives a message, SQS doesn't delete it — it makes the message **invisible** to other consumers for a configurable **visibility timeout** (default 30s). If the consumer successfully processes the message and calls `DeleteMessage` within that window, the message is gone for good. If the consumer **crashes, times out, or never calls delete**, the message becomes visible again after the timeout expires and another consumer picks it up — this is the entire mechanism behind SQS's at-least-once guarantee, and it's also exactly why duplicates happen: if your consumer finishes the work but crashes *before* calling `DeleteMessage`, the message reappears and gets processed again even though the work was already done.

```bash
aws sqs change-message-visibility --queue-url <URL> --receipt-handle <HANDLE> --visibility-timeout 300
```
You extend the visibility timeout mid-processing (e.g., via a heartbeat pattern in a long-running job) if you know a job will take longer than the default — otherwise the message reappears and gets double-processed while the original consumer is still working on it.

### Dead-Letter Queue (DLQ)

A DLQ is a second SQS queue configured as the destination for messages that fail processing repeatedly — after a message's `ApproximateReceiveCount` exceeds `maxReceiveCount` (the "redrive policy"), SQS automatically moves it to the DLQ instead of leaving it to loop forever between "delivered, fails, becomes visible again."

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123456789012:my-queue-dlq",
  "maxReceiveCount": 5
}
```
```bash
aws sqs set-queue-attributes --queue-url <URL> --attributes file://redrive-policy.json
```
**Why DLQs matter operationally**: without one, a single malformed/poison message (one that always throws an exception in your consumer) loops forever — reappearing every visibility timeout, endlessly consuming worker capacity and cluttering logs with the same failure, while never actually being resolved. A DLQ isolates it after N attempts so you can inspect and fix it out-of-band, without blocking the healthy message flow behind it.

## SNS — Simple Notification Service

SNS is a **pub/sub** system: a producer publishes one message to a **topic**, and SNS delivers a copy to **every subscriber** of that topic — email, SMS, HTTP(S) endpoints, Lambda, mobile push, or (most importantly for architecture patterns) **SQS queues**.

### The fan-out pattern (today's hands-on activity)

```
S3 event → SNS topic → SQS queue A → Consumer A (e.g., image resize)
                     → SQS queue B → Consumer B (e.g., analytics)
                     → SQS queue C → Lambda (e.g., notification)
```
Publishing once to SNS and fanning out to multiple SQS queues (rather than publishing directly to multiple queues, or worse, having one consumer call the others) means:
- **Producers don't need to know who's downstream** — adding a fourth consumer later is just "subscribe a new queue to the topic," zero changes to the producer.
- **Each consumer processes independently**, at its own pace, with its own retry/DLQ semantics — a slow analytics consumer doesn't block the fast image-resize consumer.
- **SQS in front of each consumer adds durability + buffering** — if a consumer is down for maintenance, messages queue up in its dedicated SQS queue rather than being lost (SNS itself has no built-in retry/buffer for a consumer that's simply offline the way SQS does).

**SNS message filtering**: subscribers can attach a **filter policy** so they only receive a subset of messages published to the topic (matched against message attributes), avoiding the need for every consumer to receive and then discard irrelevant messages.

```json
{
  "event_type": ["order_created", "order_updated"]
}
```

## Points to Remember

- Standard SQS = at-least-once, best-effort order, near-unlimited throughput — consumers must be idempotent. FIFO SQS = exactly-once (5-min window) + strict order per `MessageGroupId`, at a throughput cost.
- Visibility timeout is the mechanism behind SQS's delivery guarantee — a message reappears if not deleted in time, which is both how reliability works and why duplicates happen on consumer crash/timeout.
- A DLQ (via redrive policy + `maxReceiveCount`) isolates poison messages after repeated failures instead of letting them loop forever and starve the queue.
- SNS fan-out (topic → multiple SQS queues) decouples producers from an evolving set of consumers and lets each consumer have its own durability/retry/DLQ behavior via its dedicated queue.
- SNS filter policies let subscribers receive only relevant messages, avoiding wasted processing on irrelevant ones.

## Common Mistakes

- Writing SQS consumers that assume exactly-once delivery on a **standard** queue, then hitting subtle bugs (double-charging a customer, double-sending an email) the first time a consumer crashes mid-processing and the message is redelivered.
- Not extending the visibility timeout for long-running jobs, causing the same message to be picked up by a second consumer while the first is still working on it — leading to duplicate/concurrent processing of the same unit of work.
- Never configuring a DLQ, so a single malformed message loops indefinitely, consuming worker capacity and burying real errors in a wall of repeated identical failure logs.
- Choosing FIFO by default "to be safe" for a high-throughput use case that doesn't actually need strict ordering, then hitting the throughput ceiling and being confused why messages are backing up compared to standard queues.
- Publishing directly to multiple SQS queues from application code instead of using an SNS fan-out topic — this couples the producer to the full list of consumers and requires a code change every time a new consumer is added.
