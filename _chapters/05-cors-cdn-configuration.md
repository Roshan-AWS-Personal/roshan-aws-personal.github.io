---
layout: default
title: "CORS & CDN Configuration: Taming CloudFront and Cross-Origin Hurdles"
---
Once the upload flows and security layers were in place, I layered Amazon CloudFront on top of both my static S3 site and the API Gateway endpoints. While a CDN promised lower latency, built-in WAF integration, and global edge coverage, it immediately introduced subtle cross-origin and caching quirks.

### 1. Initial CloudFront Setup

I provisioned a single CloudFront distribution with two origins:

- **Static Origin** → S3 bucket serving the UI  
- **API Origin**    → API Gateway for `/api/*`  

By default, CloudFront only forwarded a minimal set of headers (`Host`, `User-Agent`, etc.) and cached aggressively (TTL = 24h). On the S3 side, I had a permissive CORS policy allowing `GET, HEAD` from `https://my-portfolio.example.com`; API Gateway was configured to respond to CORS preflight for `POST` requests.

<div align="center">
    <figure class="figure-center">
    <img src="{{ site.baseurl }}/assets/images/initial-cloudfront.png" alt="Initial CloudFront Setup" />
    <figcaption><strong>Figure 5.</strong> Initial CloudFront distribution with two origins (S3 & API) and default behaviors.</figcaption>
    </figure>
</div>

### 2. The First Failures
Almost immediately, real-world clients hit errors:

1. **Stale Preflight Caching**  
   Browsers send an `OPTIONS` preflight → CloudFront caches the response **without** `Access-Control-Allow-Origin`, so subsequent `POST` and `PUT` calls from the browser were blocked.

2. **Missing `Authorization`**  
   CloudFront stripped the `Authorization` header on requests to API Gateway. All JWT-based calls (after moving to Cognito) returned 401s.

3. **Lingering Edge Config**  
   Even after updating S3’s CORS rules and API Gateway’s preflight settings, the old 403 responses lived at edge locations for hours—breaking functionality long after the “fix” was live.

### 3. Iterative Fixes

I tackled each failure in turn:

#### a. Custom Cache Behaviors

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
    target_origin_id = "S3-Static"
    viewer_protocol_policy = "redirect-to-https"
    # forward all headers required by SPA and Cognito
    allowed_methods        = ["GET","HEAD","OPTIONS"]
    cached_methods         = ["GET","HEAD"]
    forwarded_values {
      query_string = false
      headers      = ["Origin"]
    }
    min_ttl = 0
  }

  ordered_cache_behavior {
    path_pattern = "/api/*"
    target_origin_id = "API-Gateway"
    viewer_protocol_policy = "https-only"
    allowed_methods        = ["GET","HEAD","OPTIONS","POST","PUT"]
    cached_methods         = ["GET","HEAD"]
    forwarded_values {
      query_string = false
      headers      = ["Authorization","Content-Type","Origin"]
    }
    min_ttl = 0
  }
}

```

- **Separate behaviors for static vs API paths**
- **Forward Origin (for CORS) and Authorization (for JWT)**
- **Minimize TTL on OPTIONS so each preflight hits the origin**

#### b) Enhanced S3 CORS Policy
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

- **Added OPTIONS and PUT**
- **Allowed all headers (*)**
- **Increased MaxAgeSeconds to reduce preflight frequency**

```yaml
- name: Invalidate CloudFront Cache
  run: |
    aws cloudfront create-invalidation \
      --distribution-id ${{ secrets.CF_DIST_ID }} \
      --paths "/*"
```

This ensured any behavior or CORS policy change propagated immediately to every edge location.

<div align="center">
    <figure class="figure-center">
    <img src="{{ site.baseurl }}/assets/images/refined-cache-behaviors-flow.png" alt="Refined CloudFront Setup" />
    <figcaption><strong>Figure 6.</strong> Refined Cloudfront cache behavior.</figcaption>
    </figure>
</div>

### 4. Key Takeaways
- **Behavior-Level Precision:** When fronting APIs, default caching is rarely enough—you must explicitly forward every header your auth/CORS relies on.

- **End-to-End CORS Design:** Browser → CDN → API → Lambda. Every hop must agree on allowed origins, methods, and headers.

- **Invalidation is Non-Optional:** Cache lifetimes at the edge can hide fixes; automate invalidations in your CI/CD.

With these refinements, my static site and API now deliver reliably from the edge—complete with robust CORS support and dynamic authorization flows. In the final chapter, we’ll examine how I decoupled logging and notifications into an event-driven pipeline using DynamoDB and SES.

---------------------------------------
