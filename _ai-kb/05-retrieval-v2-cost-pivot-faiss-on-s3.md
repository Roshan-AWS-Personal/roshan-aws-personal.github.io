---
layout: default
title: "Retrieval v2: Cost Pivot: FAISS on S3"
---

Although my first iteration of the project achieved the milestones I wanted, the **idle baseline cost** of OpenSearch Serverless didn’t fit a public portfolio app with **sporadic traffic**. Merely keeping the indexes open in OpenSearch costed me 250 USD a week, which I sadly found out the hard way. I still wanted strong retrieval, but with **near-zero idle** and simpler ops. That led to a deliberate pivot: keep the same chat experience, but swap the vector backend for **FAISS stored in S3** and loaded by Lambda on demand.

### What is FAISS?
FAISS (Facebook AI Similarity Search) is an open-source library for fast vector similarity search—i.e., finding the most similar embeddings to a query vector among millions or billions of vectors. It’s written in C++ with Python bindings, supports CPU and GPU, and provides multiple index types that trade accuracy, speed, and memory.

### Trade-offs from v1 → why v2
- **Idle spend vs. usage:** AOSS stays “on” even when nobody’s querying; FAISS + S3 costs almost nothing at rest.
- **Ops surface:** Managed vector search is convenient, but S3 + FAISS is simpler to own for low-traffic demos.
- **Rebuild tolerance:** My use case can rebuild indices on upload batches; I don’t need real-time incremental writes.

### What changed (and what didn’t)
- **Changed:** Vector store is now **FAISS files in S3**; Lambdas load/search locally.
- **Stayed the same:** **Upload → Ingest → Chat** UI contract; **Titan** for embeddings; **Claude** for answers; event-driven ingestion via **S3 → SQS → Lambda**.
- **Manifest:** tiny JSON with `dimension`, `count`, `created_at`, and an `etag` so the **Query Lambda** can decide whether to refresh `/tmp` cache.

### v2 diagrams
<div align="center">
  <figure class="figure-center">
    <img src="{{ '/assets/images/updated-ingest-flow.png' | relative_url }}" alt="Retrieval v2 Ingest flow: S3 ObjectCreated triggers Lambda container (ingest) which calls Bedrock Titan for embeddings and writes FAISS files to S3 indexes/latest" />
    <figcaption><strong>Figure 3. </strong>Retrieval v2 — Ingest flow (Lambda containers + FAISS on S3)</figcaption>
  </figure>
</div>

<div align="center">
  <figure class="figure-center">
    <img src="{{ '/assets/images/updated-query-flow.png' | relative_url }}" alt="Retrieval v2 Query flow: Lambda container (query) which reads FAISS files from S3 indexes/latest, embeds the question via Titan, and calls Claude Haiku; response returns JSON {answer, sources}" />
    <figcaption><strong>Figure 4. </strong>Retrieval v2 — Query flow (CloudFront → HTTP API → Lambda container → FAISS on S3)</figcaption>
  </figure>
</div>


### Ingestion path (build & publish index)
- **Trigger:** S3 upload → SQS → **Ingest Lambda**.
- **Work:** parse → chunk → embed with **Titan Text Embeddings v2 (1024-d)** → build a FAISS index in `/tmp` → upload to S3 (new version) → copy to `latest/`.

```python
# ingest_lambda.py (trimmed)
import os, json, boto3, faiss, numpy as np
from datetime import datetime

s3 = boto3.client("s3")
BUCKET   = os.environ["INDEX_BUCKET"]
VERSION  = os.environ.get("INDEX_VERSION_PREFIX", "v001")   # e.g., v003 when publishing
LATEST   = "latest"
TMP_DIR  = "/tmp"

def build_and_publish(chunks, embeddings):
    # build FAISS index in /tmp
    dim = len(embeddings[0])
    index = faiss.IndexFlatIP(dim)
    vecs  = np.array(embeddings).astype("float32")
    faiss.normalize_L2(vecs)                  # inner product with normalized vectors ≈ cosine
    index.add(vecs)

    faiss_path = f"{TMP_DIR}/index.faiss"
    faiss.write_index(index, faiss_path)

    meta_path = f"{TMP_DIR}/meta.jsonl"
    with open(meta_path, "w") as f:
        for c in chunks:
            f.write(json.dumps(c) + "\n")

    manifest = {
        "dimension": dim,
        "count": int(index.ntotal),
        "created_at": datetime.utcnow().isoformat() + "Z"
    }
    manifest_path = f"{TMP_DIR}/manifest.json"
    with open(manifest_path, "w") as f:
        json.dump(manifest, f)

    # upload versioned
    for key, path in {
        f"indexes/{VERSION}/index.faiss": faiss_path,
        f"indexes/{VERSION}/meta.jsonl": meta_path,
        f"indexes/{VERSION}/manifest.json": manifest_path,
    }.items():
        s3.upload_file(path, BUCKET, key)

    # flip "latest" (copy objects)
    for name in ["index.faiss", "meta.jsonl", "manifest.json"]:
        s3.copy_object(
            Bucket=BUCKET,
            CopySource={"Bucket": BUCKET, "Key": f"indexes/{VERSION}/{name}"},
            Key=f"indexes/{LATEST}/{name}",
        )
```

