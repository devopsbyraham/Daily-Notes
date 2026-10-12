# AWS S3 — Beginner-to-Architect Notes + Hands-on

## 1. What is Amazon S3?
**Amazon Simple Storage Service (S3)** is AWS object storage. It stores data as **objects** inside **buckets**.

Examples: images, audio, video, PDFs, backups, logs, datasets, static website files, software artifacts, and many other file types.

**Memory line:** Bucket = container; object = file + metadata + key.

A bucket is not exactly a traditional folder. S3 uses a flat object namespace; `/` in object keys is commonly used to display folder-like prefixes.

## 2. Core S3 concepts
- **Bucket:** container for objects. Bucket names are globally unique within an AWS partition.
- **Object:** data plus metadata and a key.
- **Key:** object's name/path-like identifier, e.g. `images/profile.png`.
- **Region:** where the bucket is created; a bucket's Region cannot simply be changed later.
- **Storage class:** cost/availability/access pattern option for an object.
- **Versioning:** preserves multiple versions of objects.
- **Replication:** asynchronously copies eligible objects to another bucket.
- **Lifecycle:** automatically transitions or expires objects based on rules.

S3 is designed for very high durability (commonly described as **11 nines, 99.999999999%**), but durability is not the same as availability, and it does not replace access controls, versioning, replication, or tested recovery.

## 3. Create a bucket and upload an object — hands-on
1. Open AWS Console → S3 → Create bucket.
2. Choose a globally unique bucket name and Region.
3. Keep **Block Public Access** enabled unless public access is explicitly required and reviewed.
4. Leave Object Ownership as **Bucket owner enforced** for most new setups (ACLs disabled).
5. Enable versioning if you need recovery from accidental overwrites/deletes.
6. Choose encryption settings (SSE-S3 is the standard default for many new buckets; SSE-KMS can be chosen for additional key control).
7. Create the bucket → open it → Upload → Add files → Upload.
8. Open the object to inspect properties. Keep the bucket private by default.

## 4. Storage classes
| Storage class | Typical use | Access / cost idea |
|---|---|---|
| **S3 Standard** | Frequently accessed active data | Fast access; higher storage price than archive tiers |
| **S3 Intelligent-Tiering** | Unknown/changing access patterns | Automatically moves eligible objects between tiers; monitoring/automation charges may apply |
| **S3 Standard-IA** | Infrequently accessed data needing quick access | Lower storage cost; retrieval charges and minimum-duration/minimum-size rules may apply |
| **S3 One Zone-IA** | Infrequent access where single-AZ resilience is acceptable | Lower cost, but data is stored in one AZ; not ideal as the only copy of critical data |
| **S3 Glacier Instant Retrieval** | Archive data needing millisecond access | Lower storage cost, retrieval charges and minimum-duration rules |
| **S3 Glacier Flexible Retrieval** | Archive with minutes-to-hours retrieval options | Lower storage cost; retrieval time depends on chosen option |
| **S3 Glacier Deep Archive** | Long-term, rarely accessed archive | Lowest storage cost among these common classes; retrieval can take many hours |
| **S3 Express One Zone** | Very latency-sensitive, high-request workloads | Specialized high-performance class with different architecture and cost |

**Default class:** S3 Standard is the common default when no other class is specified. “Last storage class” is not a formal term; Deep Archive is a common choice for long-term, rarely accessed data.

Always check retrieval charges, minimum storage duration, minimum billable object size, Region availability, and current pricing before choosing a class.

Official guide: https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html

## 5. Versioning
Versioning stores multiple versions of an object under the same key. It helps recover from accidental overwrites and deletes.

### Hands-on
1. S3 → Buckets → select bucket → Properties.
2. Find Bucket Versioning → Edit → Enable → Save.
3. Upload `report.txt`, then upload a modified `report.txt`.
4. Turn on **Show versions** in the object list to view versions.
5. Delete the object and inspect versions. With versioning enabled, a normal delete typically creates a delete marker rather than immediately permanently deleting prior versions.

### Best practices
- Enable versioning for important data.
- Use lifecycle rules to expire noncurrent versions when appropriate.
- Restrict `s3:DeleteObjectVersion` permissions.
- Versioning is not a substitute for cross-account or cross-Region recovery: a privileged attacker may delete versions unless protections are designed carefully.

## 6. Cross-Region Replication (CRR) and Same-Region Replication (SRR)
Replication asynchronously copies eligible objects from a source bucket to a destination bucket.

- **CRR:** destination bucket is in a different AWS Region.
- **SRR:** destination bucket is in the same Region.

