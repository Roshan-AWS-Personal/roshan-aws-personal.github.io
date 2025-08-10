---
layout: default
title: Scalable Data Flows
---

With my CI/CD runners humming and Terraform modules orchestrating everything from S3 buckets to API Gateway, I set out to implement the most straightforward upload pipeline I could imagine: client → API Gateway → Lambda → S3. In development, this flow felt very coherent, one click in GitHub Actions, and I could watch a fresh S3 bucket appear in the console complete with IAM policies, CORS rules, and a working Lambda ready to accept file payloads. Switching between dev and prod was as simple as choosing a different environment in the workflow inputs and watching Terraform seamlessly apply the right state.

<div align="center">
  <figure>
    <img src="{{ site.baseurl }}/assets/images/initial-upload-flow.png" alt="Initial File Upload Flow" />
    <figcaption><strong>Figure 1.</strong> Initial File Upload Flow: client → API Gateway → Lambda → S3</figcaption>
  </figure>
</div>

Yet, once the prototype was in place, I began to ask tougher questions: what happens when a user uploads a 50 MB video? How would sudden spikes in traffic affect my Lambdas? Is my function ever going to timeout or run out of memory? And beyond raw performance, what visibility did I have into each uploaded file’s size, type, or provenance?

I realized that in this initial design, every byte of every upload passed through my Lambda—tightly coupling data processing with business logic as well as other serious limitations:

* **Execution Time & Cost:** Large payloads prolonged Lambda execution, driving up GB‑second costs and risking throttles.
* **Constrained Observability:** My only logs were what I emitted in the function; I couldn’t capture granular upload metrics without adding more code.
* **Scaling Synergy:** While Lambda itself scales horizontally, high concurrency during big-file bursts could exhaust account-level limits.

I toyed briefly with the idea of splitting everything into microservices—one service for validation, another for upload orchestration, and yet another for metadata logging—but that would introduce distributed tracing, message queues, and more operational overhead. For now, I chose a middle path: offload the heavy lifting to S3 while maintaining a clear, event-driven pattern.

<div align="center">
    <figure>
        <img src="{{ site.baseurl }}/assets/images/presigned-url-flow.png" alt="Pre-Signed URL Upload Flow" />
        <figcaption><strong>Figure 2. </strong> Pre-Signed URL Upload Flow: client ⇄ URL-Generator Lambda ⇄ S3 & async logging</figcaption>
    </figure>
</div>

### Embracing Pre-Signed URLs

Instead of handling binary data in Lambda, I introduced a lightweight endpoint whose sole purpose was to generate pre-signed `PUT` URLs:

```python
# Generate a 5-minute pre-signed PUT URL for S3
presigned_url = s3.generate_presigned_url(
    ClientMethod='put_object',
    Params={
        'Bucket': BUCKET_NAME,
        'Key': key,
        'ContentType': content_type
    },
    ExpiresIn=300
)
```
1. **Generate URL:** The client sends a small request (`filename`, `contentType`) to `GET /upload` API.
2. **Validate & Sign:** A focused Lambda inspects or sanitizes `filename`, and calls S3’s `getSignedUrl('putObject')` with a short expiration window.
3. **Direct Upload:** The client uploads the file bytes straight to S3 using the returned URL—bypassing Lambda.
4. **Async Logging:** An S3 event notification fires a separate `log-upload` Lambda that records metadata in DynamoDB and optionally triggers SES alerts.

This refactoring untangled the data plane from my compute plane:

* **Performance Gains:** Large files no longer consume Lambda compute time—S3 handles the transfer at native speeds.
* **Cost Savings:** S3’s storage and PUT request pricing is far more economical than Lambda GB‑seconds.
* **Enhanced Observability:** The asynchronous logger captures file size, upload time, and user context, giving me a rich audit trail.

Of course, this pattern isn’t without its trade‑offs. Clients now implement a two-step process and must gracefully handle expired URLs—adding retry logic in our front‑end code. I also had to refine bucket-level CORS rules to allow `PUT` requests from our domain, including headers like `Content-Type` and `x-amz-meta-*`. But these additions felt like worthwhile investments for a more robust, scalable system.

### Reflecting on Coupling and Scope
Looking at the result, I can see it’s still a relatively cohesive service rather than a full microservices ecosystem. All upload-related concerns remain within a narrow boundary: URL generation, file ingestion, and metadata logging. That tight coupling actually serves simplicity—there’s no distributed saga or queue choreography to debug. In future iterations, I could extract the logger into an event bus or introduce image processing pipelines, but for this portfolio piece, I chose clarity and demonstrable best practices over building every conceivable component.

-----------------------------
