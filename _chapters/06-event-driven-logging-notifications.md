---
layout: default
title: "Event-Driven Logging & Notifications"
---

By this stage, uploads were zipping directly into S3 via pre-signed URLs, my API was happily behind CloudFront, and CORS was finally behaving. The core flow felt solid.  

But there was still something missing: **visibility**.  

Sure, files were landing in the bucket, but unless I went digging in the console, I had no easy way to answer questions like:
- Who uploaded this file?
- When did it arrive?
- How big is it?
- Should I be alerted right now?

In other words — my system was functional, but it wasn’t _talking back_ to me.

---

### Designing the Event-Driven Layer

I wanted something lightweight, automated, and decoupled from the upload path — a setup where uploads could happen at full speed, but the moment an object hit S3, my system would quietly log it and (optionally) ping me. That meant one obvious AWS tool: **S3 Event Notifications**.

The idea:
1. **S3 Upload Complete** — An object lands in the bucket.  
2. **Event Notification Fires** — S3 sends an event to a dedicated Lambda.  
3. **Log Metadata** — Lambda writes file info (name, size, timestamp, uploader) into a DynamoDB table.  
4. **Send Alerts (Optional)** — If the file meets certain criteria, SES sends me (or an admin) an email.

<div align="center">
    <figure>
        <img src="{{ site.baseurl }}/assets/images/event-driven-logging.png" alt="Event-Driven Logging Architecture" />
        <figcaption><strong>Figure 7.</strong> S3 Event-Driven Logging & Notifications</figcaption>
    </figure>
</div>

---

### Implementation

The Lambda was straightforward but powerful. Using the `Records` array in the S3 event payload, I could grab everything I needed without ever touching the file bytes:

```python
def handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        size = record['s3']['object']['size']
        timestamp = record['eventTime']
        uploader = record.get('userIdentity', {}).get('principalId', 'unknown')

        # Write metadata to DynamoDB
        table.put_item(Item={
            'file_key': key,
            'bucket': bucket,
            'size': size,
            'timestamp': timestamp,
            'uploader': uploader
        })

        # Optional SES alert
        if size > ALERT_THRESHOLD:
            ses.send_email(
                Source=FROM_EMAIL,
                Destination={'ToAddresses': [ADMIN_EMAIL]},
                Message={
                    'Subject': {'Data': 'Large File Uploaded'},
                    'Body': {'Text': {'Data': f'File {key} uploaded, size {size} bytes'}}
                }
            )
```

The beauty of this setup is that uploads keep flowing exactly as before — the logging happens entirely on the side, without slowing down the client.

---

### Why This Matters

This layer gave me:
- **Full Traceability:** Every upload now has a permanent, queryable record in DynamoDB.  
- **Proactive Awareness:** SES can nudge me when something unusual happens — a huge file, a suspicious user, or a specific file type.  
- **Future-Proofing:** That metadata table could power an admin dashboard, usage reports, or automated workflows without touching the upload pipeline.

---

### Trade-Offs and Lessons

The biggest decision was keeping logging **post-upload** rather than validating in real time. That means a bad file might hit the bucket before I flag it — but for my current needs, the scalability and simplicity were worth it. If I needed stronger enforcement, I could still combine this with pre-upload checks.

Next time, I’ll probably feed this into an SNS topic or EventBridge bus so other services can subscribe without me wiring them directly. But for now, DynamoDB + SES was simple win I wanted.
