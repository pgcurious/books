# Chapter 15: When Object Storage Shines

> **"The right tool for the right job makes everything easier."**

## The Sweet Spot

After 18 years, S3 has proven itself in specific scenarios. Let's explore where object storage excels.

## Use Case 1: Static Website Hosting

### Why It's Perfect

**Static websites** are pure content delivery:

```
What you need:
  ✓ Store HTML, CSS, JS, images
  ✓ Serve files over HTTP
  ✓ Scale to millions of users
  ✓ High availability
  ✓ Low cost

What you don't need:
  ✗ Server-side processing
  ✗ Databases
  ✗ Sessions
  ✗ Dynamic content generation
```

**S3 provides exactly this**:

```
S3 + CloudFront:
  Storage: $0.023/GB
  Transfer (CloudFront): $0.085/GB
  Requests: $0.0004 per 1000

Traditional web hosting:
  Server: $50/month (small)
  Can't scale to sudden traffic
  Single point of failure

S3: Pay only for what you use, infinite scale
```

### Real Example

**Personal blog**:

```
Content:
  - 100 HTML pages
  - 50 images
  - CSS/JS files
  Total: 500 MB

Traffic:
  - 10,000 visitors/month
  - 2 pages per visit
  - 20,000 requests/month
  - 10 GB transfer/month

Cost:
  Storage: $0.023 × 0.5 GB = $0.01
  Transfer: $0.085 × 10 GB = $0.85
  Requests: $0.0004 × 20 = $0.01
  Total: $0.87/month

Traditional hosting: $10-50/month
Savings: 10-50x cheaper!
```

**High-traffic launch**:

```
Product launch:
  Day 1: 1 million visitors
  Traffic spike: 100x normal

S3:
  Automatically scales
  Same latency
  Cost: $87 for the day (pay per use)

Traditional hosting:
  Server crashes (not provisioned for spike)
  Or: Over-provisioned (waste money 99% of time)
```

## Use Case 2: Data Lakes

### What Is a Data Lake?

**Store all data in raw form for later analysis**:

```
Data sources:
  - Application logs
  - IoT sensor data
  - User clickstreams
  - Transaction records
  - Social media feeds

Volume: Petabytes to exabytes

Requirements:
  ✓ Store cheaply (raw data is huge)
  ✓ Keep forever (or for years)
  ✓ Query flexibly (unknown questions)
  ✓ Scale infinitely
```

### Why S3 Is Perfect

**Cost at scale**:

```
1 PB data lake:

S3 (with lifecycle):
  First 30 days: STANDARD ($0.023/GB) = $23,000
  31-90 days: IA ($0.0125/GB) = $12,500 × 2 = $25,000
  90+ days: GLACIER ($0.004/GB) = $4,000 × 9 = $36,000
  Average: $8,400/month

Traditional storage:
  SAN: $2,000,000 upfront + $50,000/month
  NAS: $500,000 upfront + $20,000/month

S3: No upfront, $8k/month
```

**Integration with analytics**:

```
S3 → Athena (SQL queries)
S3 → Spark (big data processing)
S3 → Redshift Spectrum (data warehouse)
S3 → SageMaker (machine learning)
S3 → EMR (Hadoop/Spark)

All read directly from S3!
No ETL needed.
```

### Real Example

**E-commerce analytics**:

```
Data:
  - 10 billion events/day
  - Event size: ~1 KB
  - Daily data: 10 TB
  - Retention: 7 years

Storage:
  Year 1: 3.65 PB (with lifecycle: ~$10k/month)
  Year 7: 25.5 PB (with lifecycle: ~$20k/month)

Queries:
  - Athena: Pay per query ($5/TB scanned)
  - Query 1% of data: $1.25/query
  - 100 queries/month: $125

Total: ~$10k-$20k/month for petabytes!

Alternative (traditional data warehouse):
  $100k-$500k/month (10-50x more expensive)
```

## Use Case 3: Backup and Archive

### Why Backup Needs Object Storage

**Characteristics of backups**:

```
Write once: Initial backup
Read rarely: Only on restore
Keep long: Years to decades
Volume: Terabytes to petabytes
Criticality: Must never lose

Perfect match for S3!
```

### Backup Strategy

**Tiered backup**:

```
Daily backups:
  Day 0: Upload to STANDARD
  Day 7: Transition to STANDARD_IA
  Day 30: Transition to GLACIER
  Day 365: Transition to DEEP_ARCHIVE
  Year 7: Delete (retention policy)

Cost for 10 TB backups per month:
  Month 1: 10 TB × $0.023 = $230
  Month 2: 20 TB (10 TB STANDARD, 10 TB IA) = $347
  Month 3: 30 TB (distributed) = $420
  Month 12: 120 TB (mostly GLACIER) = $840
  Year 7: 840 TB (all DEEP_ARCHIVE) = $832/month

Average: ~$500/month for 7 years of backups
```

