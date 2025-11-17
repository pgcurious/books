# Chapter 10: Metadata at Scale

> **"Data is easy to scale. Metadata is where systems break."**

## The Metadata Problem

You can store trillions of objects. But can you find them?

### What is Metadata?

**For each object, we track**:

```
Object: "photos/beach.jpg" in bucket "vacation-pics"

Metadata:
  - Bucket: "vacation-pics"
  - Key: "photos/beach.jpg"
  - Size: 2,048,576 bytes
  - ETag: "a3f2b8d4c1e5..."
  - Last modified: 2024-01-15T10:30:00Z
  - Content-Type: "image/jpeg"
  - Storage class: "STANDARD"
  - Location: [server1:offset1, server4:offset2, ...]
  - Owner: user-123
  - ACL: private
  - Custom metadata: {"camera": "iPhone", ...}
```

**Size per object**: ~500 bytes to 2 KB (depending on custom metadata)

### The Scale Challenge

**For 1 trillion objects**:

```
Metadata size:
  1 trillion × 1 KB avg = 1 PB just for metadata!

Operations:
  - Lookups: Millions per second
  - Listings: Thousands per second
  - Updates: Millions per second

Requirements:
  ✓ Low latency (<10ms)
  ✓ High throughput (millions QPS)
  ✓ Highly available (99.99%+)
  ✓ Durable (don't lose metadata!)
  ✓ Consistent (eventual → strong)
```

**This is harder than storing the actual data.**

## Metadata Storage Options

### Option 1: Relational Database

**PostgreSQL / MySQL**:

```sql
CREATE TABLE objects (
  bucket VARCHAR(255),
  key VARCHAR(1024),
  size BIGINT,
  etag VARCHAR(64),
  modified TIMESTAMP,
  location TEXT,
  PRIMARY KEY (bucket, key),
  INDEX (bucket, modified),
  INDEX (bucket, key)
);
```

**Pros**:
```
✓ ACID transactions
✓ Rich query capabilities
✓ Mature and reliable
✓ Strong consistency
```

**Cons**:
```
✗ Vertical scaling limits:
  - Single node: ~1-10 billion rows max
  - Queries slow beyond that

✗ Complex sharding:
  - Manual partition by bucket?
  - Rebalancing is painful

✗ JOIN overhead:
  - We don't need JOINs
  - Paying for features we don't use

✗ Throughput limits:
  - ~50,000 queries/second per node
  - Need millions QPS
```

**Verdict**: Doesn't scale to trillions.

### Option 2: NoSQL (Key-Value Store)

**DynamoDB / Cassandra / HBase**:

```
Key: bucket + "/" + key
Value: {size, etag, modified, location, ...}

Example:
  Key: "vacation-pics/photos/beach.jpg"
  Value: {"size": 2048576, "etag": "a3f2...", ...}
```

**Pros**:
```
✓ Horizontal scaling (trillions of rows)
✓ High throughput (millions QPS)
✓ Distributed by design
✓ Simple data model
```

**Cons**:
```
✗ Eventual consistency (eventually strong)
✗ Limited query capabilities:
  - No prefix scans (in some systems)
  - Range queries limited

✗ Listing challenges:
  - List all objects in bucket?
  - Must scan all partitions

✗ Secondary indexes expensive
```

**Verdict**: Better, but listing is problematic.

### Option 3: Custom Distributed Index

**Build exactly what we need**:

```
Design:
  - Partition by hash(bucket + key)
  - Each partition: LSM-tree (like RocksDB)
  - Replicated 3x for durability
  - Optimized for our access patterns
```

**Pros**:
```
✓ Optimized for S3 workloads
✓ Efficient listings (prefix scans)
✓ Scales to trillions
✓ Full control over consistency
```

**Cons**:
```
✗ Must build and maintain ourselves
✗ Complex implementation
```

**S3's choice**: Option 3 (custom distributed index)

## The Custom Metadata Index

### Architecture

**Two-tier design**:

