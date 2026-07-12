# Day 27 — Autoscaling: KEDA (Event-Driven Autoscaling)

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

CPU/memory utilization is a poor scaling signal for a huge class of real workloads: a queue consumer can sit at near-zero CPU while a backlog of 50,000 unprocessed messages builds up, because it's I/O-bound, not CPU-bound — HPA on CPU alone would never scale it up. **KEDA (Kubernetes Event-Driven Autoscaling)** exists specifically to let you scale on the metric that actually reflects demand — queue depth, stream lag, request rate from an external system — for dozens of event sources, without hand-building a custom metrics adapter for each one. This is directly the assigned interview question ("how would you auto-scale pods based on SQS queue depth"), so understanding KEDA's mechanism precisely (not just "it scales on queues") matters.

## The core mechanism: KEDA wraps HPA, it doesn't replace it

KEDA does **not** reimplement autoscaling from scratch — under the hood, when you create a `ScaledObject`, KEDA's operator automatically creates and manages a standard Kubernetes **HPA** on your behalf, with KEDA acting as a **custom/external metrics API server** feeding that HPA the values it needs. This is the single most important architectural fact about KEDA: it's a metrics adapter plus a control layer around vanilla HPA, not a competing autoscaling engine.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-consumer-scaler
spec:
  scaleTargetRef:
    name: order-consumer          # the Deployment to scale
  minReplicaCount: 0               # KEDA's superpower over plain HPA: can scale to ZERO
  maxReplicaCount: 30
  pollingInterval: 15               # how often (seconds) KEDA checks the external metric
  cooldownPeriod: 300               # seconds of low activity before scaling back to minReplicaCount
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789012/orders
        queueLength: "5"            # target: ~5 messages per replica
        awsRegion: us-east-1
      authenticationRef:
        name: keda-aws-credentials   # references a TriggerAuthentication (commonly backed by IRSA)
```

```bash
kubectl get scaledobject order-consumer-scaler
kubectl describe scaledobject order-consumer-scaler
kubectl get hpa                       # KEDA created one automatically — named keda-hpa-<scaledobject-name>
```

## Why KEDA over hand-rolling a custom metrics adapter

- **Scale-to-zero**: plain HPA's `minReplicas` cannot go below 1. KEDA can scale a Deployment all the way to **0 replicas** when there's genuinely no work (e.g., an empty queue), then scale back up the moment work appears — a real cost saver for bursty, intermittent workloads, something HPA alone structurally cannot do.
- **Huge library of built-in scalers**: 60+ pre-built triggers (SQS, Kafka, RabbitMQ, Prometheus, Azure Service Bus, cron schedules, PostgreSQL row counts, and more) — each one is a small, well-tested adapter, versus writing and maintaining your own Prometheus Adapter query mapping per metric source.
- **Authentication is a first-class concept** (`TriggerAuthentication`/`ClusterTriggerAuthentication`), commonly wired to IRSA on EKS so the KEDA operator (or the scaler itself) can securely read queue depth without static credentials — directly reusing Day 26's IRSA pattern.

## RabbitMQ example (matches this day's assigned lab)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: rabbitmq-consumer-scaler }
spec:
  scaleTargetRef: { name: rabbitmq-consumer }
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: rabbitmq
      metadata:
        queueName: work-queue
        mode: QueueLength
        value: "20"                  # target ~20 messages per replica
        host: amqp://guest:guest@rabbitmq.default.svc.cluster.local:5672
```

With `value: "20"` and a queue depth of 200, KEDA/HPA computes a target of `200/20 = 10` replicas (capped by `maxReplicaCount`). As consumers drain the queue and depth drops, replicas scale back down — down to `minReplicaCount` (or all the way to 0 if configured and the queue empties, given a cooldown period).

## HPA vs KEDA-managed HPA: the practical tradeoffs

| | Plain HPA (CPU/memory) | KEDA (`ScaledObject`) |
|---|---|---|
| Signal | Resource utilization | Any of 60+ event sources (queue depth, lag, cron, custom Prometheus query) |
| Minimum replicas | 1 (cannot scale to zero) | 0 (true scale-to-zero) |
| Setup for external metrics | Manual custom/external metrics adapter (e.g., Prometheus Adapter config) | Built-in scaler, declarative YAML |
| Good fit for | CPU/memory-bound, steadily-loaded services | I/O-bound, bursty, queue/event-driven workloads |
| What actually scales | Directly manages HPA object | Creates and manages an HPA object under the hood |

## Points to Remember

- KEDA is not a replacement for HPA — it's a metrics-adapter-plus-controller that creates and manages a real HPA object for you, translating an external event source into a metric HPA understands.
- KEDA's standout capability over plain HPA is **scale-to-zero** (`minReplicaCount: 0`), impossible with vanilla HPA, ideal for genuinely idle, bursty workloads.
- `pollingInterval` controls how often KEDA checks the external metric; `cooldownPeriod` controls how long low/no activity must persist before scaling back down to the minimum — both need tuning to the real arrival pattern of the workload to avoid flapping or sluggish response.
- Authentication to the external system (SQS, RabbitMQ, etc.) is handled via `TriggerAuthentication`, which should be wired to short-lived credentials (IRSA on EKS) rather than static secrets, consistent with Day 26's security guidance.
- Choose CPU/memory HPA for steady, resource-bound workloads; choose KEDA when the real demand signal is external and not reflected in CPU/memory at all (a classic mismatch: an I/O-bound consumer sitting at 5% CPU while its queue backs up for hours).

## Common Mistakes

- Trying to scale a queue consumer using CPU-based HPA because "that's what autoscaling means in Kubernetes," then being confused why it never scales up despite a massive backlog — the workload is I/O-bound, and CPU utilization simply isn't correlated with actual demand.
- Setting `minReplicaCount: 0` for a workload with meaningful cold-start latency (e.g., a JVM app with a slow boot) without accounting for the delay between "first message arrives" and "a pod is up and ready to process it" — scale-to-zero trades cost savings for a real latency hit on the first request/message after idle.
- Forgetting that KEDA creates its own HPA object behind the scenes and then also manually creating a separate HPA for the same Deployment — the two will conflict over who owns the replica count.
- Setting `cooldownPeriod` too short for a bursty, spiky arrival pattern, causing constant scale-down-then-immediate-scale-up "flapping" that wastes resources on pod churn — needs to be tuned against real traffic patterns, not left at a default that doesn't match your workload's actual burstiness.
- Not securing the `TriggerAuthentication` properly (e.g., embedding a static RabbitMQ/SQS credential directly instead of using IRSA or a properly scoped Secret) — repeating the exact static-credential anti-pattern Day 26 covers, just for the autoscaler's own metric collection instead of the application itself.
