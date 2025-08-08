---
layout: default
title: "CORS & CDN Configuration: Taming CloudFront and Cross-Origin Hurdles"
---

Once my upload flows and security layers were humming along, I decided to put Amazon CloudFront in front of both my static S3 site and my API Gateway endpoints.  

On paper, this was a no-brainer: lower latency, built-in WAF integration, global edge coverage. But the moment I switched it on, I met my old nemesis — CORS — now wearing a CDN disguise.  

---

### 1. Initial CloudFront Setup

I started with a single CloudFront distribution and two origins:
- **Static Origin** → S3 bucket serving the UI  
- **API Origin** → API Gateway for `/api/*`  

By default, CloudFront only forwarded a handful of headers (`Host`, `User-Agent`, etc.) and cached responses for a full 24 hours. My S3 bucket had a permissive CORS policy (`GET, HEAD` from `https://my-portfolio.example.com`), and API Gateway was happily replying to CORS preflight requests for `POST`.

<div align="center">
    <figure class="figure-center">
    <img src="{{ site.baseurl }}/assets/images/initial-cloudfront.png" alt="Initial CloudFront Setup" />
    <figcaption><strong>Figure 5.</strong> Initial CloudFront distribution with two origins (S3 & API) and default behaviors.</figcaption>
    </figure>
</div>

In dev, it seemed fine. In prod? That’s when the cracks started showing.

### 2. The First Failures

Within hours, real users were hitting strange errors:
1. **Stale Preflight Caching** — Browsers sent `OPTIONS` preflight requests, CloudFront cached the response **without** `Access-Control-Allow-Origin`, and all follow-up `POST`/`PUT` requests failed.  
2. **Missing `Authorization`** — CloudFront stripped the `Authorization` header on API calls. My Cognito-protected endpoints suddenly returned 401s.  
3. **Lingering Edge Config** — Even after I fixed CORS rules in S3 and API Gateway, the old bad responses lived at edge locations for hours.

It was one of those moments where you think: _“Everything works perfectly… except in reality.”_

### 3. Iterative Fixes

<div align="center">
    <figure class="figure-center">
    <img src="{{ site.baseurl }}/assets/images/refined-cache-behaviors-flow.png" alt="Refined CloudFront Setup" />
    <figcaption><strong>Figure 6.</strong> Refined CloudFront cache behavior.</figcaption>
    </figure>
</div>

I tackled the issues one at a time.

**a) Custom Cache Behaviors**  
The big shift was defining separate behaviors for static and API paths, and explicitly forwarding the headers I needed.

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
    path_pattern           = "/api/*"
    target_origin_id       = "API-Gateway"
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
Key takeaways from this change:  
- **Separate cache behaviors** for UI vs API  
- Forwarded `Origin` (for CORS) and `Authorization` (for JWT) explicitly  
- Set `min_ttl` to 0 on `OPTIONS` so preflight checks always reach the origin

**b) Enhanced S3 CORS Policy**  
Once CloudFront was behaving, I tightened up S3’s CORS rules to fully support direct uploads and preflights:

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
Changes here: added `OPTIONS` and `PUT`, allowed all headers, and increased `MaxAgeSeconds` to reduce unnecessary preflight chatter.

**c) Cache Invalidation**  
The final piece was making sure fixes didn’t take hours to show up:

```yaml
- name: Invalidate CloudFront Cache
  run: |
    aws cloudfront create-invalidation \
      --distribution-id ${{ secrets.CF_DIST_ID }} \
      --paths "/*"
```
Now, any behavior or CORS tweak gets pushed instantly to every edge location.


### 4. Lessons Learned

If I had to distill this round of problem-solving:
- **Behavior-Level Precision** — Don’t rely on CloudFront’s defaults; forward exactly the headers your auth and CORS depend on.  
- **CORS Is End-to-End** — Browser → CDN → API → Lambda — every hop needs matching rules.  
- **Invalidate Aggressively** — Stale edge caches will hide your fixes and drive you mad.  

With these refinements, the CDN now delivers both my static site and API without breaking CORS or dropping auth headers. And next time I layer CloudFront in front of an API? I’ll remember that “just pointing it at the origin” is only the start of the journey.

-----------------------------