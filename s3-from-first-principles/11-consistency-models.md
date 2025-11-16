# Chapter 11: Consistency Models

> **"The consistency guarantee you choose determines what your system can and cannot do."**

## The Consistency Spectrum

Building a distributed storage system forces a fundamental choice:

**How consistent should reads be?**

### The CAP Theorem Reminder

```
CAP Theorem (Brewer's theorem):
  You can have at most 2 of 3:
    - Consistency: All nodes see same data
    - Availability: System always responds
    - Partition tolerance: Works despite network failures

For distributed systems across data centers:
  Partition tolerance is mandatory (networks fail)
  → Must choose: Consistency OR Availability
```

**S3's original choice**: Favor availability over consistency.

## Eventual Consistency (S3 2006-2020)

### What is Eventual Consistency?

**Definition**:
> "If no new updates are made, eventually all reads will return the latest write."

**In practice**:

```
Time    Write                   Read (from different replica)
────    ─────                   ────────────────────────────
t0      PUT /key = "v1"
t1      → Write to Replica A
t2      → Replicate to B        GET /key = "v1" ✓ (from A)
t3      → Replicate to C        GET /key = <old> ✗ (from C, not yet updated)
t4                              GET /key = "v1" ✓ (from C, now updated)

Reads during t0-t3: Might see old data
Reads after t4: Always see new data
```

**The window**: t0 to t4 = **inconsistency window** (typically milliseconds to seconds)

### Read-After-Write Consistency

**S3's original promise** (for new objects):

```
Scenario 1: New object (doesn't exist before)
  PUT /photos/new.jpg
  → Success
  GET /photos/new.jpg
  → Guaranteed to see new.jpg ✓

Scenario 2: Overwrite existing object
  PUT /photos/existing.jpg (v1)
  PUT /photos/existing.jpg (v2)
  → Success
  GET /photos/existing.jpg
  → Might see v1 or v2 (!)

Scenario 3: Delete then read
  DELETE /photos/old.jpg
  → Success
  GET /photos/old.jpg
  → Might still return the object (!)
```

**Why this model?**

```
Pros:
  ✓ High availability (always writable)
  ✓ Low latency (async replication)
  ✓ Survives network partitions
  ✓ Simpler implementation

Cons:
  ✗ Stale reads possible
  ✗ Delete not immediate
  ✗ Confusing for users
```

### How Eventual Consistency Worked

**Write path**:

```
Client: PUT /bucket/key = "value"

1. Request arrives at frontend
2. Frontend routes to primary replica (Replica A)
3. Replica A writes locally
4. Replica A acknowledges SUCCESS to client
5. Background: A replicates to B (async)
6. Background: A replicates to C (async)

Client sees success after step 4 (fast!)
Replicas B and C catch up later.
```

**Read path**:

```
Client: GET /bucket/key

1. Request arrives at frontend
2. Frontend picks random replica (A, B, or C)
3. Read from chosen replica
4. Return value

If replica hasn't received update yet → stale read
```

**Replication lag**:

```
Typical: 10-100 milliseconds
During high load: Seconds
During network issues: Minutes

Eventually: Always converges (if no more writes)
```

### Real-World Implications

**Use case 1: Static website**:

```
Scenario:
  1. Upload index.html (v1)
  2. User visits website
  3. Sees index.html (v1) ✓

  4. Upload index.html (v2) - updated
  5. User refreshes
  6. Might see v1 or v2 (random replica)

Result: User confused ("I uploaded it, why don't I see it?")
```

**Use case 2: Image upload**:

```
Scenario:
  1. PUT /profile-pic.jpg
  2. Success
  3. Immediately GET /profile-pic.jpg
  4. Might see: 404 Not Found (!)

Reason: GET routed to replica not yet updated

Workaround: Client should use read-after-write headers
  PUT with x-amz-expected-bucket-owner
  GET from same replica
```

**Use case 3: Backup and restore**:

```
Scenario:
  1. Backup job: List all objects
  2. Returns: [obj1, obj2, obj3]
  3. Someone uploads obj4
  4. Someone deletes obj2
  5. List again
  6. Returns: [obj1, obj2, obj3, obj4] or [obj1, obj3, obj4]

Result: Inconsistent listings during updates
```

## The Push for Strong Consistency

### Customer Pain Points

**Reported issues** (before Dec 2020):

```
1. "I uploaded a file but GET returns 404"
   → Read-after-write failure for overwrites

2. "I deleted a file but it's still there"
   → Eventual consistency for deletes

3. "My list is inconsistent"
   → Listings during updates

4. "My application has to retry/poll"
   → Workarounds for consistency

5. "I can't build ACID apps on S3"
   → No transactional guarantees
```

**Business impact**:
- Increased support costs
- Customer confusion
- Lost business (to competitors with stronger consistency)

### The Decision

**December 2020**: S3 announces **Strong Consistency** for all operations.

