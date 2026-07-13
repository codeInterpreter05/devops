# Day 118 — Resources: Agentic AI Infrastructure

## Primary (assigned)

- **vLLM documentation** (docs.vllm.ai) — the assigned starting point (alongside your own internship knowledge, per today's brief); covers PagedAttention, quantization options, tensor-parallel serving, and the OpenAI-compatible API server in depth.

## Deepen your understanding

- **"Efficient Memory Management for Large Language Model Serving with PagedAttention"** (the original vLLM paper, arxiv.org) — for the actual mechanism behind PagedAttention, useful to cite precisely in an interview.
- **LiteLLM documentation** (docs.litellm.ai) — proxy/gateway configuration, virtual keys, budgets, rate limits, and routing strategies across providers.
- **Qdrant documentation** (qdrant.tech/documentation) and **Weaviate documentation** (weaviate.io/developers/weaviate) — vector DB concepts (HNSW indexing, hybrid search, collections/schemas) with runnable examples for both.
- **AWS re:Invent talks on EFA and GPU networking for ML** (search AWS's YouTube channel) — for the multi-node distributed training/serving networking piece.
- **OpenTelemetry GenAI semantic conventions** (opentelemetry.io/docs/specs/semconv/gen-ai) — the emerging standard for structuring LLM observability spans (tokens, model, cost) in a vendor-neutral way.

## Reference / lookup

- **vLLM supported models and quantization methods list** (docs.vllm.ai — Models page) — check compatibility before committing to a specific quantization approach (GPTQ/AWQ/etc.) for a given model.
- **NVIDIA GPU memory sizing guides for LLM serving** — useful for back-of-envelope memory calculations (parameters x bytes-per-param + KV cache estimate) before choosing an instance type.

## Practice

- **LangSmith / Langfuse free tiers** — stand up either as a companion to the Day 118 lab's RAG pipeline to see purpose-built LLM/agentic tracing (token counts, cost, multi-step trace trees) rendered in a real UI, rather than just printed timing numbers.
