---
layout: default
title: "Conclusion & Next Steps"
---

Over two iterations, I converged on a **lean RAG stack** that fits a public portfolio: presigned uploads, event-driven ingest, **FAISS on S3**, and a **CloudFront-routed** chat API. The migration away from OpenSearch trimmed idle cost to near-zero and simplified ops. Unifying Terraform across dev/prod (consistent backends, naming, and OIDC role-assumption) removed a lot of “env drift” and made changes predictable.

## Noteworthy achievements
- **Retrieval v2** works: FAISS files in S3, deterministic prompts, and clear `answer + sources` contract.
- **Routing** is clean: one CloudFront distribution, S3 for UI, HTTP API for `/query`.
- **CI/Env hygiene**: inputs are parameterized; we can flip model IDs, K, and token budgets without code changes, as well as using OIDC to secure terraform changes.

## Suggested improvements
1. **Deletion support**
2. **Versioned publishes + atomic `latest/` flip**  
   - Write to `indexes/v00X/{index.faiss,meta.jsonl,manifest.json}` → copy to `indexes/latest/`.  
   - Query Lambda reloads only when `manifest` or object **ETag** changes.
3. **Idempotence everywhere**  
   - Stable `doc_id = hash(s3://bucket/key)`; stable `chunk_id = {doc_id}#{page}#{offset}`.  
4. **List view accuracy**  
   - On delete: update DynamoDB (or your metadata store) and emit a tombstone; list UI hides deleted docs immediately.
5. **Observability**  
   - Metrics: index version, load time, k-NN latency, tokens in/out, 4xx/5xx, retries, downloads from S3.  
   - Alarms on p95 latency and throttles.
