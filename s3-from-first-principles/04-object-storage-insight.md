# Chapter 4: The Object Storage Insight

> **"Sometimes the breakthrough comes from questioning the most basic assumptions."**

## The Whiteboard Session

You gather your team. The whiteboard question:

> **"What is the MINIMUM abstraction we need for internet-scale storage?"**

## The Thought Experiment

### What Do Users Actually Need?

Strip away everything traditional storage provides. What remains?

**Users need to**:
1. **Store** a piece of data
2. **Retrieve** that data later
3. **Delete** it when done

That's it.

### What Don't They Need?

**Things file systems provide that we DON'T need**:

```
✗ Directory hierarchy
✗ POSIX permissions
✗ File locking
✗ Inodes
✗ Hard links / soft links
✗ Special files (pipes, sockets)
✗ File system check (fsck)
✗ Mount points
```

**Things databases provide that we DON'T need**:

```
✗ Transactions
✗ Joins
✗ Indexes (beyond key lookup)
✗ Query language (SQL)
✗ ACID guarantees
✗ Foreign keys
```

**By removing what we don't need, we unlock scalability.**

## The Core Abstraction: Objects

### What is an Object?

An object is the simplest unit of storage:

```
┌─────────────────────────────┐
│         OBJECT              │
├─────────────────────────────┤
│ Key: "photos/beach.jpg"     │  ← Unique identifier
│ Value: <binary data>        │  ← The actual data
│ Metadata: {                 │  ← Optional attributes
│   size: 2048576,           │
│   type: "image/jpeg",      │
│   modified: "2024-01-15"   │
│ }                          │
└─────────────────────────────┘
```

**Properties**:
- **Immutable**: Once written, never modified (overwrite creates new version)
- **Self-contained**: All information bundled together
- **Flat namespace**: No hierarchy (just keys in a bucket)
- **Simple operations**: PUT, GET, DELETE

### Why This Works

#### 1. Immutability Enables Replication

**Mutable files**:
```
Problem: What if two users modify the same file simultaneously?

Solution requires:
- Distributed locking
- Conflict resolution
- Version vectors
- Complex consistency protocols

Result: Slow, complex, expensive
```

**Immutable objects**:
```
Modification = new write with same key

No locks needed:
- Write to all replicas
- Last write wins (by timestamp)
- Simple eventual consistency
- Fast, simple, cheap

If you want versions: Use versioning feature
```

#### 2. Flat Namespace Enables Partitioning

**Hierarchical**:
```
/users/alice/photos/2024/beach.jpg

To find this file:
1. Lookup /users
2. Lookup /users/alice
3. Lookup /users/alice/photos
4. Lookup /users/alice/photos/2024
5. Lookup beach.jpg

5 lookups! Each could be on different server.
Can't efficiently partition.
```

**Flat**:
```
Key: "users/alice/photos/2024/beach.jpg"

To find this object:
1. Hash(key) → determines server
2. Single lookup

1 lookup! Easily partitionable.
```

**Note**: The key LOOKS hierarchical, but it's just a string. No actual directory structure.

#### 3. Simple Operations Enable Performance

**File system operations (POSIX)**:
```c
int fd = open("/path/file", O_RDWR);  // Open
flock(fd, LOCK_EX);                   // Lock
lseek(fd, offset, SEEK_SET);          // Seek
write(fd, buffer, size);              // Write
fsync(fd);                            // Sync
flock(fd, LOCK_UN);                   // Unlock
close(fd);                            // Close

7 operations for one write!
Plus permission checks at each step.
```

**Object storage**:
```
PUT /bucket/key HTTP/1.1
<data>

1 operation!
```

**Performance impact**:
```
File system: 7 operations × 1ms = 7ms
Object storage: 1 operation × 1ms = 1ms

7x faster!
(Plus reduced complexity)
```

## The Object Model Details

### Keys

**Keys are strings**:
```
Valid keys:
"photo.jpg"
"documents/2024/report.pdf"
"user_123_avatar.png"
"logs/2024/01/15/app.log"
"really/long/path/that/looks/like/directories/but/is/just/a/string.txt"
```

**Best practices**:
- Use UTF-8
- Length limit: 1024 bytes (reasonable)
- Allow special chars (/, -, _, .)
- No hierarchy enforcement

