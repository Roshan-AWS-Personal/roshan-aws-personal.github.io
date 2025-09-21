---
layout: default
title: "Retrieval v1: OpenSearch Serverless"
---

I started the RAG build on **Amazon OpenSearch Serverless (AOSS)** for vector search. It let me focus on good chunking + embeddings first, without managing shards or instances. The end-to-end loop was: **S3 → SQS → Lambda (ingest) → Titan embeddings → AOSS** for write, and **API Gateway → Lambda (query) → Titan (query embed) → AOSS Top-K → Claude** for read.

### **Design (what I built)**
- **Ingest path**
  - **S3 → SQS → Lambda (ingest):** decoupled, retryable pipeline for new/updated docs.
  - **Chunking & embed:** Titan Text Embeddings v2 (1024-d).
  - **Index write:** each chunk stored in AOSS with `{text, vector, doc_id, page, meta…}`.
- **Query path**
  - **API Gateway → Lambda (query):** receive `{question}`.
  - **Embed question:** Titan → `query_vector`.
  - **Retrieve:** AOSS **k-NN Top-K** on the `vector` field.
  - **Assemble prompt:** concise snippets + question → **Claude** via Bedrock.
  - **Return:** grounded answer (+ optional citations).

### **Index mapping (AOSS)**
```json
{
  "settings": { "index.knn": true },
  "mappings": {
    "properties": {
      "vector": { "type": "knn_vector", "dimension": 1024 },
      "text":   { "type": "text" },
      "doc_id": { "type": "keyword" },
      "page":   { "type": "integer" },
      "meta":   { "type": "object", "enabled": true }
    }
  }
}
```

### **Example: k-NN search (Top-K)**
```json
{
  "size": 8,
  "query": {
    "knn": {
      "vector": { "vector": [0.018, -0.004, ...], "k": 8 }
    }
  },
  "_source": ["text", "doc_id", "page"]
}
```

### Operational notes
- **IAM scoping:** Ingest Lambda had **write-only** permissions to the collection; Query Lambda had **search-only**. Collection security policy was kept minimal.
- **SQS durability:** S3 → SQS decoupled spikes; ingest was **idempotent** by using a stable `{doc_id#page#chunk}` as the OpenSearch document ID.
- **Tuning knobs via env:** `K` (Top-K), chunk size/overlap, index name/collection, and max context tokens were **Lambda env vars** so I could tune without redeploys.
- **Timeouts & memory:** Ingest memory sized for batch embeddings; Query balanced latency vs. cost with short timeouts and reserved concurrency kept low.
- **Retries/backoff:** Exponential backoff on AOSS 429/5xx and Bedrock throttles; simple circuit-break to fail fast when upstreams were unhealthy.

### What worked well
- **Retrieval quality:** Once chunk size/overlap were dialed in, **Top-K ≈ 8** returned consistently relevant context for prompts.
- **Developer velocity:** Managed vector search meant **no shard/instance management** and fast iteration using familiar OpenSearch DSL.
- **Operational simplicity:** S3 → SQS → Lambda handled bursts cleanly; idempotent indexing made re-runs safe.
- **Debuggability:** Easy to **inspect stored chunks and scores** in AOSS while tuning embeddings and prompt assembly.

