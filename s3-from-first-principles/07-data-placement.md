# Chapter 7: Data Placement and Partitioning

> **"Where you put your data determines how well your system scales."**

## The Placement Problem

You have trillions of objects and thousands of servers. Where does each object go?

### The Challenge

**Requirements**:
- Distribute objects evenly across all servers
- Find any object in milliseconds
- Handle server additions/removals without disruption
- Avoid hotspots (overloaded servers)
- Maintain durability (multiple copies in different failure domains)
- Scale to unlimited servers

**Simple solutions don't work**:
```
Random placement:
→ Can't find objects later

Fixed assignment:
→ Can't add servers easily

Manual placement:
→ Doesn't scale
```

We need something smarter.

## Partitioning Fundamentals

### What is Partitioning?

**Partitioning** (sharding) divides data across multiple machines:

```
1 TB of data, 10 servers:
→ 100 GB per server

10 PB of data, 10,000 servers:
→ 1 TB per server

Same density, infinite scale!
```

### Partitioning Strategies

#### Strategy 1: Range Partitioning

**Divide by key ranges**:

```
Partition 1: "a" ... "d"
Partition 2: "e" ... "h"
Partition 3: "i" ... "m"
Partition 4: "n" ... "r"
Partition 5: "s" ... "z"
```

**Example**:
```
Object: "beach.jpg" → starts with "b"
→ Partition 1

Object: "sunset.jpg" → starts with "s"
→ Partition 5
```

**Pros**:
```
✓ Range queries fast (all "s*" keys on same partition)
✓ Prefix listing efficient
✓ Good for time-series data
```

**Cons**:
```
✗ Hotspots common (if keys cluster)
✗ Rebalancing complex (must split ranges)
✗ Unpredictable load distribution

Example hotspot:
  If everyone names files "photo1.jpg", "photo2.jpg"...
  → All go to Partition 4 ("p")
  → Overloaded!
```

#### Strategy 2: Hash Partitioning

**Divide by hash of key**:

```
Function: partition_id = hash(key) % num_partitions

Example with 10 partitions:
  "beach.jpg"  → MD5 = a3f2... → a3f2 % 10 = 2 → Partition 2
  "sunset.jpg" → MD5 = b8d4... → b8d4 % 10 = 4 → Partition 4
  "photo.jpg"  → MD5 = c1a9... → c1a9 % 10 = 9 → Partition 9
```

**Pros**:
```
✓ Even distribution (hash randomizes)
✓ No hotspots (assuming good hash function)
✓ Simple to implement
✓ Predictable performance
```

**Cons**:
```
✗ Range queries impossible (keys scattered)
✗ Adding partitions requires rehashing everything
✗ No locality (related keys on different partitions)
```

#### Strategy 3: Consistent Hashing

**The evolution of hash partitioning**:

Hash partitioning problem:
```
10 partitions: hash(key) % 10
Adding 11th partition: hash(key) % 11

Problem: Almost ALL keys change partition!
→ Massive data movement
→ System disruption
```

Consistent hashing solution:
```
Map both keys AND servers to the same hash space
→ Minimal movement when adding/removing servers
```

### Consistent Hashing Explained

**The hash ring**:

```
Hash space: 0 to 2^32-1 (32-bit hash)

Visualize as a ring:
         0
    ┌────────┐
   31       1
  30         2
  ...       ...
  16         16
    └────────┘

Servers and keys both map to points on the ring.
```

**Placing servers**:

```
Server A: hash("server-A") = 1000
Server B: hash("server-B") = 5000
Server C: hash("server-C") = 9000

Ring:
    0
    ├── Server A (1000)
    ├── Server B (5000)
    ├── Server C (9000)
    └── (wraps back to 0)
```

**Placing keys**:

