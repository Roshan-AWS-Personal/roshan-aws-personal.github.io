---
layout: default
title: "CORS & CDN Configuration: Taming CloudFront and Cross-Origin Hurdles"
---

Once the upload flows and security layers were in place, I introduced Amazon CloudFront in front of both my static S3 site and the API Gateway endpoints. The CDN promised lower latency, built-in WAF integration, and global edge coverage — but it also brought subtle cross-origin (CORS) and caching quirks that quickly surfaced in production.

### 1. Initial CloudFront Setup
I provisioned a single CloudFront distribution with two origins:
- **Static Origin** → S3 bucket serving the UI  
- **API Origin** → API Gateway for `/api/*`  

By default, CloudFront forwarded only minimal headers (`Host`, `User-Agent`, etc.) and cached responses aggressively (TTL = 24h). On the S3 side, the CORS policy allowed `GET, HEAD` from `https://my-portfolio.example.com`. API Gateway responded to CORS preflights for `POST` requests.

<div align="center">
    <figure class="figure-center">
    <img src="{{ site.baseurl }}/assets/images/initial-cloudfront.png" alt="Initial CloudFront Setup" />
    <figcaption><strong>Figure 5.</strong> Initial CloudFront distribution with two origins (S3 & API) and default behaviors.</figcaption>
    </figure>
</div>

### 2. Early Failures
Once live, clients began hitting problems:
1. **Stale Preflight Caching** — Browsers sent an `OPTIONS` preflight; CloudFront cached the response without `Access-Control-Allow-Origin`, so subsequent `POST` and `PUT` calls failed.  
2. **Missing `Authorization` Header** — CloudFront stripped the `Authorization` header for API Gateway requests, breaking JWT-protected endpoints after Cognito integration.  
3. **Lingering Edge Config** — Updated S3 CORS rules and API Gateway preflight settings took hours to propagate because outdated 403 responses were cached at edge locations.

### 3. Iterative Fixes
<div align="center">
    <figure class="figure-center">
    <img src="{{ site.baseurl }}/assets/images/refined-cache-behaviors-flow.png" alt="Refined CloudFront Setup" />
    <figcaption><strong>Figure 6.</strong> Refined CloudFront cache behavior.</figcaption>
    </figure>
</div>
To resolve these, I implemented targeted cache behaviors, enhanced CORS rules, and automated invalidations.

**a) Custom Cache Behaviors**
```hcl
resource "aws_cloudfront_distribution" "cdn" {
  # … other config …

  origin {
    domain_name = aws_s3_bucket.website.bucket_regional_domain_name
    origin_id   = "S3-Static"
  }

  origin {
    domain_name = aws_api_gateway_deployment.api.invoke_url
    origin_id   = "API-Gateway"
  }

  default_cache_behavior {
    target_origin_id       = "S3-Static"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    forwarded_values {
      query_string = false
      headers      = ["Origin"]
    }
    min_ttl = 0
  }

  ordered_cache_behavior {
    path_pattern          = "/api/*"
    target_origin_id      = "API-Gateway"
    viewer_protocol_policy = "https-only"
    allowed_methods        = ["GET", "HEAD", "OPTIONS", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    forwarded_values {
      query_string = false
      headers      = ["Authorization", "Content-Type", "Origin"]
    }
    min_ttl = 0
  }
}
```
Key points: separate cache behaviors for static vs API paths, forward `Origin` (CORS) and `Authorization` (JWT), and set `min_ttl` to 0 on `OPTIONS` so every preflight hits the origin.

**b) Enhanced S3 CORS Policy**
```xml
<CORSConfiguration>
  <CORSRule>
    <AllowedOrigin>https://my-portfolio.example.com</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedMethod>PUT</AllowedMethod>
    <AllowedMethod>OPTIONS</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
    <ExposeHeader>ETag</ExposeHeader>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
  </CORSRule>
</CORSConfiguration>
```
Key points: added `OPTIONS` and `PUT`, allowed all headers (`*`), and increased `MaxAgeSeconds` to reduce preflight frequency.

**c) Cache Invalidation**
```yaml
- name: Invalidate CloudFront Cache
  run: |
    aws cloudfront create-invalidation \
      --distribution-id ${{ secrets.CF_DIST_ID }} \
      --paths "/*"
```
Key point: automated invalidations ensured CORS and behavior changes took effect immediately at all edge locations.

### 4. Key Takeaways
- **Behavior-Level Precision:** Explicitly forward every header required by authentication and CORS.  
- **End-to-End CORS Design:** Browser → CDN → API → Lambda — all layers must align on origins, methods, and headers.  
- **Automated Invalidations:** Prevent stale edge caches from hiding fixes.

With these changes, my static site and API deliver reliably from the edge, with robust CORS support and secure, dynamic authorization flows.
