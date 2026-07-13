# Day 118 — Cheatsheet: Agentic AI Infrastructure

## GPU node pool for LLM workloads (EKS)

```bash
eksctl create nodegroup --cluster my-cluster --name llm-inference \
  --node-type p4d.24xlarge --nodes-min 0 --nodes-max 3 \
  --node-labels workload=llm-inference \
  --node-taints nvidia.com/gpu=present:NoSchedule
```

```yaml
resources:
  limits:
    nvidia.com/gpu: 4       # multi-GPU tensor-parallel shard, same node
```

## vLLM

```bash
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3-70b-instruct \
  --tensor-parallel-size 4 \
  --quantization awq \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90
```

```bash
curl http://localhost:8000/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "meta-llama/Llama-3-70b-instruct",
  "messages": [{"role": "user", "content": "hello"}]
}'
```

## Quantization quick reference

```
FP32  -> baseline, rarely used for serving
FP16/BF16 -> half memory of FP32, common serving default
INT8      -> ~half of FP16, usually small accuracy loss
INT4 (GPTQ/AWQ/NF4) -> ~quarter of FP16, validate quality per use case
```

## Qdrant (vector DB)

```bash
helm repo add qdrant https://qdrant.github.io/qdrant-helm
helm install qdrant qdrant/qdrant -n rag --create-namespace --set persistence.size=20Gi
```

```python
from qdrant_client import QdrantClient
from qdrant_client.models import PointStruct, VectorParams, Distance

client = QdrantClient(host="localhost", port=6333)
client.recreate_collection("notes", vectors_config=VectorParams(size=384, distance=Distance.COSINE))
client.upsert("notes", points=[PointStruct(id=1, vector=[...], payload={"text": "..."})])
results = client.search(collection_name="notes", query_vector=[...], limit=3)
```

## Weaviate (vector DB, alternative)

```bash
helm repo add weaviate https://weaviate.github.io/weaviate-helm
helm install weaviate weaviate/weaviate -n rag --create-namespace
```

```python
import weaviate
client = weaviate.connect_to_local()
collection = client.collections.get("Notes")
results = collection.query.near_vector(near_vector=[...], limit=3)
```

## RAG query-path stages (for observability instrumentation)

```
1. Embed query        -> ms-scale, local/small embedding model call
2. Vector DB search    -> tens of ms if well-tuned (HNSW index)
3. Assemble prompt      -> negligible, but top-k size drives token cost here
4. LLM generation        -> dominant latency + cost, scales with OUTPUT tokens
5. Response (+citations)
```

## LiteLLM (LLM gateway / cost control)

```yaml
# config.yaml
model_list:
  - model_name: gpt-4o
    litellm_params: {model: openai/gpt-4o, api_key: os.environ/OPENAI_API_KEY}
  - model_name: local-llama
    litellm_params: {model: openai/meta-llama/Llama-3-70b, api_base: http://vllm-service:8000/v1}
litellm_settings:
  max_budget: 1000
  budget_duration: 30d
router_settings:
  routing_strategy: "least-busy"
```

```bash
litellm --config config.yaml --port 4000

# Create a per-team virtual key with budget + rate limits
curl -X POST http://localhost:4000/key/generate -H "Authorization: Bearer $MASTER_KEY" -d '{
  "team_id": "checkout-team", "max_budget": 200, "budget_duration": "30d",
  "rpm_limit": 60, "tpm_limit": 100000
}'
```

```python
import litellm
response = litellm.completion(model="gpt-4o-mini", messages=[...])
cost = litellm.completion_cost(completion_response=response)
usage = response.usage   # prompt_tokens, completion_tokens
```

## LLM observability signals to capture per call

```
prompt_tokens, completion_tokens     (separately — different cost weight)
per_call_cost                        (tokens x model price)
latency_ms per stage (embed/search/generate)
full prompt + completion (or redacted/sampled)
trace tree for multi-step agent calls (not flat spans)
retrieved doc IDs / tool call args+results (for RAG/agentic debugging)
```