### Why use replication?
- Disaster recovery and geographic separation (CRR)
- Compliance or data residency design
- Separate production and analytics/security accounts
- Reduce the distance to a copy of data, when the application is designed to use that copy
- Log aggregation or copies for different teams (often SRR)

**Correction:** CRR does not automatically reduce user latency. It creates a replicated copy in another Region; your application, DNS, CDN, or routing strategy must direct users to the appropriate copy. For global content delivery, CloudFront is often a better first solution. Replication is asynchronous, so it is not necessarily an immediate backup or instant failover.

### Hands-on replication
1. Create source and destination buckets. For CRR, choose different Regions; for SRR, choose the same Region.
2. Enable versioning on **both** buckets.
3. Open the source bucket → Management → Replication rules → Create replication rule.
4. Name the rule (e.g. `replicate-prod-objects`).
5. Choose all objects or a prefix/tag filter.
6. Select destination bucket (same account or another account, as appropriate).
7. Create/select the IAM role that permits S3 replication.
8. Review encryption, ownership, destination account permissions, and optional replication metrics/notifications.
9. Save the rule and upload a **new** test object to the source.
10. Confirm that the replicated object appears at the destination.

### Important replication details
- Replication is not retroactive for all existing objects by default. Use **S3 Batch Replication** to replicate eligible existing objects.
- Delete marker replication behavior depends on rule settings; permanent deletion of a specific version is not replicated in the same way as ordinary object changes. Review the current S3 replication behavior for your design.
- Replication requires versioning on both buckets.
- Cross-account replication needs destination bucket permissions and a suitable role; ownership settings should be designed intentionally.
- SSE-KMS-encrypted objects need the required KMS permissions and compatible key configuration.
- Replication is asynchronous; monitor replication status/metrics if recovery objectives require it.

## 7. Pre-signed URLs
A pre-signed URL grants time-limited access to a specific S3 object without making the bucket public. It uses credentials/permissions of the signer and expires at the configured time or when the signing credentials expire/revoke.

### Console hands-on
1. Open the bucket → select object → Actions → Share with a pre-signed URL (console wording can vary).
2. Choose an allowed expiry duration and generate/copy the URL.
3. Test it in a private/incognito window while it is valid.
4. Wait for expiry and confirm access is denied.

**Expiry correction:** Do not memorize “minimum 1 minute, maximum 720 minutes” as a universal S3 rule. Console limits and SDK/CLI limits differ. With AWS CLI SigV4, presigned URLs can be configured for up to 7 days, but temporary role credentials may expire sooner. Follow the interface/API limits you are using.

### Security
- Treat the URL like a temporary password: anyone who has it can use it within its permissions and lifetime.
- Use short expirations, HTTPS, least-privilege signer permissions, and avoid putting URLs in public logs/chats.
- A pre-signed URL does not override bucket policy, explicit denies, or KMS permissions.
- For upload, generate a pre-signed URL for the exact object key and method intended.

## 8. Lifecycle rules
Lifecycle rules automatically transition objects to cheaper storage classes or expire them after a defined age.

### Example policy
- Current objects: transition to Standard-IA after 30 days if access patterns justify it.
- Older archive data: transition to Glacier Flexible Retrieval after a longer interval.
- Long-term compliance data: consider Deep Archive after verifying retrieval needs.
- Noncurrent versions: expire after a defined retention period, subject to legal/compliance requirements.

These are examples, not universal recommendations. Some transitions have minimum storage-duration charges and retrieval costs. For objects that must remain instantly accessible, do not transition them to archive tiers without confirming retrieval expectations.

### Hands-on
1. Bucket → Management → Lifecycle rules → Create lifecycle rule.
2. Enter a rule name.
3. Apply to all objects or a carefully scoped prefix/tag filter.
4. Select actions: transition current versions, transition noncurrent versions, expire current objects, or expire noncurrent versions.
5. Set days and storage classes.
6. Review all actions carefully and create the rule.
7. Test with a non-production prefix/bucket and monitor object properties.

**Caution:** Lifecycle expiration can permanently remove data (depending on versioning and rule type). Review retention/legal-hold requirements before enabling expiration.

## 9. Static website hosting — hands-on
S3 can host static HTML, CSS, JavaScript, images, and other static assets. It does not run server-side application code such as PHP, Python, or Node.js.

