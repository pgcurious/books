# Chapter 16: When Object Storage Struggles

> **"Every tool has its limits. Knowing them is as important as knowing its strengths."**

## The Anti-Patterns

S3 is powerful, but it's not a universal solution. Let's explore where it struggles.

## Anti-Pattern 1: Database Replacement

### The Temptation

**"S3 is key-value storage, like a database!"**

```
Flawed thinking:
  Database: key → value
  S3: key → object
  Therefore: Use S3 as database!

Reality: Terrible idea.
```

### Why It Fails

**Missing database features**:

```
❌ No transactions:
  Can't atomically update multiple objects
  No ACID guarantees

❌ No joins:
  Can't query across objects
  Each object is independent

❌ No indexes (beyond key):
  Can't query by non-key fields
  No secondary indexes

❌ No query language:
  No SQL, no flexible queries
  Only GET by exact key

❌ Poor write performance:
  Latency: 10-100ms per write
  vs Database: 1-10ms

❌ No concurrency control:
  No locks, no isolation levels
  Optimistic concurrency only (via ETags)
```

### Real Example (Bad)

**User management**:

```
Bad approach: Store users in S3

Structure:
  s3://users/user-123.json
  s3://users/user-456.json

Query "all users in New York":
  1. List all user objects (slow for millions)
  2. Download each object (expensive)
  3. Filter in application (inefficient)

For 1 million users:
  - List: 1000 requests (pagination)
  - Download: 1 million requests
  - Transfer: Gigabytes
  - Time: Hours
  - Cost: $$$

Database (PostgreSQL):
  SELECT * FROM users WHERE city = 'New York';
  - Time: Milliseconds
  - Cost: Query is free
```

**Better approach**: Use a real database (RDS, DynamoDB).

### When "S3 as Database" Might Work

**Very limited scenarios**:

```
Use case: Store application configuration

Requirements:
  ✓ Read >> Write (config rarely changes)
  ✓ Simple key lookup (app_name → config)
  ✓ Small objects (< 1 MB)
  ✓ No complex queries needed
  ✓ Eventual consistency OK

Example:
  s3://config/app-prod.json
  s3://config/app-staging.json

Acceptable because:
  - Few objects (< 1000)
  - Rare writes
  - Simple lookup
  - Can cache aggressively

Still not ideal, but tolerable.
```

## Anti-Pattern 2: Frequently Updated Files

### The Problem

**S3 is optimized for immutability**:

```
Scenario: Log file that grows continuously

Bad approach:
  1. Read entire log file from S3
  2. Append new lines
  3. Write entire file back to S3

Problems:
  ❌ Each append = full file upload
  ❌ Wastes bandwidth (uploading unchanged data)
  ❌ Slow (download + upload entire file)
  ❌ Expensive (storage of old versions)
```

### Real Example (Bad)

**Counter file**:

```
Bad: Use S3 for frequently updated counter

counter.json: {"count": 0}

Update:
  1. GET counter.json → {"count": 0}
  2. Increment locally → {"count": 1}
  3. PUT counter.json → {"count": 1}

10,000 updates:
  - 10,000 GET requests
  - 10,000 PUT requests
  - 20,000 total requests
  - With versioning: 10,000 versions stored

Cost: $0.08 (requests) + storage of 10,000 versions

Concurrency issues:
  User A: GET count=10, increment to 11
  User B: GET count=10, increment to 11
  Result: count=11 (should be 12!)
  → Lost update!
```

**Better approach**: Use DynamoDB or Redis.

```
DynamoDB:
  ATOMIC INCREMENT counter
  - Single operation
  - No lost updates
  - Low latency (~10ms)
  - Cost: $0.25 per million writes
```

### When Append-Like Operations Are OK

**Batch uploads**:

```
Use case: Hourly log aggregation

Process:
  1. Application writes to local file (1 hour)
  2. Every hour: Upload file to S3
     s3://logs/2024/01/15/app-10.log
  3. New hour: New file

This is fine:
  ✓ Not updating existing objects
  ✓ Write-once per file
  ✓ Immutable after upload
```

## Anti-Pattern 3: POSIX Filesystem Replacement

### The Temptation

**"Mount S3 as a filesystem!"**

Tools exist:
- s3fs-fuse
- goofys
- s3backer

But they have severe limitations.

### Why It Fails

**POSIX semantics don't map**:

```
POSIX expectations          S3 reality
─────────────────────────   ────────────────────────
Directories exist           No directories (just key prefixes)
Atomic rename               No atomic rename
Append to file              Must rewrite entire object
Locking                     No locking
Immediate consistency       Eventual consistency (originally)
Low latency (~1ms)          Higher latency (~10-100ms)
Local disk performance      Network performance
Unlimited small files       Poor for many small files
```

**Performance issues**:

