---
layout: default
title: "Event-Driven Logging & Notifications"
---

By this stage, uploads were zipping directly into S3 via pre-signed URLs, my API was happily behind CloudFront, and CORS was finally behaving. The core flow felt solid.  

However, we were missing **visibility**.  

My files would land in the bucket, but unless I went digging in the console, I had no easy way to answer questions like:
- Who uploaded this file?
- When did it arrive?
- How big is it?
- Should I be alerted right now?

In other words — my system was functional, but it wasn’t _talking back_ to me.

### Designing the Event-Driven Layer

I wanted something lightweight, automated, and decoupled from the upload path — a setup where uploads could happen at full speed, but the moment an object hit S3, my system would quietly log it and (optionally) ping me. That meant one obvious AWS tool: **S3 Event Notifications**.

The idea:
1. **S3 Upload Complete** — An object lands in the bucket.  
2. **Event Notification Fires** — S3 sends an event to a dedicated Lambda.  
3. **Log Metadata** — Lambda writes file info (name, size, timestamp, uploader) into a DynamoDB table.  
4. **Send Alerts** —  SES sends me (or an admin) an email.

<div align="center">
    <figure>
        <img src="{{ site.baseurl }}/assets/images/event-driven-logging.png" alt="Event-Driven Logging Architecture" />
        <figcaption><strong>Figure 6.</strong> S3 Event-Driven Logging & Notifications</figcaption>
    </figure>
</div>

### Implementation

The Lambda was straightforward but powerful. Using the `Records` array in the S3 event payload, I could grab everything I needed without ever touching the file bytes:

```python
def lambda_handler(event, context):
    logger.info("Received S3 event: %s", json.dumps(event))

    for record in event.get("Records", []):
        if record.get("eventName", "").startswith("ObjectCreated:"):
            bucket = record["s3"]["bucket"]["name"]
            key = record["s3"]["object"]["key"]
            size = record["s3"]["object"].get("size", 0)
            timestamp = record.get("eventTime") or datetime.utcnow().isoformat() + "Z"

            # Infer metadata
            filename = os.path.basename(key)
            ext = filename.rsplit(".", 1)[-1].lower()
            file_type = infer_type(ext)

            upload_id = str(uuid.uuid4())

            item = {
                "upload_id": upload_id,  # Using key as id if upload_id not generated elsewhere
                "filename": filename,
                "s3_bucket": bucket,
                "s3_key": key,
                "filesize": size,
                "timestamp": timestamp,
                "status": "uploaded",
                "content_type": file_type
            }

            try:
                logger.info("Writing item to DynamoDB: %s", item)
                table.put_item(Item=item)
            except Exception as e:
                logger.error("Failed to write to DynamoDB: %s", str(e))

    return {"statusCode": 200, "body": "Processed S3 event"}
```

The beauty of this setup is that uploads keep flowing exactly as before — the logging happens entirely on the side, without slowing down the client.

This layer gave me:
- **Full Traceability:** Every upload now has a permanent, queryable record in DynamoDB.  
- **Future-Proofing:** That metadata table could power an admin dashboard, usage reports, or automated workflows without touching the upload pipeline.

Next time, I’ll probably feed this into an SNS topic or EventBridge bus so other services can subscribe without me wiring them directly. But for now, DynamoDB + SES was simple win I wanted.

### Key points
- **Decoupled, post-upload logging:** S3 **Event Notifications** trigger a lightweight Lambda after the object is written—no impact on the upload fast path.
- **DynamoDB record per object:** Write an entry with `s3_bucket`, `s3_key`, `filesize`, `timestamp`, and status=`uploaded`. Use the S3 key as the primary key for easy correlation.
- **Optional alerts:** Fire **SES** emails. If SES fails, don’t fail the handler—log and continue.
- **Low cost, high leverage:** Pay per event invocation and a single DynamoDB write. No extra latency for the user.
-----------------