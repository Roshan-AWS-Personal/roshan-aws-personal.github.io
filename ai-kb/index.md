---
title: Project 2 — AI Knowledge Base & Chat
layout: default
permalink: /ai-kb/
---

# AI Knowledge Base & Chat (Project 2)

**Table of Contents**
- [Overview](#overview)
- [Architecture](#architecture)
- [Infrastructure as Code](#infrastructure-as-code)
- [Ingestion (S3 → SQS → Lambda → DynamoDB)](#ingestion-s3--sqs--lambda--dynamodb)
- [Query API (FAISS + Bedrock)](#query-api-faiss--bedrock)
- [Auth, CORS & CDN](#auth-cors--cdn)
- [CI/CD & OIDC](#cicd--oidc)
- [Costs & Ops](#costs--ops)
- [Troubleshooting Notes](#troubleshooting-notes)
- [Improvements / Next](#improvements--next)

## Overview
A production-style RAG prototype: upload docs to S3, ingest to DynamoDB, build FAISS indexes in S3, and chat over them via an HTTP API. Dev/prod split, GitHub Actions OIDC, CloudFront + Cognito.

## Architecture
- **Upload:** S3 (prefix `docs/`)
- **Ingest:** S3 events → SQS → Lambda → DynamoDB (`file_upload_metadata`)
- **Index:** FAISS files in `s3://…/indexes/latest/`
- **Query:** API Gateway (HTTP) → Lambda → Bedrock (Titan embed + Claude) → FAISS
- **Auth:** Cognito Hosted UI for dashboard; tokens to API
- **CDN:** CloudFront in front of static UI + APIs (rules split)

## Infrastructure as Code
Terraform for everything; GitHub Actions deploy with environment input (`dev|prod`). OIDC role trust is scoped by repo + branch/environment.

## Ingestion (S3 → SQS → Lambda → DynamoDB)
- S3 notification: `ObjectCreated:*` on `docs/` to SQS.
- SQS queue policy restricts `aws:SourceArn` to the bucket and `aws:SourceAccount` to the account.
- Lambda reads, extracts metadata, writes to DynamoDB.

## Query API (FAISS + Bedrock)
- HTTP API route: `POST /query` (AWS_PROXY → Lambda).
- Lambda loads `index.faiss` and `meta.jsonl` from `indexes/latest/`.
- Embeds query (Titan), does k-NN with FAISS, composes context, calls Claude, streams back answer.

## Auth, CORS & CDN
- CORS: `content-type, authorization`; methods `POST, OPTIONS`.
- CloudFront: different behaviors for S3 origin vs API origin (no `Authorization` header to S3).

## CI/CD & OIDC
- Actions → `aws-actions/configure-aws-credentials@v4` with role chosen by input.
- Trust policy patterns for `ref:` and `environment:` subjects.
- Common pitfalls fixed: stage created before route; S3→SQS SourceArn mismatch; SQS visibility < Lambda timeout.

## Costs & Ops
- Idle pennies: S3 + SQS + DynamoDB on-demand + Lambda invocations.
- Pause by disabling S3 notifications or the EventBridge cron for index rebuilds.

## Troubleshooting Notes
- **OIDC:** ensure `sub` matches trust (`repo:…:ref:refs/heads/*` or `environment:*`).
- **API:** `$default` stage must depend on routes.
- **S3→SQS:** policy `Version` capitalization + `aws:SourceArn` exact bucket.
- **SQS→Lambda:** queue `visibility_timeout_seconds ≥ lambda.timeout`.

## Improvements / Next
- **Deletes:** `DELETE /files/{key}` → S3; S3 `ObjectRemoved:*` updates DynamoDB; enqueue index maintenance.
- **Index refresh:** debounce SQS → index-builder Lambda → write `indexes/{ts}/` → update `latest/`.
- **UI polish:** Tailwind, toasts, progress; presigned GETs; filters/search on list page.
