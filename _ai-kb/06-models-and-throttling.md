---
layout: default
title: "Models & Throttling (Sonnet → Haiku)"
---

I began chat inference on **Claude 3 Sonnet** for quality, but demo-style traffic quickly exposed **rate limits** and **token cost** as the dominant constraints when using the default free tier in AWS. As a result, I made the choice to use **Claude 3 Haiku** the default with light **throttling/backoff**. For our use case, Haiku allowed for double the requests and its responses were accurate enough to justify moving away from Sonnet.

### What the Query Lambda actually does
- **Embedding & similarity**
  - Embeds with **Titan Embed v2**: `{"inputText": text}` → `embedding`.
  - **L2-normalize** every vector → cosine-friendly search (`v /= ||v||`).
  - FAISS index is loaded from **S3** (`indexes/latest/index.faiss` + `meta.jsonl`) and **cached by ETag** to avoid reloading in warm containers.
- **Retrieval**
  - Search top-k (`TOP_K`, default **8**), then **group hits by doc** and keep up to **3 best chunks per doc**; return at most `MAX_DOCS` (**3**) documents.
  - Optional **filters** supported: `doc_id` (allow-list), `tags` (intersection), `mime` (allow-list).
- **Prompt/message shape**
  - **Strict system** instruction + **single user message** that concatenates **Question** and **EXCERPTS** (ranked, truncated).  
  - Deterministic decoding: `temperature=0`, `top_p=0.0`, `max_tokens=MAX_TOKENS` (**400**).
- **Throttling/backoff**
  - All Bedrock calls use **exponential backoff with jitter** (`tries=5`, base **0.4s**, cap **4s**).

### Prompt & message shape (exactly as sent)
We keep the system instruction strict and pass both the **question** and the **retrieved EXCERPTS** in a single user message. This mirrors the Bedrock Anthropic schema used in the Lambda.

```text
SYSTEM:
You answer ONLY using the EXCERPTS provided.
If the user asks about overall contents, summarize the EXCERPTS.
If the answer is not present, say you don't know.

USER:
Question:
{{question}}

EXCERPTS:
[{{title_1}}] (s3://.../doc1)
- {{snippet_1a}}
- {{snippet_1b}}
- {{snippet_1c}}

[{{title_2}}] (s3://.../doc2)
- {{snippet_2a}}
- {{snippet_2b}}
```

### Python Code Snippet
```python
def _embed(text: str) -> np.ndarray:
    out = _invoke_json_with_retry(EMBED_MODEL_ID, {"inputText": text})
    emb = out.get("embedding")
    if not isinstance(emb, list) or not emb:
        raise RuntimeError("Bad embedding response (missing 'embedding').")
    v = np.array(emb, dtype=np.float32)
    v /= (np.linalg.norm(v) or 1.0)   # cosine-friendly
    return v.astype(np.float32, copy=False)[None, :]

def _chat(question: str, context: str) -> str:
    payload = {
        "anthropic_version": "bedrock-2023-05-31",
        "system": (
            "You answer ONLY using the EXCERPTS provided. "
            "If the user asks about overall contents, summarize the EXCERPTS. "
            "If the answer is not present, say you don't know."
        ),
        "messages": [{
            "role": "user",
            "content": [{
                "type": "text",
                "text": f"Question:\n{question}\n\nEXCERPTS:\n{context}"
            }]
        }],
        "max_tokens": MAX_TOKENS,
        "temperature": 0,
        "top_p": 0.0
    }
    data = _invoke_json_with_retry(CHAT_MODEL_ID, payload)
    for b in data.get("content", []):
        if isinstance(b, dict) and b.get("type") == "text":
            t = b.get("text")
            if isinstance(t, str) and t.strip():
                return t.strip()
    return "I couldn't find the answer in the indexed context."
```