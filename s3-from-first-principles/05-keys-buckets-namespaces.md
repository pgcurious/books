# Chapter 5: Keys, Buckets, and Namespaces

> **"A well-designed namespace is invisible when it works, and catastrophic when it doesn't."**

## The Namespace Problem

You have the object storage abstraction. Now you need to design how objects are organized and named.

### The Challenge

**Requirements**:
- Support trillions of objects
- Globally unique identification
- Fast lookup (milliseconds)
- Intuitive for users
- Partitionable across servers
- Support multiple users without conflicts

**Traditional approaches won't work**:
```
File system: /users/alice/photos/beach.jpg
→ Hierarchical, slow at scale

Database: users.photos WHERE user_id=123
→ Query overhead, schema complexity

We need something simpler and more scalable.
```

## The Two-Level Namespace

### Level 1: Buckets

**A bucket is a container for objects**:

```
┌─────────────────────────────────┐
│     Bucket: "alice-photos"      │
├─────────────────────────────────┤
│  ┌─────────────────────────┐   │
│  │ Key: "beach.jpg"        │   │
│  │ Data: <image bytes>     │   │
│  └─────────────────────────┘   │
│  ┌─────────────────────────┐   │
│  │ Key: "mountain.jpg"     │   │
│  │ Data: <image bytes>     │   │
│  └─────────────────────────┘   │
│  ┌─────────────────────────┐   │
│  │ Key: "sunset.jpg"       │   │
│  │ Data: <image bytes>     │   │
│  └─────────────────────────┘   │
└─────────────────────────────────┘
```

**Why buckets?**

1. **Isolation**: Each user gets their own namespace
2. **Security**: Permissions at bucket level
3. **Billing**: Track costs per bucket
4. **Configuration**: Settings per bucket (encryption, lifecycle, etc.)
5. **Partitioning**: Distribute buckets across servers

### Level 2: Keys

**Keys are strings that identify objects within a bucket**:

```
Bucket: "alice-photos"
Keys:
  - "beach.jpg"
  - "2024/january/beach.jpg"
  - "vacations/hawaii/day1/beach.jpg"
  - "family/photos/2024-01-15/beach.jpg"
```

**Keys look hierarchical but aren't**:
- The "/" character has no special meaning
- "2024/january/beach.jpg" is just a string
- No actual directory structure
- Users can simulate folders in UI

## The Global Namespace Challenge

### The Problem

**Every bucket name must be globally unique**:

```
Alice creates: "photos"
Bob tries to create: "photos"
→ Fails! Name already taken.
```

**Why globally unique?**

#### Reason 1: DNS Addressing

```
Bucket: "alice-photos"
DNS: alice-photos.s3.amazonaws.com

Benefits:
✓ Direct URL access
✓ CDN integration
✓ SSL certificates per bucket
✓ Routing optimization
```

**If not unique**:
```
Alice's "photos": How do we route photos.s3.amazonaws.com?
Bob's "photos": Both can't exist at same domain!
```

#### Reason 2: Simplified Routing

```
Request: GET /photos/beach.jpg
Host: s3.amazonaws.com

Routing:
1. Extract bucket name: "photos"
2. Hash("photos") → Server group 42
3. Route to Server group 42
4. Lookup object

If bucket names weren't unique:
1. Extract bucket + account ID
2. Hash(account_id + "photos") → ???
3. Need account ID in every request (complex)
```

#### Reason 3: Prevent Confusion

```
Company A: "backups" bucket (sensitive data)
Company B: "backups" bucket (different data)

If names collide:
- Logging confusion
- Support nightmares
- Security risks
```

### The Global Namespace Design

**Bucket naming rules**:

```
Length: 3-63 characters
Characters: Lowercase letters, numbers, hyphens
Format: DNS-compatible
Cannot: Start with "xn--", end with "-s3alias"

Valid:
✓ "my-photos"
✓ "company-backups-2024"
✓ "user123-data"

Invalid:
✗ "My-Photos" (uppercase)
✗ "my_photos" (underscore)
✗ "a" (too short)
✗ "my..photos" (consecutive dots)
```

**Why DNS-compatible?**
- Enables bucket.s3.amazonaws.com URLs
- Works with SSL certificates
- Standard tooling compatibility

### The Squatting Problem

**Early S3 had namespace squatting**:

```
Common names taken:
- "backup"
- "data"
- "files"
- "images"
- "videos"

Everyone had to use:
- "mycompany-backup"
- "user-data-store"
- "app-files-2024"
```

**Solutions considered**:

1. **Paid premium names**: Popular names cost more (rejected - bad UX)
2. **Namespaced buckets**: /account-id/bucket-name (rejected - complex)
3. **Accept it**: First-come-first-served (chosen)

**In practice**:
- Companies use prefixes: "acme-backups", "acme-logs"
- Users add dates: "photos-2024"
- Apps use UUIDs: "app-a3f2-b4c9-storage"

## Key Design Patterns

### Pattern 1: Flat Keys

**Simple approach**:
```
Keys:
  - "photo1.jpg"
  - "photo2.jpg"
  - "photo3.jpg"
  - "document.pdf"
```