```
Key "beach.jpg": hash = 3000
→ Walk clockwise from 3000
→ First server: Server B (5000)
→ Store on Server B

Key "sunset.jpg": hash = 7000
→ Walk clockwise from 7000
→ First server: Server C (9000)
→ Store on Server C

Key "photo.jpg": hash = 500
→ Walk clockwise from 500
→ First server: Server A (1000)
→ Store on Server A
```

**Adding a server**:

```
Add Server D: hash("server-D") = 3500

Before:
  Keys 3000-5000 → Server B

After:
  Keys 3000-3500 → Server D (new!)
  Keys 3500-5000 → Server B

Only a fraction of keys move!
  (vs 100% with simple hashing)
```

**Removing a server**:

```
Remove Server B (5000)

Before:
  Keys 1000-5000 → Server B

After:
  Keys 1000-5000 → Server C (next server clockwise)

Keys just move to next server.
```

### Virtual Nodes

**Problem with basic consistent hashing**:

```
3 servers → 3 points on ring
→ Uneven distribution (servers might cluster)

Example:
  Server A: 100
  Server B: 200
  Server C: 300

  Server A handles: 300-100 = 67% of ring!
  Server B handles: 100-200 = 23% of ring
  Server C handles: 200-300 = 10% of ring

Unbalanced!
```

**Solution: Virtual nodes**:

```
Each physical server maps to multiple points (vnodes):

Server A:
  vnode A1: hash("server-A-1") = 1000
  vnode A2: hash("server-A-2") = 5000
  vnode A3: hash("server-A-3") = 9000

Server B:
  vnode B1: hash("server-B-1") = 2000
  vnode B2: hash("server-B-2") = 6000
  vnode B3: hash("server-B-3") = 10000

Server C:
  vnode C1: hash("server-C-1") = 3000
  vnode C2: hash("server-C-2") = 7000
  vnode C3: hash("server-C-3") = 11000

Total: 9 vnodes spread evenly on ring
→ Each server handles ~33% of load
```

**Typical vnode count**: 100-256 vnodes per server

**Benefits**:
```
✓ Even load distribution
✓ Smoother rebalancing when adding/removing servers
✓ Can weight servers (more vnodes = more load)
```

## The Partition Map

### Mapping Objects to Servers

**The partition map** is the critical data structure:

```
┌──────────────────────────────────┐
│        Partition Map             │
├──────────────────────────────────┤
│ Hash Range    → Server(s)        │
├──────────────────────────────────┤
│ 0-1000        → [A, D, G]        │
│ 1001-2000     → [B, E, H]        │
│ 2001-3000     → [C, F, I]        │
│ 3001-4000     → [A, D, H]        │
│ ...                              │
└──────────────────────────────────┘
```

**For each partition**:
- Hash range it covers
- Primary server (for writes)
- Replica servers (for durability)
- Health status

### Lookup Process

**Finding an object**:

```
GET /bucket/beach.jpg

1. Compute key hash:
   hash("bucket/beach.jpg") = a3f2b8d4...
   → Partition ID = a3f2b8d4 % num_partitions = 874

2. Query partition map:
   Partition 874 → Servers [S1, S4, S9]

3. Choose server:
   - For write: Primary (S1)
   - For read: Any healthy server (S1, S4, or S9)

4. Send request to chosen server

5. Server returns object
```

**Lookup is fast**:
```
1. Hash: O(1) - microseconds
2. Partition map lookup: O(1) - hash table
3. Total: < 1 millisecond
```

### Partition Map Storage

**Where to store the partition map?**

```
Option 1: Every server has full copy
✓ Fast (local lookup)
✓ No network call
✗ Must keep in sync
✗ Memory usage (ok, map is small)

Option 2: Centralized service
✗ Network call (latency)
✗ Single point of failure
✓ Always consistent

Option 3: Hybrid (S3's approach)
✓ Cached locally on each server
✓ Updates propagated via gossip/pub-sub
✓ Fallback to central source if cache miss
✓ Best of both worlds
```

