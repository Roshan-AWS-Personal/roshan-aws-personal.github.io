---
layout: default
title: "Conclusion"
---

What began as a tiny “upload a file to S3” exercise turned into a project about making choices: where to keep things simple, where to lean on the platform, and where to add just enough structure so the system can grow without getting brittle.

The biggest shift was learning to design around the edges where the services interacted with each other. The places where a browser meets a CDN, where identity meets an API, and where files meet storage. Once those boundaries were clear, the rest followed naturally: a single public entry point, short-lived upload permissions, and lightweight logging that lets the system “speak back” without slowing anything down.

### Where I’d take it next
- A small **admin dashboard** over the upload metadata (search, filters, per-user history).
- Optional **post-upload processing** (image resizing, document parsing) triggered by S3 events.
- A unified `/api/{proxy+}` route later, then a single `/api/*` behavior in the CDN as endpoints grow.
- **Edge protections** (WAF/rate limits) and a custom domain/certificate for a fully polished surface.

More than anything, this project is proof that I can design, build, and evolve a production-grade system as well as tell the story of how it came to life.