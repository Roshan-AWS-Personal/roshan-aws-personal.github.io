---
layout: default
title: "AI Knowledge Base (RAG)"
permalink: /ai-kb/
---

## Table of Contents
{% assign chapters = site['ai-kb'] | sort: 'path' %}
{% for chap in chapters %}
  {% assign anchor = chap.path | split:'/' | last | replace:'.md','' | slugify %}
- [{{ chap.title | default: anchor }}](#{{ anchor }})
{% endfor %}

---

{% for chap in chapters %}
  {% assign anchor = chap.path | split:'/' | last | replace:'.md','' | slugify %}
<a id="{{ anchor }}"></a>
## {{ chap.title | default: anchor }}
{{ chap.content | markdownify }}
<hr/>
{% endfor %}
