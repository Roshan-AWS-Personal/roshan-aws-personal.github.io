---
layout: default
title: "AI Knowledge Base (RAG)"
permalink: /ai-kb/
---

## Table of Contents
- [Introduction](/ai-kb/01-introduction)
- [Architecture: Initial Plan](/ai-kb/02-architecture-initial-plan)
- [Infrastructure as Code & GitHub OIDC](/ai-kb/03-infrastructure-as-code-github-oidc)
- [Retrieval v1: OpenSearch Serverless](/ai-kb/04-retrieval-v1)
- [Retrieval v2: FAISS on S3](/ai-kb/05-retrieval-v2)
- [Models & Throttling](/ai-kb/06-models-and-throttling)

------

{% assign chapters = site['ai-kb'] | sort: 'path' %}
{% for chap in chapters %}
<a id="{{ chap.slug }}"></a>
## {{ chap.title }}
{{ chap.content }}
<hr/>
{% endfor %}