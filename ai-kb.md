---
layout: default
title: "AI KB — All Chapters"
permalink: /ai-kb/all/
---

## Table of Contents
- [Introduction](/ai-kb/01-introduction)
- [Infrastructure as Code](/ai-kb/02-infrastructure-as-code)
- [Retrieval v1: OpenSearch Serverless](/ai-kb/03-retrieval-v1-opensearch)
- [Cost Pivot: FAISS on S3](/ai-kb/04-cost-pivot-faiss-s3)
- [Models & Throttling](/ai-kb/05-models-and-throttling)
- [Integration with Project 1](/ai-kb/06-integration-with-project-1)
- [Conclusion & Next Steps](/ai-kb/07-conclusion-next-steps)

{% assign chapters = site['ai-kb'] | sort: 'path' %}
{% for chap in chapters %}
<a id="{{ chap.slug }}"></a>
## {{ chap.title }}
{{ chap.content }}
<hr/>
{% endfor %}