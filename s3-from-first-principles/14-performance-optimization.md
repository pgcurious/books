# Chapter 14: Performance and Optimization

> **"Speed is a feature. And at scale, every millisecond counts."**

## The Performance Challenge

Your application uploads a 5 GB video file:

```
Using single PUT:
  Upload speed: 100 Mbps
  Time: 5 GB / (100 Mbps / 8) = 400 seconds
  = 6.7 minutes

Network hiccup at 99% uploaded?
  → Retry entire 5 GB upload
  → Another 6.7 minutes

Total: 13+ minutes for one file!
```

**There must be a better way.**

## Multipart Upload

### The Problem with Single PUT

**Limitations**:

```
1. Size limit: 5 GB maximum
   → Can't upload larger files!

2. No resume: Network failure = start over
   → Wastes time and bandwidth

3. Slow: Sequential upload
   → Can't parallelize

4. Memory: Must buffer entire file
   → 5 GB in RAM?
```

### How Multipart Upload Works

**Split file into parts, upload in parallel**:

```
File: video.mp4 (5 GB)

Split into parts:
  Part 1: 100 MB (bytes 0-104857599)
  Part 2: 100 MB (bytes 104857600-209715199)
  Part 3: 100 MB (bytes 209715200-314572799)
  ...
  Part 50: 100 MB (bytes 5000000000-5242879999)

Upload:
  ┌────────┐
  │ Part 1 ├──┐
  ├────────┤  │
  │ Part 2 ├──┼─→ Upload in parallel
  ├────────┤  │
  │ Part 3 ├──┘
  ...

Each part:
  - Uploads independently
  - Can retry individual part
  - Multiple parts simultaneously
```

### Multipart Upload Process

**Step 1: Initiate**

```
POST /bucket/video.mp4?uploads

Response:
  UploadId: "abc123xyz"

This UploadId tracks the upload session.
```

**Step 2: Upload parts**

```
PUT /bucket/video.mp4?partNumber=1&uploadId=abc123xyz
  Body: <Part 1 data (100 MB)>
  → ETag: "etag1"

PUT /bucket/video.mp4?partNumber=2&uploadId=abc123xyz
  Body: <Part 2 data (100 MB)>
  → ETag: "etag2"

...

PUT /bucket/video.mp4?partNumber=50&uploadId=abc123xyz
  Body: <Part 50 data (100 MB)>
  → ETag: "etag50"

Upload these in parallel!
```

**Step 3: Complete**

```
POST /bucket/video.mp4?uploadId=abc123xyz
  Body:
  <CompleteMultipartUpload>
    <Part>
      <PartNumber>1</PartNumber>
      <ETag>etag1</ETag>
    </Part>
    <Part>
      <PartNumber>2</PartNumber>
      <ETag>etag2</ETag>
    </Part>
    ...
    <Part>
      <PartNumber>50</PartNumber>
      <ETag>etag50</ETag>
    </Part>
  </CompleteMultipartUpload>

S3 assembles parts into final object.
→ Success!
```

### Performance Improvements

**Parallelization**:

```
Single-threaded:
  5 GB / 100 Mbps = 400 seconds

10 parallel parts (10 × 100 Mbps = 1 Gbps):
  5 GB / 1 Gbps = 40 seconds

10x faster!
```

**Resumability**:

```
Scenario:
  Parts 1-30: Uploaded ✓
  Parts 31-50: Not started
  Network fails

Resume:
  Parts 1-30: Already uploaded (skip)
  Parts 31-50: Upload now
  Complete multipart upload

No need to re-upload 3 GB!
```

**Memory efficiency**:

```
Single PUT:
  Must buffer entire 5 GB in memory

Multipart:
  Buffer one part at a time (100 MB)
  Upload, discard, next part
  Peak memory: 100 MB

50x less memory!
```

### Multipart Best Practices

**Part size**:

```
Minimum: 5 MB (except last part)
Maximum: 5 GB
Recommended: 100 MB - 500 MB

Too small: Too many parts (overhead)
Too large: Not enough parallelism

Sweet spot: ~100 MB for most use cases
```

**Parallelism**:

```
Factors to consider:
  - Network bandwidth
  - CPU (encryption overhead)
  - Disk I/O

Typical:
  Home network (100 Mbps): 2-4 parallel
  Data center (10 Gbps): 10-20 parallel
  AWS EC2 → S3 (25 Gbps): 20-50 parallel

Don't over-parallelize: Diminishing returns
```

**Error handling**:

```
For each part:
  1. Upload
  2. If failure:
     a. Retry (exponential backoff)
     b. Max 3 retries
     c. If still fails: Abort upload

Partial upload cleanup:
  - Set lifecycle policy to delete incomplete uploads after 7 days
  - Prevents abandoned parts consuming storage
```

## Transfer Acceleration

### The Distance Problem

**Network latency increases with distance**:

```
User in Tokyo → S3 us-east-1 (Virginia)
  Latency: ~150ms round-trip
  Throughput: Limited by latency

Upload 1 GB:
  TCP window size: 64 KB
  Effective throughput: 64 KB / 150ms = 426 KB/s = 3.4 Mbps

Actual network: 100 Mbps available
Utilization: 3.4% (!!)

Problem: TCP slow-start + latency
```

### How Transfer Acceleration Works

**Use CloudFront edge locations as upload proxies**:

```
Without acceleration:
  Tokyo → [150ms] → Virginia S3
  Effective throughput: 3.4 Mbps

With acceleration:
  Tokyo → [5ms] → Tokyo Edge → [AWS backbone] → Virginia S3
  Effective throughput: 90+ Mbps

26x faster!
```

**Architecture**:

```
┌─────────┐
│  User   │
│ (Tokyo) │
└────┬────┘
     │ 5ms (local)
┌────▼────────┐
│Tokyo Edge   │ ← CloudFront edge location
│  Location   │    - Accepts upload
└────┬────────┘    - Optimized TCP
     │               - AWS private backbone
     │ Fast (AWS network)
┌────▼────────┐
│   S3        │ ← us-east-1
│ (Virginia)  │
└─────────────┘
```

**How it helps**:

```
1. Edge location is close (low latency)
   → TCP performs well

2. AWS backbone is optimized:
   → Higher bandwidth
   → Lower packet loss
   → Better routing

3. Concurrent uploads:
   → Edge location buffers
   → Smooths variations
```

### Enabling Transfer Acceleration

**Enable on bucket**:

```
PUT /bucket?accelerate
  <AccelerateConfiguration>
    <Status>Enabled</Status>
  </AccelerateConfiguration>
```

**Use accelerated endpoint**:

```
Standard endpoint:
  bucket.s3.amazonaws.com

Accelerated endpoint:
  bucket.s3-accelerate.amazonaws.com

Just change the hostname!
```

**Cost**:

```
Standard transfer: Included (up to 1 GB/month), then $0.09/GB
Accelerated transfer: Additional $0.04-0.08/GB (depends on regions)

When worth it?
  - Long distance (inter-continental)
  - Large files
  - Frequent uploads

When not worth it?
  - Same region
  - Small files
  - Infrequent uploads
```

## Range Reads (Byte-Range Fetches)

### The Partial Read Problem

**Scenario**: Video player wants to seek to middle of 1 GB file.

```
Without range reads:
  Download entire 1 GB → Seek to 500 MB point → Play
  Wait time: Minutes
  Waste: 500 MB unused data

With range reads:
  Request bytes 500000000-501000000 (1 MB at seek point)
  Wait time: Seconds
  Waste: None
```

### How Range Reads Work

**Request specific byte ranges**:

```
GET /bucket/video.mp4
  Range: bytes=500000000-501000000

Response:
  206 Partial Content
  Content-Range: bytes 500000000-501000000/1000000000
  <1 MB of data>

Only downloads requested range!
```

**Multiple ranges in one request**:

```
GET /bucket/file
  Range: bytes=0-1000,5000-6000,10000-11000

Response:
  206 Partial Content (multipart/byteranges)
  <Part 1: bytes 0-1000>
  <Part 2: bytes 5000-6000>
  <Part 3: bytes 10000-11000>
```

### Use Cases

**1. Video streaming**:

