---
layout: default
title: "CORS & CDN Configuration: Taming CloudFront and Cross-Origin Hurdles"
---

Once the upload flows and security layers were in place, I introduced Amazon CloudFront in front of both my static S3 site and the API Gateway endpoints. The CDN promised lower latency, built-in WAF integration, and global edge coverage — but it also brought subtle cross-origin (CORS) and caching quirks that quickly surfaced in production.

### 1. Initial CloudFront Setup
I provisioned a single CloudFront distribution with two origins:
- **Static Origin** → S3 bucket serving the UI  
- **API Origin** → API Gateway for the specific paths /upload and /files (the API stage is applied via origin_path, e.g., /dev)

By default, CloudFront forwarded only minimal headers (`Host`, `User-Agent`, etc.) and cached responses aggressively (TTL = 24h). On the S3 side, the CORS policy allowed `GET, HEAD` from `https://my-portfolio.example.com`. API Gateway responded to CORS preflights for `POST` requests.


### 2. Early Failures
Once live, clients began hitting problems:
1. **Stale Preflight Caching** — Browsers sent an `OPTIONS` preflight; CloudFront cached the response without `Access-Control-Allow-Origin`, so subsequent `POST` and `PUT` calls failed.  
2. **Missing `Authorization` Header** — CloudFront stripped the `Authorization` header for API Gateway requests, breaking JWT-protected endpoints after Cognito integration.  
3. **Lingering Edge Config** — Updated S3 CORS rules and API Gateway preflight settings took hours to propagate because outdated 403 responses were cached at edge locations.

### 3. Iterative Fixes
<div align="center">
    <figure class="figure-center">
    <img src="{{ site.baseurl }}/assets/images/refined-cache-behaviors-flow.png" alt="Refined CloudFront Setup" />
    <figcaption><strong>Figure 3.</strong> Refined CloudFront cache behavior.</figcaption>
    </figure>
</div>
To resolve these, I implemented targeted cache behaviors, enhanced CORS rules, and automated invalidations.

#### a) Custom Cache Behaviors
```hcl
# Cache policies (managed)
data "aws_cloudfront_cache_policy" "optimized" { name = "Managed-CachingOptimized" }
data "aws_cloudfront_cache_policy" "disabled"  { name = "Managed-CachingDisabled" }

# Origin request policies
# S3: forward nothing
resource "aws_cloudfront_origin_request_policy" "s3_safe" {
  name = "s3-safe-policy"
  cookies_config       { cookie_behavior = "none" }
  headers_config       { header_behavior = "none" }
  query_strings_config { query_string_behavior = "none" }
}

# API: forward all viewer headers except Host (includes Authorization)
data "aws_cloudfront_origin_request_policy" "all_viewer_except_host" {
  name = "Managed-AllViewerExceptHostHeader"
}

# Region helper for API origin hostname
data "aws_region" "current" {}

# Distribution (core parts)
resource "aws_cloudfront_distribution" "frontend" {
  enabled             = true
  default_root_object = "index.html"

  # S3 origin
  origin {
    domain_name              = aws_s3_bucket.frontend_site.bucket_regional_domain_name
    origin_id                = "s3-upload-site"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3_oac.id
  }

  # API origin (host-only; stage via origin_path)
  origin {
    domain_name = "${aws_api_gateway_rest_api.upload_api.id}.execute-api.${data.aws_region.current.name}.amazonaws.com"
    origin_id   = "api-gateway-origin"
    origin_path = "/${aws_api_gateway_stage.stage.stage_name}"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Default behavior → S3
  default_cache_behavior {
    target_origin_id        = "s3-upload-site"
    viewer_protocol_policy  = "redirect-to-https"
    allowed_methods         = ["GET", "HEAD"]
    cached_methods          = ["GET", "HEAD"]
    compress                = true

    cache_policy_id          = data.aws_cloudfront_cache_policy.optimized.id
    origin_request_policy_id = aws_cloudfront_origin_request_policy.s3_safe.id
  }

  # /upload → API (no cache; forward viewer headers except Host)
  ordered_cache_behavior {
    path_pattern            = "/upload"
    target_origin_id        = "api-gateway-origin"
    viewer_protocol_policy  = "https-only"
    allowed_methods         = ["GET", "HEAD", "OPTIONS", "POST", "PUT", "DELETE", "PATCH"]
    cached_methods          = ["GET", "HEAD", "OPTIONS"]
    compress                = true

    cache_policy_id          = data.aws_cloudfront_cache_policy.disabled.id
    origin_request_policy_id = data.aws_cloudfront_origin_request_policy.all_viewer_except_host.id
  }

  # /files → API (no cache; forward viewer headers except Host)
  ordered_cache_behavior {
    path_pattern            = "/files"
    target_origin_id        = "api-gateway-origin"
    viewer_protocol_policy  = "https-only"
    allowed_methods         = ["GET", "HEAD", "OPTIONS", "POST", "PUT", "DELETE", "PATCH"]
    cached_methods          = ["GET", "HEAD", "OPTIONS"]
    compress                = true

    cache_policy_id          = data.aws_cloudfront_cache_policy.disabled.id
    origin_request_policy_id = data.aws_cloudfront_origin_request_policy.all_viewer_except_host.id
  }

  restrictions { geo_restriction { restriction_type = "none" } }

  viewer_certificate { cloudfront_default_certificate = true }
}
```

#### b) Enhanced S3 CORS Policy
```xml
<CORSConfiguration>
  <CORSRule>
    <AllowedOrigin>https://YOUR-CF-DOMAIN.example.com</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedMethod>PUT</AllowedMethod>
    <AllowedMethod>HEAD</AllowedMethod>
    <AllowedMethod>OPTIONS</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
    <ExposeHeader>ETag</ExposeHeader>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
  </CORSRule>
</CORSConfiguration>
```

#### c) Cache Invalidation
```yaml
- name: Invalidate CloudFront Cache
  run: |
    aws cloudfront create-invalidation \
      --distribution-id ${{ secrets.CF_DIST_ID }} \
      --paths "/*"
```

### Key Points
- **One distribution, two origins:** default behavior → S3 (UI), path behaviors /upload and /files → API Gateway.
- **API header forwarding:** use AWS Managed-AllViewerExceptHostHeader so the API receives all viewer headers (incl. Authorization); Host is set by CloudFront to the origin host.
- **Caching:** Managed-CachingOptimized for S3 (static assets), Managed-CachingDisabled for API (always fresh, TTL = 0).
- **CORS reality now:** UI→API calls are same-origin, so the browser doesn’t preflight them. S3 CORS is still required for presigned PUTs (browser → S3).
- **Automated CloudFront invalidations:** CORS and behavior changes propagate instantly to all edge locations.

------------------------