**Pros**:
- Simple
- Fast lookup
- Easy to partition

**Cons**:
- Hard to organize
- Poor for listing related objects
- No logical grouping

**Use case**: Small buckets, simple data

### Pattern 2: Prefix-Based Organization

**Simulate directories**:
```
Keys:
  - "photos/2024/january/beach.jpg"
  - "photos/2024/january/sunset.jpg"
  - "photos/2024/february/snow.jpg"
  - "documents/taxes/2024.pdf"
  - "documents/receipts/january.pdf"
```

**Pros**:
- Logical organization
- Can list by prefix: "photos/2024/january/"
- User-friendly (looks like folders)
- Good for hierarchical data

**Cons**:
- Keys are longer
- Listing entire prefix can be slow

**Use case**: Most S3 usage

### Pattern 3: Hash-Prefixed Keys

**Add random prefix for better distribution**:
```
Original key: "user_123_photo.jpg"
Hashed key: "a3f2/user_123_photo.jpg"
            (hash prefix)

Distribution:
  - "a3f2/user_123_photo.jpg" → Server 1
  - "b8d4/user_456_photo.jpg" → Server 2
  - "c1a9/user_789_photo.jpg" → Server 3
```

**Pros**:
- Better load distribution
- Avoids hotspots
- Scales better at extreme scale

**Cons**:
- Harder to debug
- Can't list by semantic prefix
- Requires maintaining hash mapping

**Use case**: Extreme scale (billions of objects)

### Pattern 4: Date-Based Partitioning

**Organize by time**:
```
Keys:
  - "logs/2024/01/15/app.log"
  - "logs/2024/01/16/app.log"
  - "photos/2024-01-15/img1.jpg"
  - "events/2024/january/event1.json"
```

**Pros**:
- Natural organization
- Easy lifecycle management (delete old data)
- Good for time-series data
- Enables efficient archival

**Cons**:
- Hotspots (everyone writes to "today")
- Latest data gets all traffic

**Use case**: Logs, analytics, time-series data

### Pattern 5: Reverse Domain Pattern

**Use reverse domain for uniqueness**:
```
Keys:
  - "com/example/app/user123/photo.jpg"
  - "org/wikipedia/images/logo.png"
  - "io/github/project/release.tar.gz"
```

**Pros**:
- Natural uniqueness
- Clear ownership
- Mirrors internet structure

**Cons**:
- Verbose
- Not intuitive for end users

**Use case**: Multi-tenant systems, CDN origins

## Partitioning Strategies

### Hash-Based Partitioning

**How it works**:
```
Function: partition(bucket, key)
  hash = MD5(bucket + key)
  partition_id = hash % NUM_PARTITIONS
  return partition_id

Example:
  bucket="photos", key="beach.jpg"
  hash = MD5("photos/beach.jpg") = a3f2...
  partition = a3f2... % 1024 = 874
  → Store on partition 874
```

**Characteristics**:
```
✓ Even distribution
✓ Deterministic (same input → same partition)
✓ Scalable (add more partitions)
✗ Can't range query
✗ Listing requires fan-out
```

### Range-Based Partitioning

**How it works**:
```
Partitions by key range:
  Partition 1: "a" ... "d"
  Partition 2: "e" ... "h"
  Partition 3: "i" ... "l"
  ...

Example:
  key="beach.jpg" → starts with "b"
  → Store on partition 1
```

**Characteristics**:
```
✓ Range queries fast
✓ Listing is local
✗ Hotspots (if keys cluster)
✗ Rebalancing is complex
```

**S3's choice**: Hash-based for distribution, with optimizations for listing.

## The Listing Challenge

### The Problem

**Listing billions of objects is hard**:

```
Request: List objects in "photos" bucket

Naive approach:
1. Scan all partitions
2. Collect all keys
3. Sort
4. Return

For 1 billion objects:
  Time: Hours (!)
  Memory: Gigabytes
  Network: Terabytes
```

### The Solution: Pagination + Prefix Filtering

**Pagination**:
```
Request 1: List first 1000 objects
Response:
  Objects: [obj1, obj2, ..., obj1000]
  NextMarker: "photo_1000.jpg"
  IsTruncated: true

Request 2: List next 1000 (marker="photo_1000.jpg")
Response:
  Objects: [obj1001, obj1002, ..., obj2000]
  NextMarker: "photo_2000.jpg"
  IsTruncated: true
```

**Prefix filtering**:
```
Request: List with prefix="photos/2024/"
→ Only returns keys starting with "photos/2024/"
→ Much faster than full scan

Implementation:
1. Find partitions that might contain prefix
2. Scan only those partitions
3. Filter by prefix
4. Return results
```

**Delimiter** (simulate folders):
```
Request: List with prefix="photos/" delimiter="/"
Returns:
  Objects:
    - "photos/beach.jpg"
    - "photos/sunset.jpg"
  CommonPrefixes:
    - "photos/2024/"
    - "photos/2023/"

→ Like "ls" in a directory
```

## The Metadata Index

### What Needs to Be Indexed?

