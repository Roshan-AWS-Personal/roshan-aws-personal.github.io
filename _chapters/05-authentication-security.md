---
layout: default
title: "Authentication & Security: From Static Tokens to Cognito OAuth2"
---

When I first added security to the file-upload API, I reached for the simplest mechanism: a static API key passed as an `X-API-KEY` header. My Lambda function would read this header, compare it to a hardcoded value, and allow or deny uploads. It was quick to implement but posed several problems:

<div align="center">
  <figure>
    <img src="{{ site.baseurl }}/assets/images/initial-security-flow.png" alt="Initial Security Flow" />
    <figcaption><strong>Figure 4.</strong> Basic Authentication Flow: X-API-KEY → API Gateway → Lambda → S3</figcaption>
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
    <figcaption><strong>Figure 5.</strong> Cognito OAuth2 Flow: /authorize → Cognito User Pool → /token → JWT → API Gateway (Cognito Authorizer) → Lambda → S3</figcaption>
  </figure>
</div>

1. **User Login & Token Issuance**
   Clients redirect to Cognito’s hosted login page (`/authorize?response_type=code&client_id=...`), authenticate, and receive an authorization code. They exchange that code at the `/token` endpoint for short-lived ID and Access Tokens (JWTs).

2. **Edge Authorization with CloudFront & API Gateway**
   - **CloudFront Caching:** I put CloudFront in front of both my static UI and API Gateway. To preserve caching, I created a specific cache behavior forwarding only the `Authorization` and `Origin` headers to API Gateway.
   - **Cognito Authorizer:** API Gateway’s built-in Cognito Authorizer validates incoming `Authorization: Bearer <JWT>` headers—checking signature, issuer, audience, and expiry—before invoking Lambda. This weeded out invalid tokens at the edge.

3. **Handling expiry**
If the API returns **401**, the UI clears the token and redirects back to the Cognito login page. (No refresh tokens in implicit flow; the UI simply re-auths.)

### CloudFront in front of UI + API

One CloudFront distribution fronts both origins:

- **Default behavior → S3** (UI)
- **Path behaviors** → API Gateway for **`/upload`** and **`/files`**

For the API paths, I used:
- **Managed-CachingDisabled** (no caching of authenticated responses)
- **Managed-AllViewerExceptHostHeader** (forwards viewer headers like `Authorization`, while letting CloudFront set `Host` correctly)

### CORS

- **UI → API** is now **same-origin** through CloudFront → browsers don’t do CORS preflights for those calls.  
- **Browser → S3** (presigned `PUT`) is **cross-origin**, so **S3 CORS** allows my site domain, `PUT/GET/HEAD/OPTIONS`, and `*` headers, and exposes `ETag`.

### Challenges & lessons

- **Forwarding `Authorization`**: Initial header forwarding missed the token, causing 401s. The fix was using the AWS managed origin-request policy **AllViewerExceptHostHeader** on the API behaviors.  
- **Tokens in the browser**: Storing the `id_token` in `localStorage` is simple and works for this project; on 401 the UI re-auths. (For larger apps, consider PKCE + code flow and/or cookie-based sessions.)  
- **Keep the edge simple**: Let **API Gateway** do JWT validation; CloudFront just routes and forwards the right headers.

By evolving from a naïve API-key model to a full Cognito OAuth2 flow, I hardened the API and embraced patterns critical for production-grade, user-centric applications.

### Key points
- **Per-user JWTs, not a shared key:** Users authenticate via Cognito Hosted UI; the UI stores an `id_token` and sends `Authorization: Bearer <token>` to the API.
- **JWT validated before Lambda:** API Gateway’s **Cognito Authorizer** verifies issuer/audience/signature/expiry; only valid requests reach your functions.
- **401 handling:** On token expiry or invalidation, the UI clears the token and redirects to Cognito login (implicit flow—no refresh token).
- **Principle of least exposure:** No static API keys in the frontend; tokens are short-lived and user-scoped, reducing blast radius.
- **Consistent domains & redirects:** Cognito redirect URIs point to the CloudFront domain so auth completes cleanly in all environments.

---------------