# Day 118 — Agentic AI Infrastructure: LLM Infra & GPU Node Pools

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** MLOps | **Flag:** ⚡ Interview-critical

## Brief

Everything from Days 116-117 (MLOps fundamentals, model serving, GPU scheduling) was building toward this: LLM and agentic AI workloads are the most GPU-hungry, most operationally novel class of ML infrastructure most DevOps engineers will touch in the next few years, and "design the infra for a production agentic AI system" (today's interview question) is rapidly becoming one of the most common senior DevOps/MLOps interview prompts. This note covers the infrastructure foundation — what's structurally different about LLM workloads and how GPU node pools need to be shaped for them; the next two files cover inference optimization/observability and the vector-DB/RAG/cost-control layer that sits on top.

This day is split into three focused files:

1. **This file** — what's different about LLM infra, and GPU node pools on EKS for LLM workloads.
2. **[02-README-Inference-Optimisation-And-Observability.md](02-README-Inference-Optimisation-And-Observability.md)** — quantization, vLLM, and LLM observability/tracing.
3. **[03-README-Vector-DBs-And-RAG-Pipeline-Cost-Control.md](03-README-Vector-DBs-And-RAG-Pipeline-Cost-Control.md)** — Weaviate/Qdrant, RAG pipeline infra, and rate limiting/cost control.

## What's structurally different about LLM infrastructure

Everything in Day 117 (KServe, GPU scheduling) still applies to LLM serving, but three properties of LLMs specifically break assumptions that hold for smaller, traditional ML models:

1. **Model size dwarfs everything else.** A 70B-parameter model at 16-bit precision needs ~140GB just to hold the weights — before any KV-cache memory for in-flight requests. This routinely exceeds a single GPU's memory (an A100 has 40 or 80GB), forcing **multi-GPU tensor/pipeline parallelism** just to serve one model instance, which is a genuinely different infrastructure problem than "one GPU, one model" from Day 117.
2. **Inference is autoregressive and memory-bound, not just compute-bound.** Generating each output token requires a full forward pass conditioned on all previous tokens, and the **KV cache** (attention keys/values for every previous token, per request) grows linearly with sequence length and concurrent requests — meaning the bottleneck is often GPU memory bandwidth/capacity for cache, not raw FLOPs. This is why naive batching (just queue more requests) doesn't scale the way it does for a small classifier — see the next file's coverage of vLLM's PagedAttention, built specifically to address this.
3. **Latency is inherently variable and request-length-dependent.** A one-word answer and a 2,000-token essay from the same model take wildly different amounts of wall-clock time, which breaks simple fixed-timeout/fixed-concurrency assumptions that work fine for a sub-100ms classification endpoint.

## GPU node pools on EKS, sized for LLMs specifically

The mechanics from Day 117 (device plugin, taints, Karpenter NodePools) still apply, but LLM-specific sizing decisions matter more:

- **Instance selection by memory, not just compute.** For LLM serving, GPU **memory capacity** is usually the binding constraint before compute is — a `p4d.24xlarge` (8x A100 40GB = 320GB total) or `p5.48xlarge` (8x H100 80GB) is chosen because the model + KV cache needs to fit, not because the workload is compute-bound in the traditional sense.
- **Multi-GPU-per-pod scheduling.** Serving a model sharded across multiple GPUs means a single pod requests multiple `nvidia.com/gpu` units on the **same node** (tensor parallelism generally requires GPUs to be on the same node with fast NVLink/NVSwitch interconnects — spreading shards across nodes over standard networking introduces latency that usually defeats the purpose):
  ```yaml
  resources:
    limits:
      nvidia.com/gpu: 4   # e.g., a 4-way tensor-parallel shard of one model
  ```
- **EFA (Elastic Fabric Adapter) networking** for multi-node training/serving setups that do need cross-node communication (large-scale distributed training, or pipeline-parallel serving spanning nodes) — standard TCP/IP networking has too much latency overhead for the frequent all-reduce/collective communication these workloads require.
- **Capacity Reservations / On-Demand Capacity Reservations (ODCRs)** matter far more for LLM GPU instances than for general compute, because high-end GPU instance types (`p5`, `p4d`) frequently have genuine capacity constraints at the AWS region level — it's not uncommon to be unable to launch an on-demand `p5` instance at all in a given AZ during high-demand periods, regardless of budget. Reserving capacity ahead of a known launch is a real operational necessity, not just a cost optimization, for this instance class specifically.

```bash
eksctl create nodegroup --cluster my-cluster --name llm-inference \
  --node-type p4d.24xlarge --nodes-min 0 --nodes-max 3 \
  --node-labels workload=llm-inference \
  --node-taints nvidia.com/gpu=present:NoSchedule
```

## Points to Remember

- LLM inference is memory-bound (KV cache growth) as much as compute-bound — infrastructure sizing decisions should be driven by memory capacity/bandwidth, not just raw GPU FLOPs.
- Serving a large model often requires multiple GPUs **on the same node** for tensor parallelism, because cross-GPU communication needs low-latency interconnects (NVLink/NVSwitch) that only exist within a node.
- High-end GPU instance types (`p4d`, `p5`) have genuine regional capacity constraints — Capacity Reservations for these are often an availability necessity, not merely a cost lever, unlike general-purpose compute.
- Variable, request-length-dependent latency breaks fixed-timeout/fixed-concurrency assumptions carried over from traditional low-latency microservice infra — LLM serving infra needs to be designed around this variability explicitly.
- EFA/high-bandwidth networking is only needed when parallelism spans multiple nodes — single-node multi-GPU tensor parallelism relies on intra-node interconnects instead, a different networking problem entirely.

## Common Mistakes

- Sizing GPU instances based on compute benchmarks alone, without checking whether the target model's weights + expected concurrent KV cache actually fit in the chosen GPU's memory — leads to OOM failures under real concurrent load that never showed up in a single-request smoke test.
- Assuming multi-GPU parallelism works the same regardless of whether GPUs are on the same node or spread across nodes — tensor-parallel sharding across nodes without adequate interconnect (EFA or equivalent) can be dramatically slower than expected, sometimes slower than not parallelizing at all.
- Treating `p4d`/`p5`-class GPU capacity like ordinary EC2 capacity that's "always available if you're willing to pay" — discovering during a real launch that on-demand capacity simply isn't available in the target region/AZ, with no reservation in place.
- Applying the same fixed request-timeout values used for a traditional REST microservice to an LLM endpoint, causing long, legitimate generations to be killed mid-stream.
- Forgetting to taint/isolate expensive GPU node pools from general workloads (same mistake as Day 117, but the cost of getting it wrong scales up dramatically with `p4d`/`p5`-class instance pricing).
