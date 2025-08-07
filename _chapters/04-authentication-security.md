---
layout: default
title: "Authentication & Security"
---

When I first added security to the file-upload API, I reached for the simplest mechanism: a static API key passed as an `X-API-KEY` header. My Lambda function would read this header, compare it to a hardcoded value, and allow or deny uploads. It was quick to implement but posed several problems:

<div align="center">
  <figure>
    <img src="{{ site.baseurl }}/assets/images/initial-security-flow.png" alt="Initial Security Flow" />
    <figcaption><strong>Figure 1.</strong> Basic Authentication Flow: X-API-KEY → API Gateway → Lambda → S3</figcaption>
  </figure>
</div>

- **Key Rotation Complexity:** Rotating the API key meant updating code, pushing a new Lambda version, and updating clients—a manual process vulnerable to human error.  
- **No User Context:** All requests looked identical; I couldn’t attribute actions to individual users or enforce per-user limits.  
- **Brittle Scaling:** Exposing the same key everywhere increased the blast radius if it leaked.

### Integrating Amazon Cognito

To solve these issues, I migrated to Amazon Cognito User Pools and adopted the OAuth2 authorization code grant flow:

<div align="center">
  <figure>
    <img src="{{ site.baseurl }}/assets/images/cognito-oauth2-flow.png" alt="Cognito OAuth2.0 Flow Flow" />
    <figcaption><strong>Figure 2.</strong> Cognito OAuth2 Flow: /authorize → Cognito User Pool → /token → JWT → API Gateway (Cognito Authorizer) → Lambda → S3</figcaption>
  </figure>
</div>

1. **User Login & Token Issuance**  
   Clients redirect to Cognito’s hosted login page (`/authorize?response_type=code&client_id=...`), authenticate, and receive an authorization code. They exchange that code at the `/token` endpoint for short-lived ID and Access Tokens (JWTs).

2. **Edge Authorization with CloudFront & API Gateway**  
   - **CloudFront Caching:** I put CloudFront in front of both my static UI and API Gateway. To preserve caching, I created a specific cache behavior forwarding only the `Authorization` and `Origin` headers to API Gateway.  
   - **Cognito Authorizer:** API Gateway’s built-in Cognito Authorizer validates incoming `Authorization: Bearer <JWT>` headers—checking signature, issuer, audience, and expiry—before invoking Lambda. This weeded out invalid tokens at the edge.

3. **Lambda Fallback & Migration**  
   During the migration, I kept the old API-key logic as a fallback. My Lambda first tried JWT validation; if none was present, it fell back to the static key. After monitoring usage for a week, I removed the deprecated key path entirely.

#### Challenges & Lessons Learned

- **Token Propagation in SPAs:** My single-page app needed fresh tokens across navigations. I integrated the AWS Cognito JavaScript SDK, which automatically refreshed tokens and stored them in secure, HTTP-only cookies.  
- **CloudFront Header Forwarding:** An initial misconfiguration stripped `Authorization`, causing 401s. The fix was a dedicated cache behavior for `/api/*` that forwards all headers and sets `Minimum TTL=0` for preflight requests.  
- **CORS with OAuth Redirects:** Redirects between my static site and the Cognito domain triggered CORS issues. I updated S3 and API Gateway CORS rules to allow the Cognito domain and included credentials in preflight responses.

**Key Takeaways:**

- **User-Centric Security:** OAuth2 and Cognito provide per-user context—enabling audit trails, usage quotas, and instant revocation.  
- **Edge Authorization Reduces Blast Radius:** Validating JWTs at CloudFront/API Gateway keeps unauthorized traffic out of Lambda.  
- **Graceful Migrations Matter:** A fallback path lets you roll out new auth without breaking existing clients.

By evolving from a naïve API-key model to a full Cognito OAuth2 flow, I hardened the API and embraced patterns critical for production-grade, user-centric applications.