```
New guarantee:
  - PUT → immediately visible on GET
  - DELETE → immediately gone
  - LIST → reflects all completed PUTs/DELETEs

No extra cost.
No performance degradation.
How?!
```

## Strong Consistency (S3 2020+)

### What Changed?

**Strong consistency definition**:
> "Read always returns the result of the latest completed write."

**In practice**:

```
Time    Write                   Read
────    ─────                   ────
t0      PUT /key = "v1"
t1      → Write to all replicas (wait for quorum)
t2      ← Success returned
t3                              GET /key = "v1" ✓ (always!)
t4      PUT /key = "v2"
t5      → Write to all replicas
t6      ← Success returned
t7                              GET /key = "v2" ✓ (never v1!)

No inconsistency window!
```

### How Strong Consistency Works

**Quorum-based replication**:

```
Write quorum: W = 2 (out of 3 replicas)
Read quorum: R = 2 (out of 3 replicas)

Rule: W + R > N (total replicas)
  2 + 2 > 3 ✓

Guarantee: Read overlaps with Write
  → Always sees latest value
```

**Write path** (with quorum):

```
Client: PUT /bucket/key = "v2"

1. Frontend determines primary partition
2. Frontend sends to 3 replicas in parallel:
   - Replica A
   - Replica B
   - Replica C

3. Wait for 2 acknowledgments (quorum)
   - Replica A: ACK (10ms)
   - Replica B: ACK (12ms)
   - Replica C: (waiting... 50ms network delay)

4. After 2 ACKs: Return success to client
   Total latency: 12ms (2nd ACK)

5. Replica C eventually catches up (async)
```

**Read path** (with quorum):

```
Client: GET /bucket/key

1. Frontend sends to 2+ replicas in parallel:
   - Replica A
   - Replica B

2. Receive responses:
   - Replica A: "v2", timestamp=1640000000
   - Replica B: "v2", timestamp=1640000000

3. Compare:
   - Same value → Return "v2"
   - Different values → Return newest (by timestamp)

4. If different: Read-repair (update stale replica)

Latency: ~15ms (wait for 2nd response)
```

### Version Vectors and Timestamps

**Each write has metadata**:

```
Write metadata:
  - Value: "v2"
  - Timestamp: 1640000000 (Unix epoch)
  - Version: 123456 (monotonic counter)
  - Writer ID: frontend-42

On read:
  If replicas disagree:
    → Choose highest timestamp
    → Or highest version
    → Repair stale replica
```

**Conflict resolution**:

```
Scenario: Concurrent writes
  t0: Writer A: PUT /key = "A"
  t0: Writer B: PUT /key = "B" (same time!)

Resolution:
  - Timestamp same → Use version number
  - Version same → Use writer ID (deterministic)

Result: One value wins, system is consistent
```

### Impact on Performance

**Before** (eventual consistency):
```
Write latency: 10ms (1 replica ACK)
Read latency: 10ms (1 replica)
Availability: 99.99%
```

**After** (strong consistency):
```
Write latency: 15ms (2 replica ACKs)
Read latency: 15ms (2 replicas)
Availability: 99.99%

Increase: 5ms (50% slower)
But: Customers accept it for consistency!
```

**How S3 minimized impact**:

```
1. Parallel writes/reads:
   - Don't wait for serial ACKs
   - Wait for 2nd of parallel requests

2. Optimized networking:
   - Co-locate replicas in same AZ when possible
   - Low-latency cross-AZ links

3. Caching:
   - Cache metadata aggressively
   - Reduce repeated lookups

4. Request pipelining:
   - Batch operations
   - Amortize latency overhead
```

## Listing Consistency

### The Listing Challenge

**Lists are tricky**:

```
Scenario:
  Bucket has: [obj1, obj2, obj3]

  During list:
    t0: Start LIST
    t1: Read obj1, obj2 from Replica A
    t2: Someone adds obj4
    t3: Someone deletes obj2
    t4: Read obj3, obj4 from Replica B

  What should LIST return?
    Option 1: [obj1, obj2, obj3] (snapshot at t0)
    Option 2: [obj1, obj3, obj4] (latest state)
    Option 3: [obj1, obj2, obj3, obj4] (inconsistent!)
```

### S3's Listing Guarantee

**Strong consistency for listings** (since Dec 2020):

```
Guarantee:
  LIST reflects all PUTs and DELETEs that completed
  BEFORE the LIST request started.

Implementation:
  - Use version numbers
  - Read from consistent replica set
  - Apply versioning to filter results

Result:
  - No phantom objects
  - No missing objects (unless added during list)
  - Consistent snapshot
```

**List pagination**:

```
Request 1: LIST (first page)
  Returns: [obj1, obj2, obj3], Token="obj3", Version=123

Request 2: LIST (next page, token="obj3", version=123)
  Returns: [obj4, obj5, obj6], Token="obj6", Version=123

Version 123 ensures:
  - Both pages from same snapshot
  - No obj added between pages appears unexpectedly
  - No obj deleted between pages disappears unexpectedly
```

## Conditional Operations

### Optimistic Concurrency Control