```
Local filesystem:
  open("file.txt") → 0.01ms
  read(1 KB) → 0.01ms
  close() → 0.01ms
  Total: ~0.03ms

S3 via s3fs:
  open("s3://bucket/file.txt") → HEAD request → 50ms
  read(1 KB) → GET request → 50ms
  close() → No-op → 0ms
  Total: ~100ms

3000x slower!
```

### Real Example (Bad)

**Application working directory**:

```
Bad: Use S3 as working directory for compilation

gcc compile.c -o app

Compilation:
  1. Read hundreds of header files
  2. Write temporary .o files
  3. Link into final binary

With S3 (via s3fs):
  - Each header read: 50-100ms
  - Each temp file write: 50-100ms
  - 1000+ files involved
  - Total: Minutes to hours

With local disk:
  - Total: Seconds

100-1000x slower with S3!
```

**Better approach**: Use local disk or EBS.

### When "S3 as Filesystem" Might Work

**Very limited**:

```
Use case: Read-only data archives

Scenario:
  - Mount S3 bucket read-only
  - Access large archive files occasionally
  - Sequential reads (not random seeks)

Example:
  mount -t s3fs archive /mnt/archive
  grep "pattern" /mnt/archive/logs-2020.txt

Acceptable because:
  ✓ Read-only
  ✓ Occasional access
  ✓ Large files (sequential reads)
  ✓ Not performance-critical

Still not ideal, but tolerable for archives.
```

## Anti-Pattern 4: High-Frequency Transactions

### The Problem

**S3 is not designed for high-frequency writes to same key**:

```
Scenario: Update inventory count for popular item

Item: iPhone (very popular)
Updates: 1000/second

Bad approach:
  inventory.json: {"iphone": 500}

  Each sale:
    GET inventory.json
    Decrement count
    PUT inventory.json

  Problems:
    ❌ 1000 GET + 1000 PUT = 2000 requests/second to ONE key
    ❌ Rate limiting (S3 limits per-prefix)
    ❌ Eventual consistency issues
    ❌ Lost updates (race conditions)
```

**S3 request rate limits**:

```
Per-prefix limits:
  PUT/DELETE: 3,500 per second
  GET: 5,500 per second

If all traffic to one key:
  Exceeds limit → 503 Slow Down errors

Need to distribute across prefixes!
```

### Better Approach

**Use database for transactions**:

```
DynamoDB:
  - Designed for high-frequency updates
  - ACID transactions
  - Single-digit millisecond latency
  - Scales to millions of requests/second

RDS:
  - Traditional ACID database
  - Complex queries
  - Joins, indexes
```

**Pattern: S3 for archive, database for active**:

```
Architecture:
  1. Active data: DynamoDB (real-time)
  2. Archive data: S3 (historical)

Example: E-commerce orders
  - Recent orders (30 days): DynamoDB
  - Historical orders (1+ year): S3

Benefits:
  ✓ Fast queries on recent data
  ✓ Cheap storage for historical data
  ✓ Each system used for its strengths
```

## Anti-Pattern 5: Small File Proliferation

### The Problem

**Millions of tiny files**:

```
Scenario: Store sensor data (IoT)

Bad approach:
  Each sensor reading: One S3 object
  Reading size: 100 bytes
  Sensors: 1 million
  Frequency: Every second

  Objects per day: 86.4 billion
  Objects per year: 31.5 trillion

Problems:
  ❌ Request costs dominate:
    PUT: $0.005 per 1000 = $157,680/day
  ❌ Storage overhead (metadata > data):
    100 byte object + 1 KB metadata = 10x overhead
  ❌ Listing is slow:
    Billions of objects in bucket
```

### Better Approach

**Batch into larger files**:

```
Good approach:
  Aggregate readings into hourly files

  Process:
    1. Buffer readings (1 hour)
    2. Write to file: sensor-2024-01-15-10.json
    3. Upload to S3

  Objects per day: 24 (one per hour)
  Object size: ~360 MB (1M sensors × 3600s × 100B)

  Benefits:
    ✓ 3.6 billion times fewer objects
    ✓ Request cost: $0.12/day (vs $157k)
    ✓ Better for analytics (fewer files to read)
    ✓ Lower metadata overhead
```

**Use Kinesis Firehose**:

```
Firehose automatically batches:
  Input: Sensor stream
  Output: S3 files every 5 minutes
  Automatic compression (gzip)
  Automatic partitioning (by date/time)

No code needed for batching!
```

## Anti-Pattern 6: Complex Directory Operations

### The Problem

**S3 has no real directories**:

```
Remember: Directories are illusion
  Keys: "dir1/dir2/file.txt"
  No actual directory structure

Operations that fail:
  ❌ Rename directory:
    Must copy all objects, delete originals
    1 million files = 2 million operations

  ❌ Get directory size:
    Must list and sum all objects
    Slow for large directories

  ❌ Set permissions on directory:
    No directory-level permissions
    Must set on each object

  ❌ Move directory atomically:
    No atomic move
    Partial failures possible
```

