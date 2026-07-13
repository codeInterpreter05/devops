# Day 118 — Agentic AI Infrastructure: Inference Optimisation & Observability

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** MLOps | **Flag:** ⚡ Interview-critical

## Brief

Given the memory-bound, autoregressive nature of LLM inference covered in the previous file, two things make production LLM serving economically viable at all: inference optimization techniques that shrink memory/compute needs (quantization, vLLM's serving engine), and observability that treats an LLM call as a fundamentally different unit of work than a normal RPC (tracing prompts, tokens, and cost, not just latency and status codes). Skipping either is how a promising LLM feature turns into an unsustainable GPU bill or an undebuggable production incident.

## Quantization: trading precision for memory and speed

Quantization reduces the numeric precision used to store model weights (and sometimes activations), directly shrinking memory footprint and often increasing throughput, at some cost to output quality:

- **FP16/BF16** (16-bit): the common training/serving default, half the memory of FP32.
- **INT8**: roughly half of FP16's memory again, with typically small, workload-dependent accuracy degradation — often acceptable for many applications after validation.
- **INT4 / 4-bit quantization** (via techniques like GPTQ, AWQ, or bitsandbytes' NF4): roughly a quarter of FP16's memory, making it possible to fit models that otherwise wouldn't fit on available GPU memory at all — the tradeoff curve gets steeper here; quality degradation is more workload-dependent and needs actual evaluation, not an assumption that "4-bit is fine."

**Why this matters operationally, not just theoretically**: quantization is frequently the difference between "this 70B model needs 2x A100-80GB and doesn't fit our available node pool" and "this model fits on a single A100-80GB at INT4, unlocking a cheaper/more available instance type." The decision should always be validated against your own evaluation set/business metric, not assumed safe from a generic benchmark — a quantization level that's fine for casual chat completion might visibly degrade a task requiring precise structured output (e.g., code generation, numeric extraction).

## vLLM: the serving engine, not just "a faster runtime"

**vLLM** is a serving engine (comparable in role to KServe's per-model server, but purpose-built for LLM generation) whose headline innovation is **PagedAttention**: it manages the KV cache (the memory that grows per-token, per-request, described in the previous file) using an approach borrowed from OS virtual memory paging — allocating KV cache in non-contiguous fixed-size blocks rather than one large contiguous buffer per request.

**Why this matters mechanically**: naive KV cache allocation reserves a worst-case contiguous memory block per request up front (sized for the maximum possible sequence length), which wastes huge amounts of GPU memory on requests that end up shorter than the worst case — this is exactly the kind of "requested but not used" waste that Day 115's Kubernetes right-sizing concepts apply to, just at the GPU-memory-block level instead of the pod-request level. PagedAttention allocates cache incrementally in small blocks as generation actually proceeds, and can even **share cache blocks across requests** with a common prefix (e.g., the same system prompt reused across many requests) — dramatically improving achievable concurrency per GPU.

```bash
# Serve a model with vLLM's OpenAI-compatible API server
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3-70b-instruct \
  --tensor-parallel-size 4 \
  --quantization awq \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90
```

```bash
# OpenAI-compatible request — existing OpenAI SDK client code works unmodified
curl http://localhost:8000/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "meta-llama/Llama-3-70b-instruct",
  "messages": [{"role": "user", "content": "Summarize this incident report..."}]
}'
```

The **OpenAI-compatible API** matters as a practical infrastructure decision: it means your application code, LiteLLM routing layer (see the next file), and observability tooling can all target a single, well-understood request/response schema regardless of which model or serving engine is actually behind it — swapping vLLM for a different backend later doesn't require touching client code.

## LLM observability: tracing calls, not just requests

A traditional APM trace captures latency, status code, and maybe a few custom spans. LLM observability needs to capture categorically different signals, because the same "endpoint" can behave wildly differently request to request based on prompt content:

- **Token counts** (input/prompt tokens and output/completion tokens, separately) — because cost and latency both scale with tokens, not requests, and the two token counts have very different cost weights (output tokens are typically priced higher than input tokens, and dominate latency since they're generated sequentially one at a time).
- **The full prompt and completion** (or a sampled/redacted version, given sensitive data concerns) — because debugging a bad output requires seeing exactly what was sent, unlike a typical RPC where the request body is rarely the interesting part of the trace.
- **Per-call cost** — computed from token counts × the specific model's pricing, attributed back to the feature/user/tenant that triggered it, extending Day 114-115's cost-allocation concepts into the LLM layer specifically.
- **Multi-step/agentic traces** — for agentic systems, a single user-facing request can fan out into many chained LLM calls (planning step → tool call → follow-up reasoning → final answer), each with its own tokens/cost/latency — the observability system needs to represent this as a tree/DAG of spans, not a flat list, so a slow or expensive user request can be attributed to the *specific* step in the chain that caused it.
- **Retrieval/tool-call outcomes** — for RAG/agentic systems (next file), what documents were retrieved, what tools were invoked and with what arguments/results — because a bad final answer is often actually a bad retrieval or a failed tool call, not a bad generation per se.

Tools purpose-built for this (LangSmith, Langfuse, Arize Phoenix, or generic OpenTelemetry-based tracing with GenAI semantic conventions) structure traces around these signals specifically, rather than forcing an LLM call into a generic APM's request/response model that wasn't designed with tokens/cost/multi-step chains in mind.

## Points to Remember

- Quantization level is a real quality/cost tradeoff that must be validated per use case on your own evaluation data — never assume a quantization level "known to be fine" for one task generalizes to a different task's quality requirements.
- vLLM's PagedAttention solves KV cache memory waste the same way OS virtual memory paging solves fixed-allocation waste — non-contiguous, incrementally allocated blocks instead of worst-case contiguous reservations per request.
- Prefix/prompt cache sharing across requests with a common prefix (e.g., shared system prompts) is one of the biggest concurrency wins available and is largely automatic with a modern serving engine like vLLM — but only if your application actually structures prompts to have a shared, stable prefix.
- LLM observability must capture token counts, per-call cost, full prompt/completion content, and multi-step trace structure — a traditional APM's latency/status-code model is insufficient for debugging or cost-attributing LLM-based systems.
- An OpenAI-compatible serving API is a portability decision, not just a convenience — it decouples client/application code from the specific serving engine or model behind it.

## Common Mistakes

- Applying an aggressive quantization level purely to save memory/cost without evaluating output quality on the actual downstream task — discovering the degradation only after users notice worse answers in production.
- Treating vLLM (or any LLM serving engine) as a drop-in replacement for a general-purpose model server without accounting for its GPU-memory-utilization tuning knobs (`--gpu-memory-utilization`, `--max-model-len`) — under-tuning these leaves throughput on the table; over-tuning risks OOM under real concurrent load.
- Structuring prompts inconsistently (varying the system prompt or prefix ordering per request) in a way that defeats prefix-cache sharing, silently giving up a major concurrency/cost win the serving engine would otherwise provide for free.
- Instrumenting LLM calls with only generic APM latency/error tracking, and having no visibility into token counts or per-call cost — leads to a cost incident being discovered only when the monthly bill arrives, not from any real-time signal.
- For agentic/multi-step systems, only tracing the outermost user-facing request and losing visibility into which specific internal step (a particular tool call, a particular reasoning hop) was slow, expensive, or wrong — makes root-causing a bad agent response far harder than it needs to be.