**Use ETags for conditional writes**:

```
Scenario: Two users editing same object

User A:
  1. GET /doc.txt → ETag: "abc123"
  2. Edit locally
  3. PUT /doc.txt
     If-Match: "abc123"
     → Success (no one else modified)

User B:
  1. GET /doc.txt → ETag: "abc123"
  2. Edit locally
  3. PUT /doc.txt
     If-Match: "abc123"
     → Failure! (A already modified)
     → Re-fetch and retry
```

**Prevents lost updates**:

```
Without If-Match:
  A writes → success
  B writes → success (overwrites A's changes!)
  A's work lost!

With If-Match:
  A writes → success
  B writes → failure (detects conflict)
  B refetches, merges, retries
  No data loss!
```

### Conditional Reads

**If-None-Match** (for caching):

```
Client: GET /image.jpg
        If-None-Match: "abc123"

S3:
  Current ETag: "abc123"
  → Returns 304 Not Modified (no body)

Client:
  → Use cached version

Saves bandwidth and latency!
```

## Consistency vs Availability

### Network Partitions

**What if network partitions?**

```
Scenario: AZ-1 loses connectivity to AZ-2, AZ-3

Before partition:
  3 replicas: AZ-1 (A), AZ-2 (B), AZ-3 (C)

After partition:
  Side 1: A (alone)
  Side 2: B, C

Write attempt from Side 1:
  Needs W=2 acknowledgments
  Only has 1 replica (A)
  → WRITE FAILS (unavailable)

Write attempt from Side 2:
  Needs W=2 acknowledgments
  Has 2 replicas (B, C)
  → WRITE SUCCEEDS ✓

Read attempt from Side 1:
  Needs R=2 responses
  Only has 1 replica (A)
  → READ FAILS (unavailable)
```

**Result**:
- Consistency maintained (no divergence)
- But availability reduced (minority partition can't serve)

**CAP theorem in action**: Chose consistency over availability during partitions.

### Split-Brain Prevention

**How S3 prevents split-brain**:

```
Problem (without prevention):
  Partition: [A] | [B, C]
  Both sides accept writes
  Network heals
  → Conflicting data!

S3's solution: Partition tolerance via quorum
  Minority side: Can't form quorum
  → Rejects writes
  Majority side: Can form quorum
  → Accepts writes

When network heals:
  Minority replicates from majority
  No conflicts!
```

## The Evolution Timeline

**2006-2020**: Eventual consistency
```
Pros: High availability, low latency
Cons: Stale reads, customer confusion
```

**Dec 2020**: Strong consistency
```
Pros: No stale reads, simpler applications
Cons: Slightly higher latency, reduced availability during partitions
```

**Why it took 14 years**:

```
Challenges:
  1. Massive scale (trillions of objects)
     → Quorum coordination overhead

  2. Performance requirements (millions QPS)
     → Can't add too much latency

  3. Global distribution (100+ AZs)
     → Network delays across regions

  4. Backward compatibility
     → Can't break existing applications

  5. Zero-downtime migration
     → Can't shut down for upgrade

Solution: Multi-year engineering effort
  - New metadata index
  - Optimized quorum protocols
  - Gradual rollout
  - Extensive testing
```

## Best Practices

### For Application Developers

**Leverage strong consistency**:

```
1. Upload then immediately read:
   PUT /file
   GET /file  → Always works now! ✓

2. Delete then verify:
   DELETE /file
   GET /file  → Always 404 now! ✓

3. List after upload:
   PUT /new-file
   LIST /prefix  → new-file appears! ✓
```

**Use conditional operations**:

```
1. Prevent overwrites:
   PUT /file If-None-Match: *
   → Fails if file exists

2. Prevent lost updates:
   PUT /file If-Match: "<etag>"
   → Fails if file changed

3. Efficient caching:
   GET /file If-None-Match: "<etag>"
   → Returns 304 if unchanged
```

**Handle failures gracefully**:

```
1. Retry transient failures:
   → 503 Service Unavailable (temporary)

2. Don't retry permanent failures:
   → 412 Precondition Failed (conditional failure)

3. Use exponential backoff:
   → Avoid overwhelming system
```

## Key Takeaways

- **Consistency is a spectrum** - From eventual to strong
- **CAP theorem is real** - Must choose consistency or availability during partitions
- **Quorum enables strong consistency** - W + R > N guarantees overlap
- **Strong consistency has costs** - Slightly higher latency, reduced availability
- **S3 evolved over time** - 14 years from eventual to strong
- **Conditional operations prevent conflicts** - Use If-Match and If-None-Match
- **Trade-offs are intentional** - Strong consistency worth the costs for most use cases

## What's Next?

Consistency protects against concurrent writes. But what about accidental deletes? In the next chapter, we'll explore versioning and time travel.

---

**[← Previous: Metadata at Scale](./10-metadata-at-scale.md) | [Back to Contents](./README.md) | [Next: Versioning and Time Travel →](./12-versioning.md)**
