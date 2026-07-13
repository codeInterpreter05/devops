# Day 118 — Agentic AI Infrastructure: Vector DBs, RAG Pipelines & Cost Control

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** MLOps | **Flag:** ⚡ Interview-critical

## Brief

Retrieval-Augmented Generation (RAG) is how most production LLM/agentic systems ground model outputs in real, current, organization-specific data instead of relying purely on the model's frozen training data — and it introduces a whole new stateful infrastructure component (the vector database) plus a whole new cost-control problem (LLM APIs bill per token, with no natural rate limit unless you build one). This file closes out the day by covering both, plus the observability metrics (latency, cost per query) that today's hands-on activity specifically asks you to measure.

## Vector databases on Kubernetes: what they actually do

A vector database stores **embeddings** (dense numeric vectors representing the semantic meaning of text/images/etc., produced by an embedding model) and provides efficient **approximate nearest neighbor (ANN)** search — given a query embedding, quickly find the most semantically similar stored embeddings, at a scale (millions to billions of vectors) where naive brute-force comparison would be far too slow.

**Weaviate** and **Qdrant** are two leading open-source options, both deployable on Kubernetes:

```yaml
# Qdrant on Kubernetes via Helm (StatefulSet-backed, since it's a stateful data store)
# helm repo add qdrant https://qdrant.github.io/qdrant-helm
# helm install qdrant qdrant/qdrant -n rag --create-namespace \
#   --set persistence.size=50Gi --set replicaCount=3
```

**Key operational properties that matter for a DevOps/platform engineer, not just an ML engineer:**
- **They are stateful, persistent data stores** — deploy as a `StatefulSet` with persistent volumes, not a stateless Deployment; back them up like any other production database, because rebuilding an index by re-embedding millions of documents from scratch is slow and costly.
- **Indexing algorithm choice affects both recall and resource usage** — HNSW (Hierarchical Navigable Small World graphs) is the common default ANN algorithm; it trades index build time and memory for fast, high-recall queries, and its parameters (e.g., graph connectivity) are a real tuning knob between search quality and resource cost.
- **Embedding dimensionality directly drives storage/memory cost** — a 1536-dimension embedding (a common OpenAI embedding size) at scale across millions of documents is a genuinely large memory footprint; choosing a smaller embedding model when quality allows is a direct infrastructure cost lever, the vector-DB equivalent of Day 115's pod right-sizing.
- **Hybrid search** (combining vector similarity with traditional keyword/BM25 search) is increasingly standard because pure vector search alone can miss exact-match cases (specific product codes, exact names) that keyword search catches trivially — both Weaviate and Qdrant support this natively.

## RAG pipeline infrastructure

A production RAG pipeline has two distinct infrastructure paths that are easy to conflate:

