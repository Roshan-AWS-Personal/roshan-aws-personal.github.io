---
layout: default
title: "Introduction"
---

Hello and welcome to my second project! Building on the foundations of Project 1, this page documents my journey into **Retrieval-Augmented Generation (RAG)** using AWS. What began as a simple experiment with OpenSearch Serverless grew into a leaner, cost-aware design using **S3 + FAISS**, while keeping the same production-minded principles of Infrastructure as Code, CI/CD, and modular Terraform.

The result: a lightweight knowledge base where users can upload documents and then *chat with them* — powered by **Amazon Bedrock** (Titan embeddings + Claude), with infrastructure deployed end-to-end via **Terraform + GitHub Actions**. Over time, I migrated this project into Project 1’s repository as reusable modules, creating one cohesive, end-to-end portfolio platform.

> **What is RAG?**  
> Retrieval-Augmented Generation is a pattern where a user’s query is first enriched with relevant context retrieved from a knowledge base (e.g., documents stored as embeddings), and only then passed to a large language model. Instead of asking the model to “know everything,” RAG grounds answers in your own data — reducing hallucinations, improving relevance, and controlling costs.

In the chapters that follow, I trace how the prototype became a real system:

* **Infrastructure as Code** — Setting up OIDC with GitHub Actions and Terraform modules for dev/prod parity.  
* **Retrieval v1: OpenSearch Serverless** — Early design using managed vector search, and the lessons from cost trade-offs.  
* **Retrieval v2: FAISS on S3** — Pivoting to a Lambda + FAISS pipeline for near-zero idle cost.  
* **Models & Throttling** — Switching from Claude Sonnet to Haiku to balance quality, cost, and API rate limits.  
* **Integration with Project 1** — Folding modules into the first project for a unified, production-ready showcase.  

Each section captures the challenges I faced, the solutions I built, and the lessons that deepened my understanding of **cost-efficient, production-minded AWS architectures**.

-------------------------------------------------------------------------------------------------------------------