# Day 118 — Lab: Agentic AI Infrastructure

**Goal:** Deploy a vector database on Kubernetes and build a simple RAG pipeline with DevOps observability (latency, cost per query) — the assigned hands-on activity for today, broken into concrete steps.

**Prerequisites:** A Kubernetes cluster (kind/minikube is fine for the mechanics). `helm` and `kubectl`. Python 3.10+ locally with `pip install qdrant-client sentence-transformers openai litellm`. An API key for at least one hosted LLM provider (or a locally-running small model via Ollama/vLLM if you'd rather stay fully offline/free — adjust the LLM call step accordingly).

---

### Lab 1 — Deploy Qdrant on Kubernetes

1. Install Qdrant via Helm:
   ```bash
   helm repo add qdrant https://qdrant.github.io/qdrant-helm
   helm repo update
   helm install qdrant qdrant/qdrant -n rag --create-namespace \
     --set persistence.size=5Gi
   ```
2. Confirm it's running as a StatefulSet with a persistent volume (not a stateless Deployment):
   ```bash
   kubectl get statefulset,pvc -n rag
   ```
3. Port-forward and confirm the API responds:
   ```bash
   kubectl port-forward -n rag svc/qdrant 6333:6333
   curl http://localhost:6333/collections
   ```

**Success criteria:** Qdrant is running as a `StatefulSet` backed by a `PersistentVolumeClaim`, and you can explain in one sentence why that matters versus a stateless Deployment.

---

### Lab 2 — Ingest a small document set

1. Prepare 10-20 short text documents (paragraphs about any topic you like — reuse your own notes from earlier Days in this repo for a fun self-referential test corpus).
2. Embed and upload them:
   ```python
   from qdrant_client import QdrantClient
   from qdrant_client.models import PointStruct, VectorParams, Distance
   from sentence_transformers import SentenceTransformer
   import time

   model = SentenceTransformer("all-MiniLM-L6-v2")   # small, free, local embedding model
   client = QdrantClient(host="localhost", port=6333)

   client.recreate_collection(
       collection_name="notes",
       vectors_config=VectorParams(size=384, distance=Distance.COSINE),
   )

   docs = ["<paste your 10-20 documents here>"]
   embeddings = model.encode(docs)
   client.upsert(
       collection_name="notes",
       points=[PointStruct(id=i, vector=emb.tolist(), payload={"text": doc})
               for i, (doc, emb) in enumerate(zip(docs, embeddings))],
   )
   ```

**Success criteria:** `client.count(collection_name="notes")` returns the expected document count.

---

### Lab 3 — The core hands-on activity: build the RAG query path with timing instrumentation

1. Write a query function that times each stage separately:
   ```python
   import time

   def rag_query(question: str):
       t0 = time.perf_counter()
       q_embedding = model.encode([question])[0]
       t1 = time.perf_counter()

       results = client.search(collection_name="notes", query_vector=q_embedding.tolist(), limit=3)
       t2 = time.perf_counter()

       context = "\n".join(r.payload["text"] for r in results)
       prompt = f"Answer using only this context:\n{context}\n\nQuestion: {question}"

       # Swap in your LLM call of choice (OpenAI/Anthropic/local vLLM via LiteLLM)
       import litellm
       response = litellm.completion(
           model="gpt-4o-mini",   # or route through a local vLLM endpoint
           messages=[{"role": "user", "content": prompt}],
       )
       t3 = time.perf_counter()

       usage = response.usage
       cost = litellm.completion_cost(completion_response=response)

       print(f"embed: {(t1-t0)*1000:.1f}ms | search: {(t2-t1)*1000:.1f}ms | "
             f"llm: {(t3-t2)*1000:.1f}ms | tokens(in/out): {usage.prompt_tokens}/{usage.completion_tokens} | "
             f"cost: ${cost:.5f}")
       return response.choices[0].message.content
   ```
2. Run 5 different questions against your document set and record the per-stage timing + cost breakdown for each.
3. Identify which stage dominates total latency, and which stage dominates total cost — confirm they're not necessarily the same stage.

**Success criteria:** You have a table of 5 queries, each with embed/search/LLM timing broken out separately and a per-query dollar cost, and you can state which stage dominates latency vs. cost.

---

### Lab 4 — Add rate limiting and budget control with LiteLLM

1. Run LiteLLM as a local proxy in front of your chosen provider:
   ```bash
   pip install 'litellm[proxy]'
   cat > litellm-config.yaml <<'EOF'
   model_list:
     - model_name: gpt-4o-mini
       litellm_params: {model: openai/gpt-4o-mini, api_key: os.environ/OPENAI_API_KEY}
   litellm_settings:
     max_budget: 5
     budget_duration: 1d
   EOF
   litellm --config litellm-config.yaml --port 4000
   ```
2. Point your RAG script's `litellm.completion` call at the proxy (`api_base="http://localhost:4000"`) instead of calling the provider directly.
3. Loop your Lab 3 queries enough times to approach the `max_budget` and confirm LiteLLM actually blocks further calls once the budget is exhausted.

**Success criteria:** You've observed LiteLLM enforce a real budget cap by rejecting a request once the configured limit is reached.

---

### Cleanup

```bash
helm uninstall qdrant -n rag
kubectl delete namespace rag
# Stop the local litellm proxy process (Ctrl+C), unset any API keys from your shell history if shared.
```

### Stretch challenge

Add a simple semantic cache in front of the LLM call step: before calling the LLM, check whether a sufficiently similar question (cosine similarity above a threshold, computed the same way as your document search) was already answered recently, and return the cached answer instead of a fresh LLM call. Measure how much cost/latency this saves across a batch of near-duplicate questions versus fully unique ones.
