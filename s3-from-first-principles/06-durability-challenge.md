# Chapter 6: The Durability Challenge

> **"Durability isn't about if failures happen—it's about ensuring they don't matter."**

## The 11 Nines Promise

Your manager reminds you:

> *"99.999999999% durability. That's not negotiable. Customer data is sacred."*

Let's understand what this actually means.

### The Math

**99.999999999% durability** (11 nines):

```
Annual loss rate: 0.00000000001
= 1 in 100 billion objects per year

For 10 billion objects:
Expected loss: 0.1 objects per year
= 1 object every 10 years

For 10 trillion objects:
Expected loss: 100 objects per year
= Still 99.999999999% durability
```

**Perspective**:
```
99% durability:
→ Lose 1 in 100 objects per year
→ Unacceptable

99.999% (5 nines):
→ Lose 1 in 100,000 objects per year
→ Better, but not enough

99.999999999% (11 nines):
→ Lose 1 in 100 billion objects per year
→ This is our target
```

## Understanding Failure Modes

### Hardware Failures

**Disk failures** (most common):

```
Typical hard drive:
- Mean Time Between Failures (MTBF): 1 million hours
- Annual failure rate (AFR): ~4%

For 1000 disks:
- Failures per year: 40 disks
- Failures per month: 3.3 disks
- Failures per day: 0.11 disks

For 100,000 disks:
- Failures per day: 11 disks
- Failures per hour: 0.45 disks
→ A disk fails every 2 hours!
```

**Disk failures are constant.**

**Server failures**:
```
Typical server uptime: 99.9%
= 8.76 hours downtime per year

For 1000 servers:
- At any moment: ~1 server is down
- Reboots, crashes, hardware issues

Failures are random and unpredictable.
```

**Other hardware failures**:
```
- Network cards: ~1% AFR
- Power supplies: ~0.5% AFR
- Memory: Bit flips (cosmic rays!)
- CPUs: Rare but happen
```

### Correlated Failures

**Rack failures**:
```
Causes:
- Power distribution unit (PDU) failure
- Top-of-rack (ToR) switch failure
- Cooling failure

Impact: All servers in rack go down
Frequency: Rare but significant
```

**Data center failures**:
```
Causes:
- Power outage (entire DC)
- Network partition
- Natural disasters
- Human error (accidental cable cut)

Impact: Entire DC unavailable
Frequency: Very rare but catastrophic
```

**Correlated failures are the enemy of 11 nines.**

### Silent Data Corruption

**Bit rot**:
```
Cosmic rays, magnetic decay, firmware bugs
→ Data changes without warning

Rate: ~1 error per 10^17 bits read
= 1 error per 12.5 PB read

For 1 EB (exabyte) of data:
- ~80 errors introduced naturally per year

If undetected: Silent data loss!
```

**This is why checksums are critical.**

## Failure Domain Isolation

### What is a Failure Domain?

**A failure domain** is a group of resources that can fail together:

```
┌─────────────────────────────────────┐
│    Data Center (Availability Zone)  │  ← Largest failure domain
├─────────────────────────────────────┤
│  ┌───────────────────────────────┐  │
│  │         Rack                  │  │  ← Medium failure domain
│  ├───────────────────────────────┤  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  Server               │  │  │  ← Small failure domain
│  │  ├─────────────────────────┤  │  │
│  │  │  Disk                  │  │  │  ← Smallest failure domain
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Isolation Strategy

**Rule**: Never put all copies in the same failure domain.

**Bad** (all replicas in same rack):
```
Rack 1:
- Server A: Copy 1
- Server B: Copy 2
- Server C: Copy 3

Rack PDU fails → All copies lost!
```

**Good** (replicas across racks):
```
Rack 1:
- Server A: Copy 1

Rack 2:
- Server B: Copy 2

Rack 3:
- Server C: Copy 3

One rack fails → 2 copies remain
```

**Best** (replicas across availability zones):
```
AZ 1 (Data Center 1):
- Server A: Copy 1