**Map size**:
```
1 million partitions:
  Per partition:
    - Hash range: 16 bytes
    - Servers: 3 × 8 bytes = 24 bytes
    - Metadata: 8 bytes
    - Total: ~48 bytes

  Total map size: 1M × 48 = 48 MB
  → Easily fits in RAM
```

## Load Balancing

### The Hotspot Problem

**Even with good partitioning, hotspots happen**:

```
Scenario: Popular object
  Object: "company-logo.png"
  Requests: 1 million/second

All requests hit same partition!
→ Server overloaded
→ Latency spikes
→ Failures
```

### Solution 1: Read Replicas

**Distribute reads across multiple replicas**:

```
Write:
  "logo.png" → Primary server (S1)
  Replicate to S4, S9

Read:
  "logo.png" → Choose randomly from [S1, S4, S9]
  → Load distributed across 3 servers
  → 3x read capacity
```

**Even better with more replicas**:
```
Hot object detection:
  If object requests > threshold:
    → Create extra replicas (4, 5, 6...)
    → Distribute reads across all

When load decreases:
    → Remove extra replicas
    → Return to normal (3 replicas)
```

### Solution 2: Caching

**Cache at multiple layers**:

```
┌──────────────┐
│  CloudFront  │  ← CDN edge caches (thousands of locations)
│   (CDN)      │
└──────┬───────┘
       │
┌──────┴───────┐
│   Caching    │  ← Regional caches (hundreds of servers)
│   Tier       │
└──────┬───────┘
       │
┌──────┴───────┐
│   Storage    │  ← Origin (actual S3 storage)
│   Tier       │
└──────────────┘

Hot object:
  - First request: Miss all caches → slow
  - Subsequent requests: Hit edge cache → fast
  - Origin: Protected from load spike
```

### Solution 3: Request Routing

**Intelligent request distribution**:

```
Load balancer observes:
  - Server CPU usage
  - Server network saturation
  - Server disk I/O
  - Request latency

Routes requests to least-loaded server:
  GET /hot-object
  → Server S4 is 90% loaded
  → Server S9 is 40% loaded
  → Route to S9
```

## Rebalancing

### When to Rebalance

**Triggers**:
```
1. Adding servers:
   - Increased capacity
   - New data center

2. Removing servers:
   - Hardware retirement
   - Cost optimization

3. Imbalanced load:
   - Some servers >80% full
   - Others <20% full

4. Failure domain changes:
   - Rack decommissioned
   - AZ added
```

### Rebalancing Process

**Goal**: Move data to achieve balance with minimal disruption.

**Steps**:

```
1. Compute new partition map:
   - Add/remove vnodes
   - Calculate new hash ranges
   - Assign new servers

2. Identify data to move:
   - Diff old map vs new map
   - List partitions that changed servers
   - Estimate data volume

3. Prioritize moves:
   - Critical partitions first (low replica count)
   - Balance speed vs impact
   - Rate limit (don't overwhelm network)

4. Execute moves:
   For each partition:
     a. Copy data to new server
     b. Verify integrity (checksums)
     c. Update metadata
     d. Switch reads to new server
     e. Delete old copy
     f. Update partition map

5. Monitor and adjust:
   - Track progress
   - Handle failures
   - Throttle if needed
```

### Rebalancing Challenges

**Challenge 1: Data volume**:

```
Moving 10 PB:
  Network: 10 Gbps
  Time: 10 PB / 1.25 GB/s = 92 days!

Can't wait 3 months.

Solution:
  - Parallel moves (1000 streams)
  - Incremental moves (background process)
  - Move only what's necessary
```

**Challenge 2: Service disruption**:

```
While moving partition:
  - Reads: From old or new server?
  - Writes: To old or new server?
  - Consistency: How to ensure?

Solution: Dual-write during transition
  1. Write to both old and new server
  2. Read from both (compare)
  3. After move complete, switch to new
  4. Delete from old
```

**Challenge 3: Partial failures**:

```
Move in progress → Server crashes
  - Move incomplete
  - Data in inconsistent state

Solution:
  - Idempotent moves (can retry safely)
  - Atomic commit (all or nothing)
  - Rollback capability
```

## Placement Policies

### Policy 1: Durability-First

**Maximize durability** (default):

```
For 3 replicas:
  - 1 replica in AZ-1, Rack A
  - 1 replica in AZ-2, Rack B
  - 1 replica in AZ-3, Rack C

Survives:
  ✓ Single disk failure
  ✓ Single server failure
  ✓ Single rack failure
  ✓ Single AZ failure
```

### Policy 2: Performance-First

**Minimize latency**:

```
For 3 replicas:
  - All in same AZ (low latency)
  - Different racks (some durability)

Survives:
  ✓ Single disk failure
  ✓ Single server failure
  ✓ Single rack failure
  ✗ AZ failure (data lost!)

Trade durability for performance.
```

Use case: Temporary data, caches

### Policy 3: Cost-First

**Minimize storage cost**:

```
For 3 replicas:
  - Use cheapest storage class
  - Accept higher latency
  - Fewer replicas (2 instead of 3)

Trade durability and performance for cost.
```

Use case: Archival data, backups

### Policy 4: Geo-Distributed

**Global availability**:

```
For 6 replicas:
  - 2 in US-East
  - 2 in US-West
  - 2 in Europe

Survives:
  ✓ Entire region failure
  ✓ Serves global users with low latency
```

Use case: Mission-critical data

## Capacity Planning

### Growth Prediction

**Tracking growth**:

```
Current: 100 PB
Growth: +10 PB/month

Predictions:
  6 months: 160 PB
  12 months: 220 PB
  24 months: 340 PB

When to add capacity?
  - At 70% full: Start adding servers
  - At 80% full: Urgently add servers
  - Never exceed 90% full
```

### Provisioning Strategy

**Just-in-time provisioning**:

```
Don't:
  - Over-provision (wasted cost)
  - Under-provision (performance issues)

Do:
  - Monitor growth trends
  - Provision 3-6 months ahead
  - Keep buffer (20% free capacity)
  - Automate procurement
```

### Decommissioning

**Removing old hardware**:

```
Typical server lifecycle: 3-5 years

Decommissioning process:
1. Mark server as "draining"
2. Stop new writes
3. Allow existing reads
4. Move data to new servers
5. Verify all data moved
6. Mark server as "retired"
7. Delete data
8. Remove from partition map
9. Physically remove hardware

Time: Weeks to months (careful process)
```

## Monitoring Placement Health

### Key Metrics

**Balance metrics**:
```
1. Storage utilization variance:
   - Std dev of % full across servers
   - Target: < 10%

2. Request rate variance:
   - Std dev of requests/second across servers
   - Target: < 20%

3. Partition size variance:
   - Std dev of partition sizes
   - Target: < 15%
```

**Durability metrics**:
```
1. Replica distribution:
   - % objects with replicas in 3+ AZs
   - Target: > 99.99%

2. Cross-rack distribution:
   - % objects with replicas in 3+ racks
   - Target: > 99.99%

3. Single-AZ exposure:
   - % objects with all replicas in 1 AZ
   - Target: < 0.01%
```

## Key Takeaways

- **Hash partitioning scales best** - Even distribution, predictable performance
- **Consistent hashing minimizes rebalancing** - Only affected keys move
- **Virtual nodes smooth distribution** - Prevents hotspots from clustering
- **Partition map is critical** - Fast, cached, replicated
- **Rebalancing is continuous** - Gradual process, not big bang
- **Placement policies matter** - Durability, performance, cost trade-offs
- **Monitoring is essential** - Detect imbalance early

## What's Next?

We know how to place objects across servers. But how are objects actually stored on disk? In the next chapter, we'll dive into the storage backend.

---

**[← Previous: The Durability Challenge](./06-durability-challenge.md) | [Back to Contents](./README.md) | [Next: The Storage Backend →](./08-storage-backend.md)**