1. **Ingestion pipeline** (offline/batch, similar shape to Day 117's ML pipelines): documents → chunking → embedding model → write to vector DB. Needs to run on a schedule or event-trigger (new document uploaded) and needs its own observability (embedding failures, chunking edge cases, stale-index detection when source documents change).
2. **Query-time path** (online/real-time, in the request path): user query → embed the query → vector DB similarity search → assemble retrieved context + query into a prompt → LLM call → response. Every step here adds latency, and the vector DB search and the LLM call are typically the two dominant latency contributors, in that rough order for a well-tuned system (vector search should be tens of milliseconds; LLM generation is typically hundreds of milliseconds to seconds depending on output length).

```text
User Query
   |
   v
Embed query (embedding model call - fast, ms-scale)
   |
   v
Vector DB similarity search (top-k retrieval - tens of ms if well-tuned)
   |
   v
Assemble prompt (retrieved chunks + system prompt + query)
   |
   v
LLM call (generation - dominant latency + cost, scales with output tokens)
   |
   v
Response (+ optionally: citations back to retrieved source chunks)
```

**Why "DevOps observability (latency, cost per query)" is the right framing for today's hands-on**: a RAG/agentic system's total latency and cost are the sum of several independently-scaling components, and without breaking them apart in observability, a slow or expensive response gets blamed on "the LLM" by default when it might actually be an unindexed/slow vector search, an oversized top-k retrieval count pulling too much context into the prompt (which then costs more input tokens and adds latency), or a redundant re-embedding of the same query.

## Rate limiting and cost control for LLM APIs

LLM API costs scale with usage in a way that has no natural ceiling unless you build one — a single user (or a single misbehaving agent loop) can generate an unbounded number of expensive calls with no built-in guardrail, unlike a typical internal microservice call that's usually cheap enough per-call not to need this level of deliberate control.

**LiteLLM** is a widely-used tool specifically for this layer — a proxy/gateway that sits in front of many different LLM providers (OpenAI, Anthropic, Bedrock, local vLLM endpoints, etc.) behind one unified OpenAI-compatible API, adding cross-cutting controls the underlying providers don't uniformly offer:

```yaml
# litellm proxy config.yaml
model_list:
  - model_name: gpt-4o
    litellm_params: {model: openai/gpt-4o, api_key: os.environ/OPENAI_API_KEY}
  - model_name: claude-sonnet
    litellm_params: {model: anthropic/claude-sonnet-4-5, api_key: os.environ/ANTHROPIC_API_KEY}
  - model_name: local-llama
    litellm_params: {model: openai/meta-llama/Llama-3-70b, api_base: http://vllm-service:8000/v1}

router_settings:
  routing_strategy: "least-busy"    # or cost-based / latency-based routing across providers

litellm_settings:
  max_budget: 1000          # dollars, org-wide
  budget_duration: 30d
```

```bash
# Per-team/per-key budget and rate limit, via LiteLLM's virtual key API
curl -X POST http://litellm:4000/key/generate -H "Authorization: Bearer $MASTER_KEY" -d '{
  "team_id": "checkout-team",
  "max_budget": 200,
  "budget_duration": "30d",
  "rpm_limit": 60,
  "tpm_limit": 100000
}'
```

**What this actually buys you, operationally:**
- **Per-team/per-key budgets and rate limits** — the LLM-API equivalent of Day 114's cost allocation tags: attributing spend to a team/feature, but enforced at request time, not discovered after the fact in a monthly bill.
- **Provider failover and cost-based routing** — if one provider is down or a specific model's cost spikes, requests can route to an equivalent fallback model automatically.
- **A single point of enforcement** for prompt-injection-adjacent guardrails, logging, and caching (semantic caching of repeated/similar queries can avoid a full LLM call entirely for genuinely duplicate questions) — centralizing these concerns rather than reimplementing them per application team.

## Points to Remember

- Vector databases are stateful, persistent data stores that need the same backup/durability discipline as any production database — not disposable, stateless service replicas.
- Embedding dimensionality and index algorithm (HNSW) parameters are real cost/quality tradeoffs with direct infrastructure cost implications, analogous to right-sizing in earlier days' material.
- RAG query-time latency/cost decomposes into distinct, separately-measurable stages (embed query, vector search, LLM generation) — observability must break these apart, or a bottleneck in one gets misattributed to another.
- LLM API costs have no natural rate ceiling — deliberate rate limiting and budget enforcement (via a gateway like LiteLLM) is required infrastructure, not an optional nice-to-have, especially once autonomous agent loops are in the picture (an agent that can call itself in a loop can burn budget far faster than a human-driven chat UI).
- A gateway layer (LiteLLM or equivalent) decouples application code from any single provider and centralizes cost/rate control, failover, and caching in one place instead of reimplementing them per team.

## Common Mistakes

- Deploying a vector database as a stateless Deployment with no persistent volume or backup strategy, then losing the entire index (and the cost/time of re-embedding everything) after a routine pod restart or node replacement.
- Choosing the largest/highest-dimensionality embedding model by default "for best quality" without evaluating whether a smaller embedding model meets the actual quality bar at a fraction of the storage/memory cost.
- Pulling a large `top-k` of retrieved documents into the prompt "to be safe," inflating input token count (and therefore cost and latency) without measurably improving answer quality — retrieval count is a tunable parameter, not a fixed given.
- Shipping an agentic feature with no per-user/per-team rate limit or budget cap, especially one involving autonomous multi-step agent loops that can retry or re-plan — a single bug in the agent's stopping condition can generate a very large, very fast bill with no automatic circuit breaker.
- Measuring only end-to-end request latency/cost for a RAG system and never breaking it down by stage (embedding, retrieval, generation) — makes it impossible to tell whether an optimization effort should target the vector DB, the retrieval count, or the LLM call itself.
