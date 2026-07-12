# Day 52 — Resources: AWS Messaging Deep Dive

## Primary (assigned)

- **AWS messaging comparison docs** (aws.amazon.com — "Choosing between messaging services") — the assigned starting point comparing SQS, SNS, EventBridge, Kinesis, and MSK side by side directly from AWS's own guidance.

## Deepen your understanding

- **AWS SQS Developer Guide — Visibility Timeout & Dead-Letter Queues** — the authoritative mechanics behind at-least-once delivery and poison-message isolation, with the exact API sequence.
- **AWS SNS Developer Guide — Fanout scenario** — the canonical S3 → SNS → multiple SQS walkthrough, essentially today's lab straight from the source.
- **Confluent — "Apache Kafka vs. traditional messaging systems"** (free blog/whitepaper) — Kafka's own ecosystem explaining the log-vs-queue distinction in more depth, useful for grounding the MSK comparison in the mental model Kafka's own maintainers use.
- **AWS Kinesis Data Streams Developer Guide — shards, retention, and consumer models (Enhanced Fan-Out vs. classic)** — the mechanics behind why Kinesis scales and behaves differently from SQS.

## Reference / lookup

- `aws sqs help`, `aws sns help`, `aws events help`, `aws kinesis help`, `aws kafka help` — on-box CLI reference for exact parameters.
- **AWS Architecture Blog — messaging pattern posts** (aws.amazon.com/blogs/architecture) — real reference architectures using fan-out, event-driven, and streaming patterns together.

## Practice

- **AWS Free Tier sandbox account** — SQS, SNS, and EventBridge all have generous free tiers; build today's full fan-out pipeline for near-zero cost.
- **Confluent's free Kafka tutorials / Kafka in a Docker container locally** — hands-on with real Kafka semantics (consumer groups, offsets, replay) without needing to provision MSK, useful for grounding the RabbitMQ/SQS/Kafka comparison in direct experience rather than just reading about it.
