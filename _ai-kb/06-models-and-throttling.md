---
layout: default
title: "Models & Throttling (Sonnet → Haiku)"
---

I began chat inference on **Claude 3 Sonnet** for quality, but demo-style traffic quickly exposed **rate limits** and **token cost** as the dominant constraints. The fix was pragmatic: make **Claude 3 Haiku** the default for Q&A, keep Sonnet available for longer or more nuanced questions, and add a small set of **throttling/backoff** rules around Bedrock.

### Why switch (and how we kept quality)
- **Throughput & latency:** Haiku delivered lower p95 and higher requests/sec for short, grounded answers.
- **Cost control:** Smaller output windows and cheaper tokens reduced spiky spend.
- **Quality guardrails:** For prompts that need depth (or when users request “explain step-by-step”), **fallback to Sonnet** with a higher token budget.

### Prompt template (compact, grounded)
```text
You are a helpful assistant answering from the provided context.
If the answer is not in context, say you don't know.

Question:
{{question}}

Context (ranked by similarity, cut for length):
{{snippet_1}}
---
{{snippet_2}}
---
{{snippet_3}}

Answer concisely. If relevant, cite brief quotes from the snippets.
```

### Bedrock call (Python, trimmed)
```python
import json, boto3, os

bedrock = boto3.client("bedrock-runtime", region_name=os.getenv("AWS_REGION"))
MODEL_ID = os.getenv("MODEL_ID_CHAT", "anthropic.claude-3-haiku-20240307-v1:0")
MAX_TOKENS = int(os.getenv("MAX_TOKENS", "400"))
TEMPERATURE = float(os.getenv("TEMPERATURE", "0.2"))

def chat_with_context(question, context_text):
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": MAX_TOKENS,
        "temperature": TEMPERATURE,
        "messages": [
            {"role": "user",
             "content": f"Question:\n{question}\n\nContext:\n{context_text}\n\nAnswer:"}
        ]
    }
    resp = bedrock.invoke_model(
        modelId=MODEL_ID,
        body=json.dumps(body).encode("utf-8"),
        accept="application/json",
        contentType="application/json",
    )
    payload = json.loads(resp["body"].read())
    return payload.get("content", [{}])[0].get("text", "").strip()
```