### Basic S3 website endpoint lab
1. Create a dedicated test bucket.
2. Upload `index.html` and optionally `error.html`.
3. Open bucket → Properties → Static website hosting → Edit → Enable.
4. Choose **Host a static website**; set `index.html` as the index document and `error.html` as the error document if used.
5. Save and copy the website endpoint.
6. To use the direct S3 website endpoint, objects must be publicly readable, which means carefully configuring public access and a bucket policy. Do this only for a non-sensitive demo and understand that website endpoints do not provide HTTPS directly.
7. Test the endpoint.

### Production recommendation
For production static sites, prefer **CloudFront in front of a private S3 bucket using Origin Access Control (OAC)**. Keep S3 Block Public Access enabled and allow CloudFront to read the objects. This provides HTTPS, caching, custom domains, and better access control. A static S3 website endpoint itself is not a private HTTPS origin for OAC; configure CloudFront with the appropriate S3 REST origin.

Example `index.html`:
```html
<!doctype html>
<html>
<head><meta charset="utf-8"><title>S3 Demo</title></head>
<body><h1>Hello from Amazon S3</h1><p>Static website demo.</p></body>
</html>
```

## 10. Event notifications
S3 can send notifications when events occur, such as object creation or deletion, to supported destinations including:
- Amazon SNS
- Amazon SQS
- AWS Lambda
- EventBridge

### Example: upload event → Lambda
1. Create a test bucket and a Lambda function.
2. Ensure the Lambda execution role has only the permissions it needs.
3. In S3 → bucket → Properties → Event notifications → Create event notification.
4. Select event types such as object creation and optionally filter by prefix/suffix (e.g. prefix `uploads/`, suffix `.jpg`).
5. Choose Lambda as destination and allow S3 invocation permission through the console flow.
6. Upload a test object matching the filter and inspect Lambda logs in CloudWatch.

Avoid recursive loops: if a Lambda triggered by an S3 upload writes back into the same triggering prefix, it can invoke itself repeatedly. Use separate input/output prefixes or buckets and configure least-privilege permissions.

S3 notifications are at-least-once and may be duplicated or arrive out of order; design consumers to be idempotent. Use EventBridge for richer event-routing/filtering patterns.

## 11. Multipart upload
Multipart upload splits a large object into parts, uploads parts independently, and combines them when complete. It improves throughput and allows retrying failed parts without restarting the entire upload.

Use it for large files and resumable transfers. AWS CLI and SDKs can manage multipart upload automatically.

Example AWS CLI upload:
```bash
aws s3 cp ./large-video.mp4 s3://YOUR-BUCKET/videos/large-video.mp4
```
The CLI may automatically use multipart upload depending on file size/configuration. For lower-level control, use the multipart APIs: CreateMultipartUpload → UploadPart (each part has a part number) → CompleteMultipartUpload. If the upload fails, abort incomplete uploads when appropriate.

**Cost tip:** Configure a lifecycle rule to abort incomplete multipart uploads after a suitable number of days. Otherwise, uploaded parts may continue to incur storage charges.

## 12. S3 Transfer Acceleration
Transfer Acceleration uses AWS edge locations and the AWS network to accelerate long-distance uploads to an S3 bucket. It is most useful when clients are geographically far from the bucket and the network path is a bottleneck.

### Hands-on
1. Bucket → Properties → Transfer acceleration → Enable.
2. Wait for the setting to become active.
3. Use the acceleration endpoint (often `bucket-name.s3-accelerate.amazonaws.com`) with a supported SDK/CLI configuration.
4. Compare transfer time with and without acceleration using a representative test file and client location.

It is a paid feature. Performance gains vary; test before enabling broadly. It is primarily for transfer acceleration, not a general-purpose replacement for CloudFront or replication.

## 13. S3 Batch Operations
Batch Operations runs a supported action across a large list of objects, using an object manifest.

Examples include copying objects, invoking a Lambda function, restoring archived objects, replacing tags, or applying supported access-control actions (available operations can change).

Hands-on outline:
1. Prepare/select a manifest of target objects (for example, an S3 Inventory report or CSV manifest).
2. S3 → Batch Operations → Create job.
3. Choose the operation and manifest.
4. Configure permissions and a completion report.
5. Review cost and scope carefully, then run on a test subset before a large job.

Batch Operations is for bulk object actions; it is not simply a way to upload multiple files (ordinary console/CLI uploads can already upload multiple files).

