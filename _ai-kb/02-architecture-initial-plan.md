---
layout: default
title: "Initial Architecture & End-to-End Vision (v1: OpenSearch Serverless)"
---

This section lays out the first design for the chat experience: **upload → ingest → vector search → chat**, built on Amazon Bedrock, OpenSearch Serverless (AOSS), and Lambda.

> **What is RAG (in 2 lines)?**  
> Retrieval-Augmented Generation retrieves relevant context from your own data (via embeddings + vector search) **before** sending a prompt to the LLM. It grounds answers, cuts hallucinations, and reduces cost by keeping prompts concise.

## High-Level Flow (happy path)
1. **User uploads a document** (UI) → lands in **S3** under `docs/`.
2. **S3 Event → SQS** (buffer) to absorb spikes and guarantee delivery.
3. **Ingest Lambda** reads from SQS → chunks text → **Titan Text Embeddings v2 (1024-d)** → **index into OpenSearch Serverless** (AOSS).
4. **User opens Chat page** and asks a question.
5. **Query Lambda**: embed the user query (Titan) → vector search Top-K in AOSS → assemble concise context snippets.
6. **Bedrock Chat (Claude)** receives `{question + context}` → returns a grounded answer.
7. **Response** shown in UI (with optional citations/snippets).
8. **Observability**: CloudWatch logs/metrics for both Lambdas; alarms on errors/throttles. (Optional) DynamoDB row per upload for audit; SES alerts for failures.


<!-- Side-by-side diagrams -->
<style>
  .img-row{display:flex;gap:12px;align-items:flex-start;flex-wrap:wrap;margin:1rem 0}
  .img-row figure{flex:1 1 420px;margin:0}
  .img-row img{width:100%;height:auto;border:1px solid #e2e8f0;border-radius:8px}
  .img-row figcaption{font-size:.9rem;color:#6b7280;margin-top:.5rem}
</style>

<div class="img-row">
  <figure>
    <img src="{{ '/assets/images/ingest-flow.png' | relative_url }}"
         alt="Document ingestion & vectorization: S3 → SQS → Lambda → Titan → OpenSearch Serverless">
    <figcaption>Document Ingestion & Vectorization (S3 → SQS → Lambda → Titan → AOSS)</figcaption>
  </figure>
  <figure>
    <img src="{{ '/assets/images/query-flow.png' | relative_url }}"
         alt="Query & chat: API Gateway → Lambda → Titan → OpenSearch Serverless → Claude">
    <figcaption>Query & Chat (API GW → Lambda → Titan → AOSS → Claude)</figcaption>
  </figure>
</div>