```
User seeks to 10:00 in video:
  Byte offset = 10 minutes × bitrate / 8
  Request bytes from that offset
  Start playing immediately

Benefits:
  ✓ Instant seek
  ✓ No buffering entire file
  ✓ Save bandwidth
```

**2. Parallel downloads**:

```
Download 1 GB file using 10 threads:
  Thread 1: bytes 0-100000000
  Thread 2: bytes 100000001-200000000
  ...
  Thread 10: bytes 900000001-1000000000

Combine locally.

10x faster (if bandwidth allows)!
```

**3. Resume downloads**:

```
Download 1 GB file:
  Downloaded: 400 MB
  Connection drops

Resume:
  Range: bytes=400000000-999999999
  Continue from where left off

No re-download!
```

**4. Checksum verification**:

```
Verify file integrity without downloading:
  GET bytes 0-1000 (header)
  GET bytes -1000 (footer, last 1000 bytes)
  Check magic numbers, headers
  If valid: Download rest

Fail fast on corrupted files.
```

## S3 Select and Glacier Select

### The Over-Fetching Problem

**Scenario**: Query 1 row from 10 GB CSV file.

```
Without S3 Select:
  1. Download entire 10 GB file
  2. Parse CSV locally
  3. Filter for desired row
  4. Use that row

Cost: $0.90 (data transfer out)
Time: Minutes
Waste: 9.999 GB unused
```

### How S3 Select Works

**Run SQL queries on S3 objects**:

```
SELECT * FROM s3object
  WHERE name = 'Alice'
  AND age > 30

S3:
  1. Read file from disk
  2. Parse CSV/JSON/Parquet
  3. Filter rows
  4. Return only matching rows

Download: Only results (KB vs GB)
```

**Example**:

```
POST /bucket/data.csv?select&select-type=2

Query:
  SELECT name, email
  FROM s3object s
  WHERE s.age > 30

Response:
  name,email
  Alice,alice@example.com
  Bob,bob@example.com
  ...

Downloaded: 2 KB (vs 10 GB)
Savings: 99.98%!
```

**Supported formats**:

```
CSV (with headers)
JSON (newline-delimited)
Parquet (columnar)
```

**Benefits**:

```
✓ Reduced data transfer (400x savings possible)
✓ Faster (process server-side)
✓ Cheaper (pay for data scanned, not transferred)
```

**Limitations**:

```
✗ Limited SQL (no JOINs)
✗ Single file only (no cross-file queries)
✗ Max 256 KB per record
✗ Not as flexible as Athena
```

**When to use**:

```
Use S3 Select when:
  ✓ Simple queries (filters, projections)
  ✓ Single file
  ✓ Want to avoid downloading

Use Athena when:
  ✓ Complex queries (JOINs, aggregations)
  ✓ Multiple files
  ✓ Need SQL analytics
```

## CloudFront Integration

### Caching at the Edge

**Problem**: Same file requested millions of times.

```
Without CDN:
  1 million requests → S3
  S3 serves 1 million times
  Cost: $0.40 (requests) + $90 (transfer)

With CloudFront:
  1st request: CloudFront → S3
  Next 999,999 requests: CloudFront cache
  Cost: $0.01 (CloudFront) + $0.09 (S3 to CloudFront)

Savings: ~70%
```

**Architecture**:

```
┌─────────┐      ┌──────────────┐      ┌─────────┐
│  User   │─────►│  CloudFront  │─────►│   S3    │
│         │ Fast │ Edge (cache) │ Slow │ Origin  │
└─────────┘      └──────────────┘      └─────────┘

First request: Cache miss → S3
  Latency: 100ms

Subsequent requests: Cache hit
  Latency: 10ms (10x faster!)
```

### Cache Control

**Control caching behavior**:

```
When uploading to S3:
  PUT /bucket/logo.png
    Cache-Control: public, max-age=3600

CloudFront:
  Caches for 3600 seconds (1 hour)
  After 1 hour: Revalidate with S3

For static assets:
  Cache-Control: public, max-age=31536000 (1 year)

For dynamic content:
  Cache-Control: no-cache
  (Always revalidate)
```

**Invalidation**:

```
Update a file but it's cached?

Create invalidation:
  POST /distribution/invalidation
    Paths: ["/images/*"]

CloudFront:
  Removes cached copies
  Next request: Fetch fresh from S3

Cost: First 1000 invalidations/month free
```