**Benefits**:
- Users can simulate directories (by using "/" in key)
- But we don't have to implement directory operations
- Partitioning is simple (hash the key)

### Values

**Values are byte arrays**:
```
Anything goes:
- Text files
- Images
- Videos
- Binary data
- Archives
- Encrypted data
```

**Size limits**:
```
Minimum: 0 bytes (empty object is valid)
Maximum: ?

Practical considerations:
- Single PUT limit: 5 GB (HTTP timeout issues)
- Total object size: 5 TB (via multipart)
- No minimum (even 0 bytes is useful for flags)
```

### Metadata

**System metadata** (automatic):
```
{
  "size": 1048576,
  "etag": "md5-hash-of-content",
  "last-modified": "2024-01-15T10:30:00Z",
  "storage-class": "STANDARD"
}
```

**User metadata** (custom):
```
{
  "content-type": "image/jpeg",
  "cache-control": "max-age=3600",
  "x-custom-field": "my-value"
}
```

**Metadata is immutable too**:
- Can't change metadata without rewriting object
- Simple consistency model

## The Bucket Abstraction

### Why Buckets?

Objects need to be grouped somehow:

**Option 1: Global namespace**
```
Key: "users/alice/photo.jpg"

Problems:
- Name collisions (what if Bob also has photo.jpg?)
- Security (everyone shares same namespace)
- Billing (can't separate Alice's data from Bob's)
```

**Option 2: Buckets (containers)**
```
Bucket: "alice-photos"
Key: "photo.jpg"

Full path: s3://alice-photos/photo.jpg

Benefits:
✓ Name isolation (Alice and Bob can both have photo.jpg)
✓ Security boundary (permissions per bucket)
✓ Billing boundary (track costs per bucket)
✓ Configuration boundary (encryption, lifecycle, etc.)
```

### Bucket Design

**Properties**:
```
Bucket name:
- Globally unique (across all users!)
- DNS-compatible (lowercase, numbers, hyphens)
- 3-63 characters

Why globally unique?
- Enables bucket-as-subdomain: alice-photos.s3.amazonaws.com
- Simplifies routing
- Prevents confusion
```

**Bucket operations**:
```
CreateBucket(name)
DeleteBucket(name)  # Must be empty
ListBuckets()       # Per account
```

**Object operations within bucket**:
```
PutObject(bucket, key, data)
GetObject(bucket, key)
DeleteObject(bucket, key)
ListObjects(bucket, prefix)  # Filter by prefix
```

## The API Design

### RESTful HTTP

**Why HTTP?**
- Universal (every language/platform has HTTP)
- Firewall-friendly (port 80/443)
- Well understood (caching, proxies, CDN)
- Stateless (scales easily)

**Core operations**:
```
PUT /bucket/key HTTP/1.1        → Store object
Host: s3.amazonaws.com
Content-Length: 1048576

<binary data>

---

GET /bucket/key HTTP/1.1        → Retrieve object
Host: s3.amazonaws.com

---

DELETE /bucket/key HTTP/1.1     → Delete object
Host: s3.amazonaws.com

---

GET /bucket?prefix=photos/ HTTP/1.1  → List objects
Host: s3.amazonaws.com
```

**That's 90% of the API!**

### Additional Operations

**HEAD**: Get metadata without data
```
HEAD /bucket/key HTTP/1.1
→ Returns metadata, no body (fast!)
```

**Copy**: Server-side copy
```
PUT /dest-bucket/dest-key HTTP/1.1
x-amz-copy-source: /source-bucket/source-key
→ Copies without downloading/uploading
```

**Multipart Upload**: For large objects
```
1. Initiate: POST /bucket/key?uploads
2. Upload parts: PUT /bucket/key?partNumber=N&uploadId=X
3. Complete: POST /bucket/key?uploadId=X
```

## The Storage Model

### How Objects are Stored

**Conceptually**:
```
Map<String, ByteArray> objects;

PUT:  objects[key] = value;
GET:  return objects[key];
DELETE: objects.remove(key);
```

**Actually** (simplified):
```
┌──────────────────┐
│  Frontend API    │  Receives HTTP requests
├──────────────────┤
│  Metadata Store  │  Tracks: bucket → keys → metadata
├──────────────────┤
│  Data Store      │  Stores: object data on disks
└──────────────────┘
```