### Real Example (Bad)

**User workspace**:

```
Bad: Organize user files like filesystem

Structure:
  /users/alice/documents/
  /users/alice/photos/
  /users/alice/videos/

Rename alice → alice_backup:
  1. List all objects under /users/alice/
  2. Copy each to /users/alice_backup/
  3. Delete originals

  For 100,000 files:
    - 1 LIST operation (paginated)
    - 100,000 COPY operations
    - 100,000 DELETE operations
    - Time: Hours
    - Not atomic (can fail midway)
```

**Better approach**:

```
Store metadata in database:
  Database: user_id → folder structure
  S3: Objects with UUIDs as keys

Rename:
  UPDATE users SET username = 'alice_backup' WHERE user_id = 123;
  (Instant, atomic)

S3 objects unchanged:
  s3://files/uuid-abc123
  s3://files/uuid-def456
```

## Anti-Pattern 7: Shared Mutable State

### The Problem

**No locking mechanism**:

```
Scenario: Multiple workers updating shared file

Bad approach:
  shared-state.json: {"processed": 100}

  Worker A:
    GET shared-state.json → {"processed": 100}
    Process 10 items
    PUT shared-state.json → {"processed": 110}

  Worker B (simultaneously):
    GET shared-state.json → {"processed": 100}
    Process 15 items
    PUT shared-state.json → {"processed": 115}

  Result: {"processed": 115}
  Reality: 25 items processed (10 + 15)

  Lost update!
```

**Conditional updates help, but are limited**:

```
Better: Use If-Match

Worker A:
  GET shared-state.json → ETag: "abc123"
  Process 10 items
  PUT shared-state.json If-Match: "abc123"
  → Success

Worker B:
  GET shared-state.json → ETag: "abc123"
  Process 15 items
  PUT shared-state.json If-Match: "abc123"
  → 412 Precondition Failed (ETag changed)
  → Must retry

This prevents corruption but requires retry logic.
```

### Better Approach

**Use DynamoDB with atomic updates**:

```
DynamoDB:
  UPDATE state
  SET processed = processed + :count
  WHERE id = :id

  Atomic increment:
    ✓ No lost updates
    ✓ No retry needed
    ✓ Low latency
```

**Use SQS for coordination**:

```
Architecture:
  Queue (SQS) → Workers → Results (S3)

  No shared state needed:
    - Each worker pulls from queue
    - Processes independently
    - Writes results to S3(unique keys)
```

## When to Choose Alternatives

### Use Block Storage (EBS) When...

```
✓ Need POSIX filesystem
✓ Frequent small updates
✓ Random I/O patterns
✓ Low latency critical (<1ms)
✓ Single-instance access

Example: Database data files
```

### Use File System (EFS) When...

```
✓ Need shared filesystem across instances
✓ POSIX semantics required
✓ Many small files
✓ Directory operations

Example: Shared application code
```

### Use Database (RDS/DynamoDB) When...

```
✓ Need transactions
✓ Complex queries
✓ Joins
✓ Indexes
✓ High-frequency updates

Example: User accounts, orders, inventory
```

### Use Object Storage (S3) When...

```
✓ Immutable or rare updates
✓ Large files
✓ Simple key lookup
✓ Unlimited scale needed
✓ Durability critical
✓ Cost optimization important

Example: Backups, archives, media, data lakes
```

## The Decision Matrix

```
Requirement              EBS  EFS  RDS  DynamoDB  S3
──────────────────────   ───  ───  ───  ────────  ──
POSIX filesystem          ✓    ✓    ✗      ✗     ✗
Shared across instances   ✗    ✓    ✓      ✓     ✓
Unlimited scale           ✗    ✗    ✗      ✓     ✓
Low latency (<1ms)        ✓    ✓    ~      ✓     ✗
Transactions              ✗    ✗    ✓      ✓     ✗
Complex queries           ✗    ✗    ✓      ~     ✗
11 nines durability       ✗    ✗    ~      ✓     ✓
Cost (per GB)             $$   $$$  $$$$   $$    $
```

## Key Takeaways

- **Not a database** - Use RDS or DynamoDB for transactional data
- **Not a POSIX filesystem** - Use EBS or EFS for filesystem needs
- **Not for frequent updates** - Optimize for write-once, read-many
- **Not for tiny files** - Batch into larger objects
- **Not for complex directory ops** - Store metadata separately
- **Not for shared mutable state** - Use proper coordination primitives
- **Know the limits** - Request rates, consistency model, latency
- **Use the right tool** - Each storage type has its sweet spot

## What's Next?

We've explored where S3 shines and where it struggles. In the next chapter, we'll systematically review all the trade-offs inherent in S3's design.

---

**[← Previous: When Object Storage Shines](./15-when-it-shines.md) | [Back to Contents](./README.md) | [Next: The Trade-offs →](./17-tradeoffs.md)**