```
┌─────────────────────────────────┐
│     Global Index (Routing)      │  ← Maps bucket to partition
│   Tracks: Bucket → Partition    │
└─────────────────────────────────┘
              │
        ┌─────┴─────┬──────────┬──────────┐
        │           │          │          │
┌───────▼─────┐ ┌───▼────┐ ┌──▼─────┐ ┌──▼─────┐
│ Partition 1 │ │ Part 2 │ │ Part 3 │ │ Part N │  ← Stores metadata
│  (Replica)  │ │        │ │        │ │        │
└─────────────┘ └────────┘ └────────┘ └────────┘
```

**Partition structure**:

```
Each partition manages:
  - Subset of buckets/keys
  - LSM-tree storage (RocksDB-like)
  - Replicated 3x (durability)
  - Serves: GET, PUT, DELETE, LIST
```

### Partitioning Strategy

**Partition by bucket first, then key**:

```
Why bucket-first?
  - Listings are per-bucket
  - Keep bucket's metadata together
  - Efficient prefix scans

Partition function:
  partition_id = hash(bucket) % NUM_PARTITIONS

All objects in "vacation-pics" → same partition
→ List operations are local!
```

**Partition size**:

```
Target: 10-100 GB per partition

For 1 PB metadata:
  Partition size: 50 GB
  Number of partitions: 1 PB / 50 GB = 20,000 partitions

Replication factor: 3
  Total partitions: 60,000 partition replicas
  Servers: 60,000 / 3 = 20,000 servers
  (assuming 3 partitions per server)
```

### LSM-Tree Storage

**Why LSM-tree?** (Log-Structured Merge tree)

```
Characteristics:
  ✓ Write-optimized (append to log)
  ✓ Range scans efficient (sorted SSTables)
  ✓ Compaction reclaims space
  ✓ Scales to billions of keys per partition

Perfect for metadata!
```

**Write path**:

```
PUT /vacation-pics/photos/beach.jpg

1. Hash("vacation-pics") → Partition 874
2. Write to partition's LSM-tree:
   a. Append to WAL (Write-Ahead Log)
   b. Insert into MemTable (in-memory)
   c. Acknowledge success
   d. Background: Flush MemTable → SSTable (on disk)
   e. Background: Compact SSTables

Latency: <5ms (mostly in-memory)
```

**Read path**:

```
GET /vacation-pics/photos/beach.jpg

1. Hash("vacation-pics") → Partition 874
2. Query partition's LSM-tree:
   a. Check MemTable (in memory)
   b. If not found: Check SSTables (disk)
      - Use bloom filters (skip irrelevant files)
      - Binary search within SSTable
   c. Return metadata

Latency: <10ms (may require disk read)
```

## Listing at Scale

### The Listing Challenge

**List all objects in bucket**:

```
Request: GET /?list-type=2&prefix=photos/

Expected:
  - All objects starting with "photos/"
  - Sorted by key
  - Paginated (1000 at a time)

Challenges:
  - Bucket might have billions of objects
  - Listing must be fast (<100ms)
  - Results must be sorted
  - Pagination must be efficient
```

### Listing Implementation

**Step 1: Partition Locality**

```
All objects in "vacation-pics" are in same partition.

List request:
  1. Route to Partition 874
  2. Partition does local scan
  3. Return results

No cross-partition coordination needed!
```

**Step 2: Prefix Filtering**

```
Request: List with prefix="photos/2024/"

LSM-tree scan:
  1. Seek to "photos/2024/" (start key)
  2. Scan forward until key no longer matches prefix
  3. Collect up to 1000 keys
  4. Return results + continuation token

Time complexity: O(results returned)
  Not O(total objects in bucket)!

Fast even with billions of objects.
```

**Step 3: Pagination**

```
Request 1:
  GET /?list-type=2&prefix=photos/&max-keys=1000

Response:
  Keys: ["photos/a.jpg", "photos/b.jpg", ..., "photos/z999.jpg"]
  IsTruncated: true
  NextContinuationToken: "photos/z999.jpg"

Request 2:
  GET /?list-type=2&prefix=photos/&max-keys=1000&continuation-token=photos/z999.jpg

Response:
  Keys: ["photos/z1000.jpg", ..., "photos/z2000.jpg"]
  IsTruncated: true
  NextContinuationToken: "photos/z2000.jpg"

Continue until IsTruncated: false
```