**Data path**:
```
PUT request:
1. API validates request
2. Metadata store: Create entry (bucket, key, size, etc.)
3. Data store: Write bytes to disk(s)
4. Metadata store: Mark as committed
5. Return success

GET request:
1. API validates request
2. Metadata store: Lookup object location
3. Data store: Read bytes from disk(s)
4. Return data

DELETE request:
1. API validates request
2. Metadata store: Mark as deleted
3. Data store: (Asynchronously) Reclaim space
4. Return success
```

## The Consistency Model (Initial)

### What Consistency Do We Need?

**Strong consistency** (like POSIX):
```
write(file, data);
read(file);  // Must see latest write

Requires:
- Synchronization across replicas
- Locks
- High latency
```

**Eventual consistency** (relaxed):
```
write(key, data);
// ... data propagates to replicas ...
read(key);  // Might see old data briefly, but eventually sees new data

Allows:
- Async replication
- No locks
- Low latency
```

**For web-scale storage, eventual consistency is acceptable**:

### The Consistency Guarantees (S3's original model)

**Read-after-write consistency for new objects**:
```
PUT /bucket/new-key  → Success
GET /bucket/new-key  → Guaranteed to see the data ✓
```

**Eventual consistency for overwrites**:
```
PUT /bucket/existing-key (v1)
PUT /bucket/existing-key (v2)
GET /bucket/existing-key  → Might see v1 or v2 (!)

Eventually: All GETs will see v2
```

**Eventual consistency for deletes**:
```
DELETE /bucket/key
GET /bucket/key  → Might still return data briefly (!)

Eventually: Returns 404
```

### Why This Trade-off?

**Strong consistency**:
```
Write to 3 replicas:
1. Write to Replica 1
2. Wait for Replica 1 ACK
3. Write to Replica 2
4. Wait for Replica 2 ACK
5. Write to Replica 3
6. Wait for Replica 3 ACK
7. Return success

Total latency: 6 network round-trips
```

**Eventual consistency**:
```
Write to 3 replicas:
1. Write to Replica 1 (async)
2. Write to Replica 2 (async)
3. Write to Replica 3 (async)
4. Return success (after first ACK)

Total latency: 1 network round-trip
6x faster!
```

**Most applications can tolerate brief inconsistency.**

## The Object Storage Advantage

### What We Gained

**Simplicity**:
- 3 core operations (PUT, GET, DELETE)
- No complex semantics
- Easy to understand and implement

**Scalability**:
- Flat namespace → easy partitioning
- Immutability → simple replication
- Eventual consistency → no coordination

**Performance**:
- Fewer operations per request
- Async replication
- HTTP caching

**Cost**:
- Simpler code → less bugs
- Less coordination → less overhead
- Commodity hardware friendly

### What We Lost

**Compared to file systems**:
- ✗ No in-place updates (must rewrite entire object)
- ✗ No atomic append
- ✗ No directory rename
- ✗ No hard links

**Compared to databases**:
- ✗ No transactions
- ✗ No queries (only key lookup)
- ✗ No indexes (besides key)
- ✗ No joins

**These are acceptable trade-offs for our use case.**

## The Moment of Clarity

You step back from the whiteboard:

```
Object Storage = Key-Value Store for Bytes

Simple abstraction:
- Keys (strings)
- Values (byte arrays)
- Metadata (key-value pairs)

Simple operations:
- PUT (create/overwrite)
- GET (retrieve)
- DELETE (remove)

Simple consistency:
- Read-after-write for new objects
- Eventual for overwrites/deletes

This can scale infinitely.
```

**The insight**: By simplifying the abstraction, we unlock scalability.

---

## Key Takeaways

- **Objects are the right abstraction** - Simpler than files or blocks
- **Immutability enables scale** - No locks, simple replication
- **Flat namespace enables partitioning** - Hash-based distribution
- **HTTP API is universal** - Works everywhere, simple integration
- **Eventual consistency is acceptable** - Most applications can tolerate brief inconsistency
- **Trade-offs are intentional** - Lost features are ones we don't need

## What's Next?

Now that we have the object storage abstraction, we need to design the namespace model. How do keys, buckets, and namespaces work together?

---

**[← Previous: The Cost Constraint](./03-cost-constraint.md) | [Back to Contents](./README.md) | [Next: Keys, Buckets, and Namespaces →](./05-keys-buckets-namespaces.md)**