### Signed URLs

**Provide temporary access to private content**:

```
S3: Private bucket (no public access)

CloudFront:
  Generate signed URL with expiration
  URL: https://cdn.example.com/video.mp4?Expires=...&Signature=...

User can access for limited time (e.g., 1 hour).
After expiration: Access denied.

Use case: Paid content, time-limited access
```

## Request Rate Optimization

### Partition Key Prefixes

**Problem**: Hotspot on one partition.

```
All files start with "photos/":
  photos/001.jpg
  photos/002.jpg
  photos/003.jpg

All hash to same partition → Hotspot!
```

**Solution**: Add random prefix.

```
Hash prefix:
  a3f2/photos/001.jpg
  b8d4/photos/002.jpg
  c1a9/photos/003.jpg

Different partitions → Distributed load
```

**Naming patterns**:

```
Bad (sequential):
  uploads/20240115/001.jpg
  uploads/20240115/002.jpg
  → Same partition

Good (hash prefix):
  a3/uploads/20240115/001.jpg
  f8/uploads/20240115/002.jpg
  → Different partitions

Or use reversed date:
  51042024/uploads/001.jpg (MMDDYYYYreverseD)
  → Better distribution
```

### Connection Pooling

**Reuse connections**:

```
Bad:
  For each request:
    Open TCP connection
    TLS handshake
    Send request
    Close connection
  Overhead: 50-100ms per request

Good:
  Open connection pool (10 connections)
  Reuse for multiple requests
  Keep-alive connections
  Overhead: 0ms (amortized)
```

**Configuration**:

```python
import boto3
from botocore.config import Config

config = Config(
    max_pool_connections=50,
    retries={'max_attempts': 3}
)

s3 = boto3.client('s3', config=config)
```

### Retry Strategy

**Exponential backoff**:

```
Request fails → Retry with backoff

Attempt 1: Immediate
Attempt 2: Wait 1 second
Attempt 3: Wait 2 seconds
Attempt 4: Wait 4 seconds
Attempt 5: Wait 8 seconds
...

Prevents overwhelming S3 during issues.
```

**Which errors to retry**:

```
Retry:
  ✓ 500 Internal Server Error
  ✓ 503 Service Unavailable
  ✓ 429 Too Many Requests (rate limit)
  ✓ Network timeout

Don't retry:
  ✗ 404 Not Found (won't change)
  ✗ 403 Forbidden (permission issue)
  ✗ 400 Bad Request (client error)
```

## Monitoring and Metrics

### Key Performance Metrics

```
1. Latency (p50, p99, p99.9):
   - Time from request to first byte
   - Target: p99 < 100ms

2. Throughput:
   - Bytes/second
   - Requests/second
   - Target: Maximize within limits

3. Error rate:
   - 4xx errors (client)
   - 5xx errors (server)
   - Target: < 0.1%

4. Transfer acceleration benefit:
   - Latency improvement
   - Throughput improvement
```

### CloudWatch Metrics

```
S3 metrics:
  - AllRequests
  - 4xxErrors, 5xxErrors
  - BytesDownloaded, BytesUploaded
  - FirstByteLatency
  - TotalRequestLatency

Set alarms:
  - 5xxErrors > 1%: Alert
  - FirstByteLatency > 500ms: Investigate
```

## Key Takeaways

- **Multipart upload is essential** - For files >100 MB, 10x faster
- **Transfer acceleration helps distance** - Use for intercontinental
- **Range reads enable seeking** - Critical for video/large files
- **S3 Select reduces transfer** - Filter server-side
- **CloudFront saves cost and improves speed** - Cache at edge
- **Connection pooling essential** - Reuse connections
- **Monitor performance** - Latency, throughput, errors
- **Optimize for your workload** - Different patterns need different optimizations

## What's Next?

We've covered how to make S3 fast. But when should you use S3, and when should you choose something else? In the next chapter, we'll explore when object storage shines.

---

**[← Previous: Lifecycle and Storage Classes](./13-lifecycle-storage-classes.md) | [Back to Contents](./README.md) | [Next: When Object Storage Shines →](./15-when-it-shines.md)**