**Continuation token** = last key from previous page

**Seek directly** to continuation token (no re-scanning)

### Listing Optimizations

**Optimization 1: Bloom Filters**

```
Problem: LSM-tree has multiple SSTables

Without bloom filter:
  1. Check SSTable 1: Binary search (disk I/O)
  2. Check SSTable 2: Binary search (disk I/O)
  ...
  Many disk I/Os!

With bloom filter:
  1. Check bloom filter for SSTable 1: NOT PRESENT (no disk I/O)
  2. Check bloom filter for SSTable 2: NOT PRESENT (no disk I/O)
  3. Check bloom filter for SSTable 3: MAYBE PRESENT
     → Read SSTable 3 (disk I/O)

Saves 90%+ of disk I/Os!
```

**Optimization 2: Caching**

```
Cache hot metadata in memory:
  - Recently accessed objects
  - Common prefixes
  - Directory-like listings

Hit rate: 80-90% for popular buckets
→ Most listings served from memory
```

**Optimization 3: Parallel Scans**

```
For very large listings:
  1. Split range into chunks
  2. Scan chunks in parallel
  3. Merge results

Example:
  Range: "photos/2024/"
  Chunks:
    - "photos/2024/01/" (thread 1)
    - "photos/2024/02/" (thread 2)
    - "photos/2024/03/" (thread 3)
  Merge sorted results

Faster for massive listings.
```

## Hot Buckets

### The Problem

**Popular buckets** get disproportionate traffic:

```
Scenario:
  - Netflix stores videos in S3
  - Popular movie: 1 million requests/second
  - All requests hit same partition!

Result:
  - Partition overloaded
  - High latency
  - Potential failures
```

### Solution 1: Partition Splitting

**Split hot partition**:

```
Before:
  Partition 874: "vacation-pics" (entire bucket)
  Load: 100,000 QPS

After:
  Partition 874-A: "vacation-pics" keys "a"-"m"
  Partition 874-B: "vacation-pics" keys "n"-"z"
  Load: 50,000 QPS each

Split can be recursive if needed.
```

**Challenges**:
```
- Listing now requires merging results from both partitions
- More complex routing
- Rebalancing overhead
```

### Solution 2: Read Replicas

**Replicate metadata more**:

```
Normal: 3 replicas (for durability)
Hot bucket: 10 replicas (for read throughput)

Distribute reads across all 10 replicas:
  100,000 QPS / 10 = 10,000 QPS per replica
  Manageable!

Writes still go to all replicas (synchronous or async).
```

### Solution 3: Caching

**Metadata cache** (in-memory):

```
┌──────────────────┐
│  Metadata Cache  │  ← In-memory (Redis/Memcached)
│  Hot objects     │
└────────┬─────────┘
         │
┌────────▼─────────┐
│  Metadata Index  │  ← On-disk (LSM-tree)
│  All objects     │
└──────────────────┘

For popular objects:
  1. Check cache: HIT (99% of time)
  2. Return cached metadata (1ms)
  3. Periodically refresh from index

For normal objects:
  1. Check cache: MISS
  2. Query index (10ms)
  3. Cache for future requests
```

## Consistency Guarantees

### Eventual Consistency (Original S3)

**Write path**:

```
PUT /bucket/key

1. Write to primary partition replica
2. Acknowledge success
3. Asynchronously replicate to other 2 replicas

Client sees success immediately.
Replicas eventually converge.
```

**Read path**:

```
GET /bucket/key

1. Read from any replica (random)
2. Might see stale data (if replica not yet updated)
3. Eventually all replicas converge

Result: Eventually consistent reads
```

**Trade-off**:
```
Pro: Low latency (fast writes)
Con: Stale reads possible
```