AZ 2 (Data Center 2):
- Server B: Copy 2

AZ 3 (Data Center 3):
- Server C: Copy 3

One DC fails → 2 copies remain in other DCs
```

## Replication Strategies

### Strategy 1: Simple Replication

**3x replication** (naive approach):

```
Write process:
1. Client sends object
2. Write to Server A
3. Write to Server B
4. Write to Server C
5. Return success

Read process:
1. Read from any of [A, B, C]
2. Return data
```

**Durability calculation**:
```
Single disk AFR: 4%
Survival rate: 96%

Three independent disks:
Probability all three fail in one year:
= 0.04 × 0.04 × 0.04
= 0.000064
= 0.0064%

Durability: 99.9936%
= Only 4 nines!

Not enough for 11 nines.
```

**Why not enough?**
- Doesn't account for correlated failures
- Doesn't account for bit rot
- Doesn't account for operator errors
- Rebuild time matters

### Strategy 2: Asynchronous Replication

**Write immediately, replicate in background**:

```
Write process:
1. Client sends object
2. Write to Server A (primary)
3. Acknowledge success to client
4. Async: Replicate to B
5. Async: Replicate to C

Speed: Fast (1 write latency)
Risk: If A fails before replication completes, data lost
```

**Durability-speed tradeoff**:
```
Synchronous (wait for all replicas):
  Latency: 3x single write
  Durability: Best

Async (return after first write):
  Latency: 1x single write
  Durability: Worst (window of vulnerability)

Quorum (wait for majority):
  Latency: 2x single write
  Durability: Good compromise
```

### Strategy 3: Quorum Writes

**Wait for majority** (2 out of 3):

```
Write process:
1. Send to A, B, C in parallel
2. Wait for 2 acknowledgments
3. Return success
4. Third replica catches up later

Read with quorum:
1. Read from 2 replicas
2. If they match: return data
3. If they differ: use newer (by timestamp)
4. Repair the stale replica
```

**Durability**:
```
2 out of 3 must survive:

Scenarios where data survives:
- All 3 survive: Yes
- 2 survive: Yes
- 1 survives: No

Probability of losing data:
= Probability exactly 0 or 1 survive

Math:
P(all 3 fail) = 0.04^3 = 0.000064
P(exactly 2 fail) = C(3,2) × 0.04^2 × 0.96 = 0.004608
P(data loss) = 0.000064 + 0.004608 = 0.004672
= 0.4672%

Still only 99.5% durability!
```

**Still not 11 nines. We need more.**

## Multi-Layer Redundancy

### Layer 1: Local Redundancy

**Within a data center**:

```
Use erasure coding (8+4 scheme):
- Split object into 8 data chunks
- Generate 4 parity chunks
- Store 12 chunks on different servers
- Can survive any 4 failures

Durability improvement:
- vs 3x replication: Better
- Storage overhead: 1.5x vs 3x
- Cost: 50% savings!
```

**More on erasure coding in Chapter 9.**

### Layer 2: Cross-AZ Redundancy

**Replicate across availability zones**:

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│      AZ-1       │  │      AZ-2       │  │      AZ-3       │
├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│ Erasure coded   │  │ Erasure coded   │  │ Erasure coded   │
│ chunks (4)      │  │ chunks (4)      │  │ chunks (4)      │
└─────────────────┘  └─────────────────┘  └─────────────────┘

Each AZ stores 4 of 12 chunks.
Any 8 of 12 chunks can reconstruct object.
Can lose entire AZ and still have data!
```

**Durability calculation**:
```
AZ failure rate: 0.001% per year (rare but possible)

With data in 3 AZs:
Need to lose 2 AZs simultaneously to lose data

P(2 AZs fail simultaneously) ≈ 10^-12
= 11 nines!
```

### Layer 3: Geographic Redundancy

**For critical data, replicate across regions**:

```
US-East (primary):
- Full erasure-coded replica

US-West (backup):
- Full erasure-coded replica

Europe (backup):
- Full erasure-coded replica

Can survive:
- Entire region failure
- Natural disasters
- Regional network partitions
```