**For each object**:
```
Index entry:
  - Bucket name
  - Key
  - Size
  - ETag (content hash)
  - Last modified timestamp
  - Storage class
  - Location (which servers/disks)
```

**For 1 trillion objects**:
```
Per object: ~500 bytes (metadata)
Total: 500 TB just for metadata!

This must be:
✓ Fast to query
✓ Highly available
✓ Distributed
✓ Consistent
```

### Metadata Storage Design

**Option 1: Dedicated database**
```
PostgreSQL with indexes:
  CREATE INDEX ON objects(bucket, key);

Pros:
  ✓ Strong consistency
  ✓ ACID
Cons:
  ✗ Limited scale (billions, not trillions)
  ✗ Single point of failure
  ✗ Expensive
```

**Option 2: Distributed key-value store**
```
DynamoDB / Cassandra:
  Key: bucket + key
  Value: {size, etag, location, ...}

Pros:
  ✓ Scales to trillions
  ✓ Distributed
Cons:
  ✗ Eventual consistency
  ✗ Complex queries
```

**Option 3: Custom distributed index**
```
Sharded by hash(bucket + key):
  Shard 1: a-d
  Shard 2: e-h
  ...

Each shard:
  - B-tree for fast lookup
  - Replicated (3x)
  - Backed up

Pros:
  ✓ Scales infinitely
  ✓ Optimized for our workload
Cons:
  ✗ Must build it ourselves
```

**S3's approach**: Custom distributed index (option 3)

## The Partition Map

### Tracking Object Locations

**The partition map** tracks which objects are on which servers:

```
┌──────────────────────────────┐
│     Partition Map            │
├──────────────────────────────┤
│ Bucket: "photos"             │
│ Key: "beach.jpg"            │
│ → Partition: 874             │
│ → Servers: [s1, s4, s9]     │
│                              │
│ Bucket: "docs"               │
│ Key: "report.pdf"           │
│ → Partition: 123             │
│ → Servers: [s2, s5, s7]     │
└──────────────────────────────┘
```

**Lookup process**:
```
GET /photos/beach.jpg

1. Hash("photos" + "beach.jpg") → partition 874
2. Query partition map: partition 874 → servers [s1, s4, s9]
3. Read from one of [s1, s4, s9]
4. Return data
```

**Partition map properties**:
- Distributed (replicated)
- Cached aggressively
- Updated on rebalancing
- Small size (millions of partitions, not billions of objects)

## Namespace Scalability

### How Many Buckets?

**Per account limits** (early S3):
```
Default: 100 buckets per account

Why limit?
- Prevents abuse
- Simplifies management
- Most users don't need more

Can be increased: Yes (via support)
```

**In practice**:
```
Most accounts: 1-10 buckets
Power users: 100-1000 buckets
Enterprises: Thousands (with limit increase)
```

### How Many Objects Per Bucket?

**No limit** (theoretically):
```
S3 supports:
- Billions of objects per bucket
- Trillions across all buckets

Practical limits:
- Listing slows down (use pagination)
- Eventually need multiple buckets for organization
```

### How Many Requests?

**Per bucket**:
```
Default: 3,500 PUT/POST/DELETE per second
         5,500 GET per second

For higher:
- Distribute load across key prefixes
- Use hash prefixes
- Contact support for limit increase
```

**Why limits?**
- Prevent hotspots
- Ensure fair sharing
- Predictable performance

## The DNS Problem

### Bucket Names as Domains

**Early design**:
```
bucket.s3.amazonaws.com

Example:
alice-photos.s3.amazonaws.com
→ Points to S3 infrastructure
→ Routes to correct bucket
```

**SSL certificate challenge**:
```
Certificate for: *.s3.amazonaws.com
→ Covers all buckets

But DNS rules:
- Must be lowercase
- No underscores
- No dots (except as subdomain separators)

Hence: Strict bucket naming rules
```

### Virtual-Hosted-Style URLs

**Modern approach**:
```
Virtual-hosted:
https://bucket-name.s3.region.amazonaws.com/key

Path-style:
https://s3.region.amazonaws.com/bucket-name/key
```

**Virtual-hosted benefits**:
- Better performance (CDN caching)
- Cleaner URLs
- Easier CORS handling

**Path-style** (deprecated):
- Simpler routing
- But less efficient

## Key Takeaways

- **Two-level namespace** - Buckets provide isolation, keys identify objects
- **Global uniqueness required** - Enables DNS routing and simplifies architecture
- **Keys are just strings** - No hierarchy enforced, but "/" enables organization
- **Hash partitioning scales** - Even distribution across unlimited servers
- **Listing requires optimization** - Pagination and prefix filtering essential
- **Metadata index is critical** - Must scale to trillions of entries
- **Namespace design enables scale** - Simple but powerful model

## What's Next?

We have the namespace model. Now we need to ensure objects never get lost. In the next chapter, we'll tackle the durability challenge: achieving 11 nines.

---

**[← Previous: The Object Storage Insight](./04-object-storage-insight.md) | [Back to Contents](./README.md) | [Next: The Durability Challenge →](./06-durability-challenge.md)**