## 14. Security best practices
- Keep **Block Public Access enabled** unless a reviewed use case requires public access.
- Prefer **Bucket owner enforced** object ownership and disabled ACLs for most new buckets.
- Apply least-privilege IAM policies and bucket policies; avoid broad `s3:*` access.
- Use SSE-S3 or SSE-KMS encryption as required; restrict KMS key usage.
- Use HTTPS/TLS for data in transit.
- Enable versioning for important objects; consider Object Lock for supported compliance/WORM requirements.
- Enable CloudTrail data events for sensitive buckets when audit requirements justify the cost; use server access logging or CloudTrail as appropriate.
- Use access logs/inventory/metrics where useful, and alert on suspicious access or policy changes.
- Avoid hard-coded AWS access keys; use IAM roles and temporary credentials.
- Use VPC endpoints for private access from supported VPC workloads where appropriate.
- Test recovery from versioning, replication, and backups.
- Replication alone is not a backup: accidental or malicious changes may also replicate depending on configuration.
- Never store secrets in public objects or expose pre-signed URLs in logs.

## 15. Common misconceptions corrected
- **Bucket = folder:** approximately for beginners, but technically S3 stores objects with keys and folder-like prefixes.
- **CRR reduces latency:** not automatically; use CloudFront or application routing for global delivery.
- **CRR shares data between accounts:** it can replicate across accounts if configured, but cross-account sharing can also use bucket policies/IAM, access points, or other controlled methods. Replication is not the only way to share.
- **Transfer Acceleration transfers any bucket-to-bucket copy faster:** its main purpose is accelerating client uploads/download paths to/from an S3 bucket; measure before using it.
- **S3 is 99.999999999% available:** that number describes durability, not availability.
- **5 GB free:** do not rely on this as a universal current allowance. Free-tier eligibility and quotas depend on account type, signup date, Region, and current AWS terms.
- **10,000 buckets:** bucket quotas can change and may differ by account; consult current S3 bucket quota documentation rather than treating one number as universal.
- **Companies using S3:** many organizations use S3, but avoid claiming a specific company uses it for a particular workload without a reliable source.

## 16. Interview questions
1. **How do you store data in AWS?** For object storage, use Amazon S3.
2. **What is S3?** Amazon Simple Storage Service, an object storage service.
3. **What can S3 store?** Images, video, audio, documents, backups, logs, datasets, and static assets.
4. **What is a bucket?** A container for objects, with a globally unique bucket name within an AWS partition.
5. **How do you reduce S3 storage costs?** Choose suitable storage classes and lifecycle rules; clean up noncurrent versions and incomplete multipart uploads where safe.
6. **How do you securely share an object temporarily?** Use a short-lived pre-signed URL.
7. **What is CRR?** Asynchronous replication to a bucket in another Region.
8. **What is SRR?** Asynchronous replication to another bucket in the same Region.
9. **How do you reduce global content latency?** Often CloudFront; replication alone does not route users to the nearest copy.
10. **How do you transfer a large file reliably?** Multipart upload; consider Transfer Acceleration for long-distance client transfers after testing.
11. **How do you operate on millions of objects?** S3 Batch Operations with a manifest.
12. **What is the common default storage class?** S3 Standard, when no other class is specified.
13. **What is the durable design principle?** Durability is extremely high, but still design access controls, recovery, and tested backups.
14. **How do you host a static website?** S3 static website hosting for a demo; CloudFront + private S3/OAC is usually preferred for production.
15. **What is versioning?** Keeps multiple versions of an object key and helps recover from overwrites/deletes.
16. **What does lifecycle management do?** Transitions or expires objects according to rules.
17. **What is an S3 event notification?** An event-driven trigger for object actions to supported services.
18. **Does replication copy existing objects automatically?** Not necessarily; use S3 Batch Replication for eligible existing objects.

## 17. Quick memory lines
- S3 = object storage.
- Bucket = container; object = data + metadata + key.
- Versioning = recover older versions.
- CRR = another Region; SRR = same Region.
- Pre-signed URL = temporary access to one object.
- Lifecycle = move to cheaper storage or expire according to policy.
- CloudFront = global content delivery; replication = asynchronous copy.
- Multipart upload = parts, retries, assemble.
- Transfer Acceleration = paid edge-assisted transfer path.
- Batch Operations = bulk actions across many objects.
- Block Public Access + least privilege = safe default.

## Official AWS references
- Storage classes: https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html
- Replication: https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html
- Pre-signed URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- Static website hosting: https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html
- Versioning: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
- Lifecycle: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- Event notifications: https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventNotifications.html
- Multipart upload: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- Transfer Acceleration: https://docs.aws.amazon.com/AmazonS3/latest/userguide/transfer-acceleration.html
- Batch Operations: https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops.html
- Security best practices: https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html
