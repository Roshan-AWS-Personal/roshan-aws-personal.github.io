---
layout: default
title: "Integration with Project 1 (Upload → Chat)"
---

With the RAG chatbot and page drafted, it was time to incorporate it with my previous project to finish everything up. Here, we experience the challenges of managing Terraform state, multiple backends, and untangling the importing of modules so that one repo can be in charge of all AWS resources. 

## End-to-End Flow (what the user experiences)
1. **Upload a file** in Project 1 → S3 (per-type buckets).  
2. **Ingest Lambda** (Project 2) chunks + embeds → writes **`indexes/latest/{index.faiss, meta.jsonl}`** to S3.  
3. **Chat page** (new link from Project 1 header) calls **Query API** → FAISS top-k → Bedrock **Haiku** → returns answer + sources.

### Migration steps (what we actually did)
- Create new S3/DynamoDB backends; run:
  - `terraform init -migrate-state` (move local/old state to new backend).
- Normalize resource prefixes (`ai-kb-dev` / `ai-kb-prod`) and module inputs.
- Where addresses changed, use targeted moves (avoid re-create):  
  - `terraform state mv module.old.aws_iam_role.ingest_exec module.core.aws_iam_role.ingest_exec`
- Add **`output "github_oidc_role_arn"`** in `oidc.tf` so CI can assume the right role.  
- Re-plan each env and apply in this order: **IAM/OIDC → S3/backends → API/Lambdas → CloudFront**.

> **Problems I faced**
> - Changing resource names without `state mv` will destroy/recreate.  
> - Keep the old backend reachable during migration (no half-moved states).  
> - Always migrate **dev** first; copy the pattern to **prod** only after a clean plan.

## Path routing (what goes where)

| Viewer URL (browser) | CloudFront behavior | Target origin | Origin path |
|---|---|---|---|
| `/index.html`, `/chat.html`, assets | **default** | S3 (UI) | — |
| `/upload*` | ordered behavior | REST API | `/${stage}` if stage ≠ `$default` |
| `/files*`  | ordered behavior | REST API | `/${stage}` if stage ≠ `$default` |
| `/query*`  | ordered behavior | HTTP API | `/${stage}` if stage ≠ `$default` |

- The **origin path** is computed from `local.*_api_stage`. For HTTP API with `$default`, the origin path is **empty** so `/query` on CloudFront maps to `/query` at API Gateway.
- API behaviors use the **Managed “CachingDisabled”** cache policy and **Managed “AllViewerExceptHostHeader”** origin request policy so:
  - **Authorization** and other viewer headers reach API Gateway.
  - Nothing is cached for POSTs.

## HTML → CloudFront → API Gateway flow
Because the chat page is served by the same CloudFront domain, the simplest/cleanest call is a **same-origin** fetch:

```html
<script>
  // If you want zero env-plumbing for the path, just use a relative URL:
  // const API = "/query";

  // If you prefer templating (keeps your existing placeholders), point to CF + /query:
  // window.APP_CONFIG.apiUrl = "https://${CLOUDFRONT_DOMAIN}/query";
  const cfg = window.APP_CONFIG || {};
  const API = cfg.apiUrl || "/query";

  async function ask() {
    const token = localStorage.getItem("id_token");   // works with/without authorizer
    const q = document.getElementById("q").value.trim();
    if (!q) return;

    const r = await fetch(API, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        // Safe even if the API currently doesn't enforce auth; required once you add a JWT authorizer
        "Authorization": token ? ("Bearer " + token) : ""
      },
      body: JSON.stringify({ q, k: 8 })
    });

    const data = await r.json();
    // render answer + sources...
  }
</script>

---