**Durability**: Beyond 11 nines (12-13 nines possible)

## Data Integrity Verification

### Checksums Everywhere

**Why checksums?**

```
Without checksums:
1. Write data to disk
2. Later: read data from disk
3. Data might be corrupted (bit rot)
4. Return corrupted data to user
5. User doesn't know it's corrupted!

With checksums:
1. Write data + MD5(data) to disk
2. Later: read data from disk
3. Compute MD5(read_data)
4. Compare: stored_MD5 == computed_MD5?
5. If match: data is good
6. If mismatch: detect corruption, use replica
```

**The checksum strategy**:

```
On write:
1. Client computes MD5 of object
2. Server computes MD5 of received data
3. Compare: if mismatch, reject (network corruption)
4. Store: object data + MD5
5. Return ETag = MD5 to client

On read:
1. Read object data + MD5 from disk
2. Compute MD5 of read data
3. Compare with stored MD5
4. If mismatch: data corrupted!
   → Try another replica
   → Log for repair
5. If match: return data to client
```

### Continuous Scrubbing

**Problem**: Bit rot happens slowly over time.

**Solution**: Proactively scan all data.

```
Scrubber process (runs continuously):
1. For each object:
   a. Read data from disk
   b. Compute checksum
   c. Compare with stored checksum
   d. If mismatch:
      → Mark replica as corrupted
      → Create new replica from good copy
      → Delete corrupted replica

2. Repeat for all objects
3. Full scan every N months

For 1 EB (exabyte):
- Scan rate: 1 PB/day
- Full scan: ~1000 days (~3 years)
- But finds corruption before it matters
```

**Cost**:
```
Reading 1 PB/day:
- Network: ~100 Gbps
- CPU: Minimal (checksums are fast)
- Benefit: Prevent silent data loss

Worth it for 11 nines!
```

## Recovery and Rebuild

### Detecting Failures

**Heartbeat monitoring**:

```
Every server:
1. Sends heartbeat every 10 seconds
2. Central monitor tracks heartbeats
3. If no heartbeat for 30 seconds:
   → Server marked as failed

Fast detection: Within 30 seconds
```

### Rebuild Process

**When a disk fails**:

```
1. Detection:
   - Server reports disk failure
   - Or: Heartbeat timeout

2. Assessment:
   - Which objects were on failed disk?
   - Query metadata index
   - List: [obj1, obj2, ..., objN]

3. Rebuild:
   For each object:
   a. Count existing replicas
   b. If < target replicas:
      → Copy from good replica
      → Create new replica on healthy disk
   c. Update metadata index

4. Completion:
   - Log rebuild stats
   - Monitor: rebuild rate, errors
```

**Rebuild rate**:
```
Single disk: 2 TB
Network bandwidth: 1 Gbps = 125 MB/s
Time to rebuild: 2 TB / 125 MB/s = 4.5 hours

For 100,000 disks:
- ~11 disk failures per day
- ~49.5 hours of rebuild work per day
- Need parallel rebuilds!

Solution: Distribute rebuild across many servers
```

### Rebuild Storms

**Problem**: Multiple failures during rebuild.

```
Scenario:
1. Disk 1 fails
2. Start rebuild (reading from Disks 2-10)
3. Heavy read load on Disks 2-10
4. One of them fails due to stress!
5. Now need to rebuild 2 disks
6. More stress...
7. Cascade failure?

This is a real risk.
```

**Mitigation**:

```
1. Rate limiting:
   - Don't rebuild too fast
   - Throttle to 70% of bandwidth

2. Prioritization:
   - Objects with fewer replicas first
   - Critical data first

3. Overprovisioning:
   - Extra replicas during rebuild
   - Temporary 4x replication

4. Monitoring:
   - Track rebuild progress
   - Alert on slow rebuilds
```

## The Durability Budget

### Calculating Actual Durability

**All failure modes combined**:

```
1. Disk failure: 4% AFR
   With 3 AZ erasure coding: 99.99999% durability

2. Silent corruption: ~1 error per 12.5 PB read
   With checksums + scrubbing: 99.999999% durability

3. Operator error: ~1% of data loss incidents
   With versioning + soft delete: 99.9999% durability

4. Software bugs: Rare but happen
   With extensive testing: 99.9999% durability

5. Disaster (entire region): Very rare
   With cross-region replication: 99.99999999% durability

Combined (multiply survival rates):
= 0.9999999 × 0.99999999 × 0.999999 × 0.999999 × 0.9999999999
≈ 99.999999% durability

Still not quite 11 nines...
```

### Getting to 11 Nines

**Additional measures**:

```
1. More replicas:
   - Use 12+4 erasure coding (vs 8+4)
   - Higher overhead, but higher durability

2. Faster rebuild:
   - Reduce window of vulnerability
   - More aggressive rebuild

3. Better hardware:
   - Enterprise disks (lower AFR)
   - Cost tradeoff

4. More availability zones:
   - 4 or 5 AZs instead of 3
   - Geographic diversity

5. Defense in depth:
   - Multiple layers
   - No single point of failure
```

**The real secret**: Combine all of these.

## Monitoring and Alerting

### Durability Metrics

**Key metrics to track**:

```
1. Object loss rate:
   - Objects lost per billion per year
   - Target: < 0.01

2. Replica health:
   - % of objects with sufficient replicas
   - Target: > 99.999%

3. Rebuild rate:
   - Objects/second being rebuilt
   - Target: > disk failure rate

4. Checksum failures:
   - Corrupted objects detected
   - Target: All detected and repaired

5. Mean time to repair (MTTR):
   - Time from failure to full rebuild
   - Target: < 24 hours
```

### Alerting

**Critical alerts**:

```
1. Replica count low:
   - Any object < 2 replicas
   - Immediate action required

2. Rebuild queue growing:
   - Failures faster than rebuilds
   - Add resources

3. Checksum failures increasing:
   - Possible bad batch of disks
   - Investigate hardware

4. AZ unreachable:
   - Network partition or outage
   - Critical durability risk
```

## The Tradeoffs

### Durability vs Cost

```
Durability       Storage Overhead    Cost/GB
---------        ----------------    -------
99% (2 nines)    1.0x (single copy)  $0.01
99.99% (4 nines) 3.0x (triple rep)   $0.03
99.999999% (8)   4.5x (EC 8+4)       $0.045
99.999999999%    6.0x (EC 12+4)      $0.06
11 nines+        6.0x + cross-region $0.10+

We can't afford maximum durability everywhere.
```

**Solution**: Storage classes (Chapter 13)

### Durability vs Performance

```
Higher durability requires:
- More replicas → Slower writes
- More checks → Slower reads
- More scrubbing → More background load

Tradeoff: Acceptable latency increase for 11 nines
```

### Durability vs Consistency

```
Stronger consistency requires:
- Synchronous replication
- Quorum reads/writes
- Higher latency

Weaker consistency allows:
- Async replication
- Single replica reads
- Lower latency

S3's choice: Eventual consistency (initially)
  → Optimizes for durability and performance
  → Chapter 11 covers evolution to strong consistency
```

## Key Takeaways

- **11 nines is extreme** - Requires multiple layers of redundancy
- **Correlated failures are the enemy** - Isolation across failure domains essential
- **Checksums are mandatory** - Detect and repair silent corruption
- **Continuous scrubbing needed** - Proactive detection of bit rot
- **Fast rebuild is critical** - Minimize window of vulnerability
- **No single technique suffices** - Defense in depth is the strategy
- **Durability costs money** - Tradeoff between durability and cost

## What's Next?

We know we need multiple replicas across failure domains. But with trillions of objects, how do we decide where to place each one? In the next chapter, we'll explore data placement and partitioning.

---

**[← Previous: Keys, Buckets, and Namespaces](./05-keys-buckets-namespaces.md) | [Back to Contents](./README.md) | [Next: Data Placement and Partitioning →](./07-data-placement.md)**