### Query path
- **POST /chat** → **API Gateway** → **Lambda (query)** receives `{question}`.
- **Warm cache check:** If `/tmp/index.faiss` or `/tmp/manifest.stamp` is missing/stale, **download** `indexes/latest/` from S3.
- **Embed question:** **Amazon Bedrock – Titan Text Embeddings v2** → `query_vector` (L2-normalized).
- **Retrieve context:** **FAISS** Top-K search in-memory (e.g., `K=6–8`) → get ids/scores, read snippets from `meta.jsonl`.
- **Assemble prompt:** concise, deduped snippets + the user question; enforce a **token budget**.
- **Answer:** call **Claude 3 Haiku** with `{question + context}`; return text (+ optional citations) to the client.

### query_lambda.py (trimmed)

```python
import os, json, boto3, faiss, numpy as np

s3 = boto3.client("s3")
BUCKET  = os.environ["INDEX_BUCKET"]
TMP_DIR = "/tmp"

def load_index_if_stale():
    # compare a small stamp in /tmp with S3 manifest (etag or created_at)
    s3.download_file(BUCKET, "indexes/latest/manifest.json", f"{TMP_DIR}/manifest.json")
    new = json.load(open(f"{TMP_DIR}/manifest.json"))
    stamp_path = f"{TMP_DIR}/manifest.stamp"
    old = json.load(open(stamp_path)) if os.path.exists(stamp_path) else {}

    if old.get("created_at") != new.get("created_at"):
        # pull fresh index + meta
        s3.download_file(BUCKET, "indexes/latest/index.faiss", f"{TMP_DIR}/index.faiss")
        s3.download_file(BUCKET, "indexes/latest/meta.jsonl", f"{TMP_DIR}/meta.jsonl")
        with open(stamp_path, "w") as f: json.dump(new, f)

    return faiss.read_index(f"{TMP_DIR}/index.faiss")

def knn(query_vec, k=8):
    index = load_index_if_stale()
    q = np.array([query_vec]).astype("float32")
    faiss.normalize_L2(q)
    scores, ids = index.search(q, k)   # inner product on normalized vectors
    return ids[0], scores[0]
```

### Performance & cost notes
- **Near-zero idle:** Pay only for **Lambda invocations**, **S3 storage/GET**, and **Bedrock tokens**; no always-on vector service.
- **Cold start behavior:** First query after deploy/index-flip downloads the index to `/tmp`; subsequent queries are **hot**.
- **Memory sizing:** Allocate enough for `index.faiss` + working set (e.g., **512–2048 MB** depending on index size).
- **S3 transfer impact:** Keep the index compact (float32 vs. float16 trade-off), and **reuse `/tmp`** to avoid repeated downloads.
- **Throughput knobs:** Tune `K`, snippet length, and model `max_tokens` to control latency and cost.
- **Concurrency control:** Limit provisioned/concurrent executions to avoid thundering-herd re-downloads.

### Operational notes
- **Config via env (no redeploy):** `INDEX_BUCKET`, `INDEX_PREFIX=indexes/latest/`, `TOP_K`, `EMBED_DIM=1024`, `MODEL_ID`, `MAX_TOKENS`.
- **Cache policy:** Store `index.faiss` + `meta.jsonl` in **`/tmp`**; refresh only when `manifest` changes.
- **Observability:** Log `k`, `hit_count`, `latency_ms`, `downloaded_bytes`, `index_version`, `tokens_in/out`; add p95/error-rate alarms.
- **Retries/backoff:** Exponential backoff for S3 GET and Bedrock throttles; fail fast on repeated 5xx with clear error messages.

### Key points
- **Same UX, cheaper core:** Upload → Chat remains unchanged; retrieval is now **FAISS on S3**.
- **Cost-efficient for demos:** **Idle ≈ $0**; costs scale with actual usage.

---
