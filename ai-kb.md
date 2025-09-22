---
layout: default
title: "AI Knowledge Base (RAG)"
permalink: /ai-kb/
---

## Table of Contents
- [Introduction](/ai-kb/01-introduction)
- [Initial Architecture](/ai-kb/02-architecture-initial-plan)
- [GitHub OIDC: From Static Keys to Short-Lived Roles](/ai-kb/03-infrastructure-as-code-&-github-oidc)
- [Retrieval v1: OpenSearch Serverless](/ai-kb/04-retrieval-v1-opensearch-serverless)
- [Retrieval v2: Cost Pivot: FAISS on S3](/ai-kb/05-retrieval-v2-cost-pivot-faiss-on-s3)
- [Models & Throttling (Sonnet → Haiku)](/ai-kb/06-models-and-throttling)
- [Integration with Project 1 (Upload → Chat)](/ai-kb/07-integration-with-project-1)
- [Conclusion & Next Steps](/ai-kb/08-conclusion)

------


{% for chap in chapters %}
<a id="{{ chap.slug }}"></a>
## {{ chap.title }}
{{ chap.content | markdownify }}
{% endfor %}