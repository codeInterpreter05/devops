# Day 34 — AWS Compute: EKS, ECS, Lambda: Lambda Internals & EventBridge

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** —

## Brief

Lambda is the compute option where you genuinely don't manage any server, node, or cluster — but "serverless" doesn't mean "no operational characteristics to understand." Cold starts, concurrency limits, and layers are the specific mechanics that determine whether Lambda is a good fit for a given workload, and they come up constantly in interviews because they're where the "serverless is magic" mental model breaks down in practice. This file also covers EventBridge, the event bus that's usually what actually triggers a Lambda function in a real event-driven architecture — directly relevant to today's hands-on activity (a Lambda triggered by SQS, compared against a Kubernetes-based consumer).

## Cold starts — what actually happens on invocation

When a Lambda function is invoked and there's no already-initialized execution environment available, AWS must: download your deployment package/image, start a new micro-VM (Firecracker, the same lightweight virtualization technology underpinning Fargate), initialize the language runtime, and run your handler's top-level/init code (anything outside the handler function — SDK client construction, DB connection setup) — **before** your actual handler logic starts executing. This entire sequence is the **cold start**, and it can add anywhere from tens of milliseconds (a small Node.js function) to several seconds (a large Java/.NET function with heavy dependency initialization, or a container-image-packaged function).

```python
import boto3
# This runs ONCE per cold start, reused across warm invocations of the same environment
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("my-table")

def handler(event, context):
    # This runs on EVERY invocation, warm or cold
    return table.get_item(Key={"id": event["id"]})
```

This is why the standard cold-start mitigation is **moving expensive setup (SDK clients, DB connections, config loads) to the top level, outside the handler** — a warm invocation reuses the same execution environment (and thus the same already-initialized `dynamodb`/`table` object) for some period after the previous invocation finishes, avoiding that setup cost on every single call. AWS doesn't guarantee how long an environment stays warm — it depends on invocation frequency and internal capacity management, so you should treat warm reuse as an optimization, never a guarantee your code depends on for correctness.

## Provisioned concurrency — paying to eliminate cold starts

```bash
aws lambda put-provisioned-concurrency-config \
  --function-name my-function --qualifier prod \
  --provisioned-concurrency-config AllocatedConcurrentExecutions=10
```

**Provisioned concurrency** pre-initializes a specified number of execution environments and keeps them warm continuously, regardless of traffic — an invocation routed to one of these environments skips the cold-start sequence entirely, at the cost of paying for that capacity to sit ready **even when idle** (unlike normal on-demand Lambda billing, which only charges for actual invocation time). This is the correct tool specifically for **latency-sensitive, user-facing endpoints** (a synchronous API Gateway-backed Lambda where p99 latency matters) — it is *not* generally worth paying for on asynchronous, latency-insensitive workloads like an SQS-triggered batch processor, where a few hundred extra milliseconds on an occasional cold invocation is invisible to any real user.

**Auto-scaling of provisioned concurrency** exists too — you can attach an Application Auto Scaling policy so provisioned concurrency itself scales up ahead of a predictable traffic pattern (e.g., scheduled scale-up before a known daily peak) rather than being a fixed number.

## Lambda layers — sharing code/dependencies across functions

```bash
aws lambda publish-layer-version --layer-name common-utils \
  --zip-file fileb://layer.zip --compatible-runtimes python3.12
```

```python
# in your function config:
"Layers": ["arn:aws:lambda:ap-south-1:123456789012:layer:common-utils:3"]
```

A **layer** is a versioned ZIP of shared code/libraries (a common internal utility module, a large dependency like `pandas` or `numpy`, or a shared set of DB-connection helpers) that gets mounted into a function's execution environment at `/opt` — multiple functions can reference the same layer version without each function's own deployment package needing to bundle a private copy. This keeps individual function deployment packages small (faster to upload/cold-start) and centralizes a shared dependency's version in one place — bump the layer version, and every function referencing it (once redeployed to point at the new version) gets the update, rather than needing to manually re-vendor the dependency into N separate function packages.

**Caveat worth knowing**: layers count toward Lambda's overall deployment package size limits (250MB unzipped, across function code + all attached layers combined) — layers reduce duplication, they don't grant unlimited extra size headroom.

## EventBridge — the event bus that actually triggers most real-world Lambdas

```bash
aws events put-rule --name daily-report --schedule-expression "cron(0 6 * * ? *)"
aws events put-targets --rule daily-report --targets "Id"="1","Arn"="arn:aws:lambda:...:function:generate-report"
```

**EventBridge** is AWS's serverless event bus — it routes events from AWS services, SaaS partners, or your own applications to targets (Lambda, Step Functions, SQS, and more) based on **rules** that match event patterns, without the producer and consumer needing to know about each other directly. This is the backbone of genuinely event-driven architectures on AWS: an S3 upload, an ECS task state change, a scheduled cron, or a custom application event ("OrderPlaced") can all be EventBridge sources, decoupling "something happened" from "here's what should react to it."

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": { "bucket": { "name": ["my-uploads-bucket"] } }
}
```

**Contrast with SQS as a trigger** (today's hands-on activity): SQS is a point-to-point queue — a message goes to one queue, and Lambda's SQS event-source mapping polls it and invokes your function with a batch of messages, with built-in retry/DLQ semantics per message. EventBridge is pub/sub-style routing based on content pattern matching — better suited when **multiple, potentially unrelated consumers** need to react differently to the same class of event, or when the source is something other than your own application putting messages on a queue you control (e.g., reacting to native AWS service state changes). Many real architectures use both together: EventBridge routes a matched event into an SQS queue (for buffering/retry semantics), which a Lambda then consumes.

## Points to Remember

- A cold start includes environment provisioning, runtime init, and your handler file's top-level code — move expensive setup (SDK clients, DB connections) outside the handler function to benefit from warm reuse, but never assume warm reuse is guaranteed.
- Provisioned concurrency eliminates cold starts by keeping environments pre-warmed continuously, at a cost even when idle — worth it for latency-sensitive synchronous endpoints, rarely worth it for async/batch workloads.
- Layers share code/dependencies across multiple functions via a mounted `/opt` directory, keeping individual deployment packages smaller — but still count toward the combined 250MB unzipped size limit.
- EventBridge is pattern-matching pub/sub event routing across many possible sources/targets; SQS is a simpler point-to-point queue with per-message retry/DLQ semantics — many real systems combine both.
- "Serverless" doesn't mean "no capacity planning" — cold starts, concurrency limits, and layer size limits are real operational characteristics that shape whether Lambda fits a given workload.

## Common Mistakes

- Initializing SDK clients/DB connections inside the handler function on every invocation instead of at the top level, paying the initialization cost repeatedly even on warm invocations that didn't need it.
- Turning on provisioned concurrency for an asynchronous batch-processing Lambda where cold-start latency is invisible to any user, paying continuously for idle warm capacity with no real benefit.
- Assuming a warm execution environment is guaranteed to persist between invocations, and writing code that depends on in-memory state surviving across calls (it can vanish at any time when AWS reclaims an idle environment) instead of treating any request-scoped assumption as unsafe.
- Bundling large dependencies directly into every function's deployment package instead of factoring them into a shared layer, bloating deployment size and slowing every cold start unnecessarily.
- Choosing EventBridge for a simple one-producer-one-consumer queueing need (adding pattern-matching-rule complexity for no benefit) or, conversely, choosing plain SQS when multiple unrelated services actually need to react differently to the same event — picking the wrong tool for the actual fan-out shape of the problem.