**Disaster recovery**:

```
Backup strategy:
  1. Primary: Daily backups to S3 (same region)
  2. Secondary: Cross-region replication
  3. Tertiary: Periodic export to separate account

Scenarios:
  Accidental delete: Restore from primary (minutes)
  Regional outage: Failover to secondary region (hours)
  Account compromise: Restore from tertiary (days)

All data safe!
```

### Real Example

**Database backups**:

```
PostgreSQL database: 500 GB

Daily backup:
  Full backup: 500 GB
  Upload to S3: 30 minutes (multipart)
  Cost: $11.50/month (STANDARD)

After 7 days → STANDARD_IA:
  Cost: $6.25/month

After 30 days → GLACIER:
  Cost: $2/month

Keep 90 days of daily backups:
  7 days STANDARD: $80.50
  23 days IA: $143.75
  60 days GLACIER: $120
  Total: $344/month for complete backup history

Restore:
  Recent (7 days): Instant
  Medium (30 days): Minutes (from IA)
  Old (90 days): Hours (from GLACIER)
```

## Use Case 4: Media Storage and Streaming

### Why Media Loves S3

**Media characteristics**:

```
Large files: 100 MB to 10 GB per file
Immutable: Create once, view many times
Global: Users worldwide
Scalable: Viral content needs instant scale
```

**S3 + CloudFront = Perfect combo**:

```
Origin: S3 (durable, cheap storage)
Distribution: CloudFront (global CDN)

User requests video:
  1. CloudFront edge checks cache
  2. Cache miss: Fetch from S3 (once)
  3. Cache hit: Serve from edge (millions of times)

Benefits:
  ✓ Low latency (edge is close to user)
  ✓ High throughput (edge has bandwidth)
  ✓ Cost-effective (S3 → CloudFront once, then cached)
```

### Real Example

**Video streaming platform**:

```
Content:
  - 10,000 videos
  - Average size: 2 GB
  - Total: 20 TB

Storage:
  20 TB × $0.023 = $460/month

Delivery (1 million views/month):
  Without CDN:
    20 TB × 1M / 10K = 2,000 TB transfer
    S3 transfer: $180,000/month (!!)

  With CloudFront:
    S3 → CloudFront: 20 TB × $0.02 = $400
    CloudFront → Users: 2,000 TB × $0.085 = $170,000
    Total: $170,400/month

Still expensive, but necessary for scale.
```

**Adaptive bitrate streaming**:

```
Store multiple versions:
  video-480p.m3u8 (500 MB)
  video-720p.m3u8 (1.2 GB)
  video-1080p.m3u8 (2.5 GB)

Player selects based on bandwidth:
  Slow connection → 480p
  Fast connection → 1080p

S3:
  Stores all versions
  CloudFront serves appropriate version
  Seamless quality adaptation
```

## Use Case 5: Application Assets

### Static Assets

**What are static assets?**

```
Images: Logos, icons, photos
Fonts: TTF, WOFF, WOFF2
CSS: Stylesheets
JavaScript: Libraries, bundles
Documents: PDFs, downloads
```

**Why S3?**

```
Traditional approach:
  Bundle assets with application
  Deploy to servers
  Serve from app server

Problems:
  ✗ App server does file serving (waste of resources)
  ✗ All assets copied to all servers (redundant)
  ✗ Update asset = redeploy app (coupling)
  ✗ No CDN (slow for distant users)

S3 approach:
  Upload assets to S3
  Serve via CloudFront
  App references CDN URLs

Benefits:
  ✓ Offload serving from app (app focuses on logic)
  ✓ Single source of truth (S3)
  ✓ Update asset without redeploying app
  ✓ Global CDN (fast everywhere)
```

### User Uploads

**User-generated content**:

```
Examples:
  - Profile pictures
  - Document uploads
  - Video uploads
  - File sharing

Requirements:
  ✓ Durable (can't lose user data)
  ✓ Scalable (millions of users)
  ✓ Access control (private vs public)
  ✓ Virus scanning
  ✓ Thumbnail generation

S3 provides all of this:
  - Durability: 11 nines
  - Scale: Unlimited
  - ACLs: Per-object permissions
  - Lambda: Trigger virus scan on upload
  - Lambda: Generate thumbnails on upload
```

**Architecture**:

```
User uploads photo:
  1. App generates pre-signed URL
  2. User uploads directly to S3 (no app server)
  3. S3 triggers Lambda (on upload event)
  4. Lambda: Virus scan, thumbnail generation
  5. Lambda: Update database (photo metadata)
  6. App: Display photo from CloudFront

App server: Not involved in file upload/serving!
  → Scales better (stateless)
```

