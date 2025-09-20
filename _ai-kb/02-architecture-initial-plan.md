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


<div align="center">
  <figure>
    <img src="{{ site.baseurl }}/assets/images/ingest-flow.png" alt="Document Ingestion & Vectorization: S3 → SQS → Lambda (ingest) → Titan Embeddings → OpenSearch Serverless" />
    <figcaption><strong>Figure 1. </strong> Document Ingestion & Vectorization — Client → S3 (docs) → S3 Event → SQS → Lambda (ingest) → Titan Text Embeddings v2 → OpenSearch Serverless (vector index).</figcaption>
  </figure>
</div>

<div align="center">
  <figure>
    <img src="{{ site.baseurl }}/assets/images/query-flow.png" alt="Query & Chat: API Gateway → Lambda (query) → Titan Embeddings (query) → OpenSearch vector search → Claude 3 (chat) → response" />
    <figcaption><strong>Figure 2. </strong> Query & Chat Flow — Client → API Gateway → Lambda (query) → Titan embed(q) → OpenSearch Top-K → Claude 3 with {q+ctx} → response to client.</figcaption>
  </figure>
</div>