### Strong Consistency (Modern S3)

**Write path** (with quorum):

```
PUT /bucket/key

1. Write to all 3 replicas in parallel
2. Wait for 2+ acknowledgments (quorum)
3. Acknowledge success to client

Slower, but ensures durability.
```

**Read path** (with quorum):

```
GET /bucket/key

1. Read from 2+ replicas
2. Compare results:
   - If same: Return data
   - If different: Return newest (by timestamp)
   - Repair stale replica (read-repair)

Result: Strongly consistent reads
```

**Trade-off**:
```
Pro: No stale reads (strong consistency)
Con: Higher latency, more network traffic
```

**S3's evolution**: Started with eventual, now offers strong consistency (since Dec 2020).

## Metadata Durability

### Protecting Metadata

**Metadata is as important as data**:

```
Data loss: Bad
  - Object content missing
  - Can be restored from backup

Metadata loss: Catastrophic
  - Can't find any objects
  - Lost forever (no location info)
```

**Protection strategy**:

```
1. Replicate 3x (minimum):
   - Across different servers
   - Across different racks
   - Across different AZs (if critical)

2. Write-Ahead Log (WAL):
   - All changes logged before applied
   - WAL replicated separately
   - Can replay to recover

3. Snapshots:
   - Periodic snapshots of entire index
   - Stored durably (S3 itself!)
   - For disaster recovery

4. Cross-region replication:
   - For critical buckets
   - Async replication to other regions
   - Survive regional failure
```

### Recovery from Failure

**Scenario: Partition replica fails**

```
1. Detection:
   - Heartbeat timeout
   - Mark replica as failed

2. Switch to other replicas:
   - Requests route to healthy replicas
   - No downtime

3. Rebuild failed replica:
   - Copy data from healthy replica
   - Or replay WAL from checkpoint
   - Time: Minutes to hours

4. Add back to cluster:
   - Resume serving traffic
```

## Monitoring Metadata Health

### Key Metrics

**Performance metrics**:

```
1. Read latency (p50, p99, p99.9):
   - Target: p99 < 10ms

2. Write latency (p50, p99, p99.9):
   - Target: p99 < 20ms

3. Listing latency:
   - Target: p99 < 100ms for 1000 objects

4. Throughput:
   - Reads: QPS per partition
   - Writes: QPS per partition
   - Target: < 10,000 QPS per partition
```

**Health metrics**:

```
1. Replication lag:
   - Time for replica to catch up
   - Target: < 1 second

2. Partition size:
   - Bytes per partition
   - Target: 10-100 GB

3. Compaction lag:
   - Pending compaction work
   - Target: < 10% of data

4. Cache hit rate:
   - % of requests served from cache
   - Target: > 80%
```

### Alerting

**Critical alerts**:

```
1. Partition unavailable:
   - Less than 2 replicas healthy
   - Immediate action required

2. High latency:
   - p99 > 100ms sustained
   - Investigate hot partitions

3. Replication lag:
   - Lag > 10 seconds
   - Check network, load

4. Capacity:
   - Partition > 80 GB
   - Plan to split
```

## Key Takeaways

- **Metadata is harder than data** - Scale, performance, and consistency challenges
- **Custom index required** - NoSQL and RDBMS don't fit perfectly
- **LSM-tree is optimal** - Write-optimized, range scans, scales to billions
- **Partition by bucket** - Enables efficient listing within partition
- **Caching is critical** - 80%+ hit rate makes hot buckets manageable
- **Durability matters** - Losing metadata is worse than losing data
- **Consistency evolved** - From eventual to strong consistency
- **Monitoring essential** - Metadata performance impacts everything

## What's Next?

We've covered metadata storage and consistency basics. But consistency models deserve deeper exploration. In the next chapter, we'll dive into consistency models and how S3 evolved from eventual to strong consistency.

---

**[← Previous: Replication and Erasure Coding](./09-replication-erasure-coding.md) | [Back to Contents](./README.md) | [Next: Consistency Models →](./11-consistency-models.md)**