## Use Case 6: Big Data and Analytics

### Log Aggregation

**Centralized logging**:

```
Sources:
  - 1000 application servers
  - Each generates 10 GB logs/day
  - Total: 10 TB/day

S3 logging pipeline:
  1. Servers stream logs to S3 (via Kinesis Firehose)
  2. S3 stores in partitioned structure:
     s3://logs/app/year=2024/month=01/day=15/hour=10/
  3. Athena queries logs directly
  4. Lifecycle: Delete after 90 days

Cost:
  Storage: 10 TB/day × 90 days = 900 TB × $0.023 = $20,700/month
  Queries: $5/TB scanned (pay per query)

Alternative (Elasticsearch):
  900 TB Elasticsearch: $100,000+/month
```

**Query example**:

```sql
SELECT
  user_id,
  COUNT(*) as error_count
FROM logs
WHERE
  year = 2024
  AND month = 01
  AND level = 'ERROR'
GROUP BY user_id
ORDER BY error_count DESC
LIMIT 100;

Athena: Scans only relevant partitions
Cost: ~$5 (scanning 1 TB)
Time: 30 seconds
```

### ML Training Data

**Machine learning datasets**:

```
Training data:
  - Images: 10 million × 2 MB = 20 TB
  - Labels: 10 million × 1 KB = 10 GB
  - Total: ~20 TB

S3 for ML:
  Storage: $460/month
  SageMaker reads directly from S3
  No data movement needed

Benefits:
  ✓ Version datasets (S3 versioning)
  ✓ Share across teams (IAM policies)
  ✓ Archive old datasets (lifecycle)
  ✓ Integration with SageMaker, TensorFlow, PyTorch
```

## Use Case 7: Software Distribution

### Release Artifacts

**Distributing software**:

```
Examples:
  - Docker images
  - npm packages
  - OS images (ISOs)
  - Mobile apps (APKs, IPAs)
  - Game assets

Requirements:
  ✓ High availability (downloads must succeed)
  ✓ Global distribution (users worldwide)
  ✓ Versioning (multiple releases)
  ✓ Bandwidth (large files, many downloads)

S3 + CloudFront:
  Storage: $0.023/GB
  Distribution: $0.085/GB (CloudFront)
  Availability: 99.99%
```

### Real Example

**Docker Hub alternative**:

```
Docker images:
  - 1000 images
  - Average size: 500 MB
  - Total: 500 GB
  - Downloads: 10,000/month

S3 storage: $11.50/month
CloudFront transfer: 5 TB × $0.085 = $425/month
Total: ~$437/month

Serves 10,000 developers reliably!
```

## Common Patterns

### Pattern 1: S3 as a Data Bus

```
Architecture:
  Producer → S3 ← Consumer

Example:
  Video encoder → S3 → Streaming app
  ETL job → S3 → Analytics
  Microservice A → S3 → Microservice B

Benefits:
  ✓ Decoupling (services don't talk directly)
  ✓ Buffering (S3 holds data)
  ✓ Replay (consumers can re-read)
```

### Pattern 2: S3 Event-Driven Processing

```
S3 event triggers:
  - Lambda function
  - SQS queue
  - SNS notification

Example: Image pipeline
  1. Upload image to S3
  2. S3 triggers Lambda
  3. Lambda: Resize, optimize, watermark
  4. Lambda: Save processed image back to S3
  5. Lambda: Update database

All serverless, scales automatically!
```

### Pattern 3: S3 as Source of Truth

```
Architecture:
  S3 (source) → Replicate to working stores

Example:
  S3 (raw data) → Load to:
    - Redshift (analytics)
    - Elasticsearch (search)
    - DynamoDB (app queries)

If working stores corrupted:
  - Delete
  - Reload from S3 (source of truth)
```

## Key Takeaways

- **Static websites** - Cheap, scalable, zero maintenance
- **Data lakes** - Store everything, query later
- **Backups** - Durable, cheap, tiered storage
- **Media** - Large files, global delivery via CDN
- **User uploads** - Offload from app servers
- **Analytics** - Direct SQL queries on massive datasets
- **Software distribution** - High availability, global reach
- **Event-driven** - Trigger processing on upload

## What's Next?

We've seen where S3 shines. But what about where it struggles? In the next chapter, we'll explore the anti-patterns and when NOT to use object storage.

---

**[← Previous: Performance and Optimization](./14-performance-optimization.md) | [Back to Contents](./README.md) | [Next: When Object Storage Struggles →](./16-when-it-struggles.md)**
