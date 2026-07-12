# Day 52 — AWS Messaging Deep Dive: EventBridge & Kinesis

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** AWS | **Flag:** —

## Brief

SNS/SQS (previous file) cover simple pub/sub and point-to-point queuing. EventBridge and Kinesis solve two different, more specialized problems: EventBridge is about **routing structured events by content**, across dozens of AWS services and SaaS partners, without writing glue code; Kinesis is about **ordered, replayable, high-throughput streaming** where multiple independent consumers need to read the same data at their own pace, potentially re-reading history. Knowing which of these four services (SQS, SNS, EventBridge, Kinesis) fits a given problem is a recurring systems-design interview question.

## EventBridge — content-based event routing

EventBridge centers on **event buses** (the default bus, custom buses you create, or partner buses from SaaS vendors like Zendesk/Datadog), **rules** (pattern matches against event content), and **targets** (where matching events get sent — Lambda, SQS, SNS, Step Functions, Kinesis, and 20+ other AWS service targets).

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["stopped", "terminated"]
  }
}
```
```bash
aws events put-rule --name ec2-state-changes --event-pattern file://pattern.json
aws events put-targets --rule ec2-state-changes --targets "Id"="1","Arn"="<LAMBDA_ARN>"
```

**What makes EventBridge different from SNS**, and why this is a common point of confusion: SNS routes based on **topic subscription** (you subscribe to a topic, you get everything published to it, optionally narrowed by a filter policy on message *attributes*). EventBridge routes based on **matching the actual event content/structure** (its `event pattern` can match deeply into the JSON `detail` payload itself, not just top-level attributes), and it natively ingests events from ~200 AWS services (any AWS service action can generate an event on the default bus) plus SaaS partners, without you having to publish anything yourself. This is exactly why GuardDuty findings (Day 49) land on the EventBridge default bus automatically — AWS services push events there natively; you don't wire up that plumbing.

**Scheduled rules** are also EventBridge's replacement for what used to require a standalone cron-like service: `aws events put-rule --schedule-expression "rate(5 minutes)"` or a full cron expression — this is a common way teams trigger periodic Lambda invocations without managing a scheduler themselves (EventBridge Scheduler, a newer, more feature-rich sibling, has since taken over some of this use case with per-invocation retry/DLQ config, but the classic scheduled-rule pattern is still everywhere in existing infrastructure).

**Archive and replay**: EventBridge can archive matched events and **replay** them later against a bus — useful for reprocessing after fixing a bug in a downstream consumer, without needing the original producer to re-emit anything.

## Kinesis Data Streams — ordered, replayable streaming

Kinesis Data Streams is built for a fundamentally different access pattern than SQS: **multiple independent consumers reading the same stream of data, in order, at their own pace, with the ability to replay/re-read recent history**.

Core mechanics:
- A stream is divided into **shards**. Each shard supports up to **1 MB/s or 1,000 records/s ingest**, and **2 MB/s** read (with the classic GetRecords API — Enhanced Fan-Out gives each consumer its own dedicated 2 MB/s instead of sharing it). Throughput scales by adding shards.
- Records are appended with a **partition key** that determines which shard they land on (same partition key → same shard → preserved relative order for that key) — this is conceptually similar to SQS FIFO's `MessageGroupId`, but the ordering guarantee here is a structural property of the shard, not a queue-level feature.
- Data is retained in the stream for a **configurable window (24 hours default, up to 365 days)** — and critically, reading a record via `GetRecords` **does not remove it from the stream** the way SQS's `DeleteMessage` does. Multiple consumers (or the same consumer replaying from an earlier point) can independently read the same data, each tracking their own position via a **sequence number** / checkpoint.

**This is the single biggest conceptual difference from SQS to internalize**: SQS is a queue (a message is claimed, processed, and deleted — one logical consumer "owns" it). Kinesis is a **log** (data stays put; consumers each independently track where they are in reading it, and can rewind). That's why Kinesis fits scenarios like "both a real-time fraud-detection Lambda and a batch analytics job need to process the same clickstream data independently, and the analytics job needs to be able to reprocess the last 3 days after a bug fix" — SQS's claim-and-delete model can't support "read the same message twice from two independent, uncoordinated consumer applications" without significant extra plumbing (fan-out via SNS, essentially reinventing part of what Kinesis gives natively).

**KCL (Kinesis Client Library)** handles the hard parts for you in a real consumer application: shard discovery, checkpointing progress (usually to a DynamoDB table), and rebalancing shard ownership across a fleet of consumer workers as they scale up/down.

## Kinesis Data Streams vs. SQS — the decision in one paragraph

Choose **SQS** when you have a classic work-queue problem: a task needs to be done exactly (or at-least) once by exactly one worker, you don't need multiple independent readers of the same data, and you want the simplest possible operational model. Choose **Kinesis** when you need **ordered, replayable streaming data** consumed independently by multiple applications, especially at high, continuous throughput (clickstreams, IoT telemetry, log aggregation, change-data-capture) — the trade-off is more operational complexity (shard management, checkpointing) in exchange for replay and true multi-consumer fan-out without needing SNS in front of it.

## Points to Remember

- EventBridge routes by matching event **content/structure** (deep into the JSON payload) across ~200 native AWS event sources plus SaaS partners; SNS routes by **topic subscription**, optionally filtered on message attributes only.
- GuardDuty (and most native AWS service events) land on the EventBridge default bus automatically — no producer-side wiring needed, unlike SNS where something must explicitly `Publish`.
- Kinesis is a **log**: reading doesn't delete data, multiple independent consumers can each track their own position, and data can be replayed within the retention window (24h–365 days).
- SQS is a **queue**: a message is claimed, processed, and deleted — built for single-logical-consumer work distribution, not multi-consumer independent replay.
- Kinesis throughput scales by adding shards (1 MB/s in, 2 MB/s out per shard by default); partition key determines shard placement and per-key ordering.

## Common Mistakes

- Trying to use SNS to give two completely independent applications the ability to replay the last few days of historical events — SNS has no retention/replay; that's a signal you actually need Kinesis (or EventBridge Archive/Replay for discrete events, or SNS fanning out into per-consumer SQS queues for durability, though still without true replay).
- Under-provisioning Kinesis shards for the actual ingest rate, causing `ProvisionedThroughputExceededException` throttling — shard count must be sized to peak, not average, ingest rate (or use on-demand capacity mode to avoid manual shard math).
- Assuming Kinesis works like SQS (claim-and-delete) and being confused when the same record shows up again for a different consumer application — that's expected; each consumer tracks its own checkpoint independently.
- Writing an EventBridge rule pattern that's too broad (matching on `source` alone without narrowing `detail-type`/`detail`), causing a Lambda target to be invoked far more often than intended and driving up cost/noise.
- Forgetting that Kinesis retention is time-bounded (default 24 hours) — assuming "replay" means infinite history, and losing the ability to reprocess data older than the configured retention window.
