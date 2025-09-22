---
layout: default
title: "AI Knowledge Base (RAG)"
permalink: /ai-kb/
---

{% assign chapters = site['ai-kb'] | sort: 'path' %}

## Table of Contents
{% for chap in chapters %}
- [{{ chap.title }}]({{ page.url | relative_url }}#{{ chap.slug }})
{% endfor %}

------

{% for chap in chapters %}
<a id="{{ chap.slug }}"></a>
## {{ chap.title }}
{{ chap.content | markdownify }}
{% endfor %}
