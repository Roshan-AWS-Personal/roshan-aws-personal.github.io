---
layout: default
title: Introduction
---

Hello and welcome to my first project! This page is the first tangible demonstration of applying my software engineering experience to my newly earned AWS Solutions Architect skills. What started as a simple file-upload API, just a Lambda writing to S3, grew into a production-minded template you can brand for a client and ship.

The result: a resilient web front end served from **Amazon S3** and accelerated by **CloudFront**, secured by **Amazon Cognito** (OAuth 2.0), with **Lambda behind API Gateway** for control APIs, **DynamoDB** for upload metadata, and **SES** for notifications. The static UI is intentionally generic so it can be swapped for client branding, while the surrounding infrastructure stays rock-solid.

A few design choices shaped everything that follows:
- **One CloudFront in front of S3 and API**: default behavior serves the UI from S3; path behaviors route **`/upload`** and **`/files`** to API Gateway with auth headers forwarded.
- **Presigned uploads**: the browser streams directly to S3 at native speed; the API just authorizes and issues URLs.
- **Right-sized CORS**: UI→API calls are same-origin (no CORS); S3 still has a strict CORS policy for the browser’s presigned `PUT`.

In the chapters that follow, I trace how the prototype became a real system:

1. **Infrastructure as Code** — Turning a console-click sketch into Terraform across environments (and the little gotchas, like `${...}` escaping in templates).  
2. **Authentication & Security** — Moving from static tokens to **Cognito OAuth2**, and handling 401s cleanly in the UI.  
3. **Scalable Data Flows** — Switching to **presigned URLs** so S3 does the heavy lifting and costs stay flat.  
4. **CORS & CDN Strategies** — Eliminating API CORS via CloudFront, forwarding the right headers, and caching only what should be cached.  
5. **Logging & Notifications** — Writing upload intent to **DynamoDB** and sending **SES** emails at presign; optionally confirming final writes with **S3 Event Notifications**.

Each section details the challenge I encountered, the solution I implemented, and the lessons that solidified my understanding of AWS architectures.

--------------------------------------------