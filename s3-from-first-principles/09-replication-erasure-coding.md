# Chapter 9: Replication and Erasure Coding

> **"The difference between 99% and 99.999999999% durability is the difference between art and mathematics."**

## The Durability-Cost Tradeoff

You need 11 nines of durability, but at pennies per gigabyte. These requirements conflict.

### The Economics

**Simple replication**:
```
Store 3 copies:
  User data: 1 TB
  Actual storage: 3 TB
  Cost multiplier: 3x

For $0.01/GB target:
  Raw storage cost: $0.003/GB
  Impossible! (Disks cost ~$0.02/GB)
```

**We need a better way.**

## Replication Strategies

### Strategy 1: Simple Replication

**How it works**:

```
Write object:
  1. Send data to Server A
  2. Server A writes to local disk
  3. Server A sends to Server B
  4. Server B writes to local disk
  5. Server B sends to Server C
  6. Server C writes to local disk
  7. Acknowledge success

┌──────────┐
│ Server A │ ── Copy ──> Server B ── Copy ──> Server C
│ Copy 1   │             Copy 2              Copy 3
└──────────┘
```

**Durability math**:
```
Assumptions:
  - Disk AFR (Annual Failure Rate): 4%
  - Disks fail independently
  - Mean Time To Repair (MTTR): 24 hours

Probability of data loss:
  = P(all 3 disks fail within MTTR)
  = P(2 disks already failed) × P(3rd fails before repair)

  P(2 failures) = C(3,2) × 0.04² × 0.96 = 0.004608
  P(3rd failure in 24h) = 0.04 / 365 = 0.0001096
  P(data loss) ≈ 0.004608 × 0.0001096 = 0.0000005
  = 0.00005%

Durability: 99.99995% (only ~5 nines)
Not enough!
```

**Storage overhead**: 3x (200% overhead)

**Conclusion**: Simple 3x replication is expensive and insufficient.

### Strategy 2: Higher Replication Factor

**Use 5 copies instead of 3**:

```
Durability improves:
  P(all 5 fail) ≈ 0.04^5 = 0.00000001024
  = 99.999999% durability (~8 nines)

Better, but still not 11 nines.

Storage overhead: 5x (400% overhead)
Cost: Prohibitive!
```

**We need a mathematical breakthrough.**

## Enter Erasure Coding

### The Core Idea

**Instead of copying**, use mathematics to create redundancy.

**Analogy**: RAID 5/6, but distributed across servers.

**Basic concept**:
```
Take N data chunks
Generate M parity chunks
Store N+M total chunks
Can recover from any M failures
```

**Example** (4+2 scheme):
```
Original object: 4 MB

Split into 4 data chunks: D1, D2, D3, D4 (each 1 MB)
Generate 2 parity chunks: P1, P2 (each 1 MB)

Store 6 chunks total: D1, D2, D3, D4, P1, P2

Can survive any 2 chunk failures:
  - Lose D1, D2 → Recover from D3, D4, P1, P2
  - Lose D3, P1 → Recover from D1, D2, D4, P2
  - Lose P1, P2 → Still have all data (D1-D4)

Storage overhead: 6/4 = 1.5x (50% overhead)
  vs 3x for triple replication (200% overhead)

Savings: 50%!
```

### The Mathematics: Reed-Solomon Codes

**Reed-Solomon encoding** is the algorithm:

**How it works** (simplified):

```
1. Represent data as polynomials over a finite field

Original data: [D1, D2, D3, D4]

2. Create polynomial:
   P(x) = D1 + D2·x + D3·x² + D4·x³

3. Evaluate polynomial at different points:
   P(0) = D1
   P(1) = D1 + D2 + D3 + D4
   P(2) = D1 + 2·D2 + 4·D3 + 8·D4
   ...

4. First N evaluations are data chunks (D1-D4)
   Next M evaluations are parity chunks (P1-P2)

5. Any N evaluations can reconstruct the polynomial
   → Can reconstruct all data
```

**Real implementation uses Galois Field arithmetic (GF(2^8))**:
- Addition is XOR
- Multiplication uses lookup tables
- Fast in hardware

**Key property**: Any k chunks can recover the original data, where k = number of data chunks.

### Common Erasure Coding Schemes

**For S3-scale storage**:

```
Scheme   Data  Parity  Total  Overhead  Can Survive
------   ----  ------  -----  --------  -----------
4+2       4      2      6      50%       2 failures
6+3       6      3      9      50%       3 failures
8+4       8      4     12      50%       4 failures
10+4     10      4     14      40%       4 failures
12+4     12      4     16      33%       4 failures

Compare to replication:
3x replication: 200% overhead, survives 2 failures
```

**S3's typical scheme**: 8+4 (or variations)
- 50% overhead
- Survives 4 failures
- Good balance

## Erasure Coding Implementation

### Encoding Process

**Storing a 4 MB object with 8+4 scheme**:

```
1. Split object into 8 data chunks:
   D1, D2, D3, D4, D5, D6, D7, D8
   Each chunk: 512 KB

2. Apply Reed-Solomon encoding:
   Generate 4 parity chunks: P1, P2, P3, P4
   Each chunk: 512 KB

3. Distribute 12 chunks across servers:
   Server 1:  D1
   Server 2:  D2
   Server 3:  D3
   ...
   Server 8:  D8
   Server 9:  P1
   Server 10: P2
   Server 11: P3
   Server 12: P4

4. Update metadata index:
   "bucket/object" → [S1, S2, ..., S12]

Total stored: 6 MB (4 MB × 1.5)
```

### Decoding Process

**Reading the object**:

```
Normal case (all chunks available):
  1. Read D1-D8 from servers 1-8
  2. Concatenate: D1 + D2 + ... + D8
  3. Return object

Degraded case (some chunks unavailable):
  Scenario: Servers 3 and 5 are down

  1. Read available chunks:
     D1, D2, D4, D6, D7, D8, P1, P2, P3, P4
     (10 chunks, need only 8)

  2. Apply Reed-Solomon decoding:
     Reconstruct D3 and D5

  3. Concatenate: D1 + D2 + D3 + ... + D8

  4. Return object

Latency impact:
  Normal: ~1 network round-trip
  Degraded: ~1 network round-trip + decoding CPU
  (Still fast!)
```

### Chunk Placement

**Critical: Place chunks in different failure domains**:

```
Bad placement (all in same rack):
  Rack 1: D1, D2, D3, D4, P1, P2, P3, P4 ✗
  → Rack failure = data loss!

Good placement (across racks):
  Rack 1: D1, D2, D3
  Rack 2: D4, D5, D6
  Rack 3: D7, D8, P1
  Rack 4: P2, P3, P4
  → Can survive 1 rack failure ✓

Best placement (across AZs):
  AZ-1: D1, D2, D3, D4
  AZ-2: D5, D6, P1, P2
  AZ-3: D7, D8, P3, P4
  → Can survive 1 AZ failure ✓
```

**Placement algorithm**:
```
For each chunk:
  1. Exclude servers in same rack as other chunks
  2. Exclude servers in same AZ (if possible)
  3. Choose least-loaded eligible server
  4. Store chunk
```

## Durability Analysis

### Erasure Coding Durability

**For 8+4 scheme across 3 AZs**:

```
Scenario 1: Random disk failures
  Need 5+ failures (out of 12) to lose data

  P(5 failures in 1 year):
    With 4% AFR per disk:
    = C(12,5) × 0.04^5 × 0.96^7
    ≈ 0.0000000012
    = 99.9999999988% durability
    ≈ 11 nines ✓

Scenario 2: Entire AZ failure
  AZ-1 fails: Lose D1, D2, D3, D4 (4 chunks)
  Remaining: D5, D6, D7, D8, P1, P2, P3, P4 (8 chunks)
  Can still reconstruct! ✓

  Need 2 AZs to fail simultaneously:
    P(2 AZ failures) ≈ 0.001% × 0.001% = 0.00000001%
    = 99.99999999% durability ✓

Combined durability: ~11 nines ✓
```

**At 50% overhead instead of 200%!**

### Comparing Durability

```
Scheme              Overhead    Durability    Cost/TB
-----------------   --------    ----------    -------
Single copy         0%          ~96%          $0.01
2x replication      100%        ~99.9%        $0.02
3x replication      200%        ~99.995%      $0.03
5x replication      400%        ~99.9999%     $0.05
8+4 erasure coding  50%         ~99.999999999% $0.015

Winner: Erasure coding!
```

## Performance Implications

### Write Performance

**Replication**:
```
Write path:
  1. Send to Server 1: 50ms
  2. Server 1 → Server 2: 50ms
  3. Server 2 → Server 3: 50ms
  Total: 150ms (sequential)

Or parallel:
  1. Send to all 3 servers in parallel: 50ms
  Total: 50ms (parallel)
```

**Erasure coding**:
```
Write path:
  1. Split into chunks: <1ms (CPU)
  2. Encode (compute parity): ~10ms (CPU)
  3. Send 12 chunks in parallel: 50ms (network)
  Total: ~61ms

Slightly slower, but acceptable.
```

**Optimization**: Pipeline encoding and sending.

### Read Performance

**Replication** (normal case):
```
Read from any of 3 servers: 50ms
```

**Erasure coding** (normal case):
```
Read 8 data chunks in parallel: 50ms
Concatenate: <1ms
Total: ~51ms

Nearly identical!
```

**Erasure coding** (degraded case):
```
Scenario: 2 chunks unavailable

Read 8 chunks (instead of full 12): 50ms
Decode missing chunks: ~5ms (CPU)
Total: ~55ms

Still acceptable!
```

### CPU Overhead

**Encoding/decoding cost**:

```
For 8+4 scheme on 4 MB object:

Encoding:
  - Reed-Solomon encode: ~10ms on modern CPU
  - ~400 MB/s throughput

Decoding (normal):
  - No decoding needed (just concatenate)
  - Negligible CPU

Decoding (degraded):
  - Reed-Solomon decode: ~15ms
  - ~270 MB/s throughput

Modern servers (16+ cores):
  - 1 core can encode at 400 MB/s
  - 16 cores: 6.4 GB/s encoding
  - Typically network-bound, not CPU-bound
```

## Repair and Reconstruction

### When Chunks Are Lost

**Scenario**: Server 3 fails (holding D3).

```
Detection:
  1. Server 3 stops responding
  2. Heartbeat timeout (30 seconds)
  3. Mark Server 3 as failed

Repair:
  1. Identify objects with chunks on Server 3
  2. For each object:
     a. Read 8 surviving chunks (D1,D2,D4-D8,P1)
     b. Decode to reconstruct D3
     c. Choose new server (Server 13)
     d. Store reconstructed D3 on Server 13
     e. Update metadata

  3. Object is fully durable again
```

### Repair Bandwidth

**For 1 PB of data**:

```
8+4 scheme with 12 chunks:
  Each chunk: 1/8 of object size
  If 1 PB spread across 1000 servers:
    Each server: ~1 TB

  Server fails:
    Need to reconstruct: 1 TB
    Read from 8 servers: 8 × 1 TB = 8 TB read
    Write to 1 server: 1 TB write

    With 10 Gbps network: 1.25 GB/s
    Time to repair 1 TB: 1000/1.25 = 800 seconds
    = ~13 minutes per failed server

Much faster than replication!
  (Replication: copy entire 3 TB, take ~40 min)
```

### Lazy Repair

**Optimization**: Don't repair immediately.

```
Philosophy:
  - Object is still durable with 11 chunks (need only 8)
  - Wait for more failures before repairing
  - Reduces unnecessary repair work

Thresholds:
  - 12 chunks: Healthy
  - 11 chunks: Monitor
  - 10 chunks: Start repair
  - 9 chunks: Urgent repair
  - 8 chunks: Critical (no redundancy!)

Benefits:
  ✓ Reduces network traffic
  ✓ Handles transient failures
  ✓ Batch repairs for efficiency

Risks:
  ✗ Window where durability is reduced
  ✗ Need good monitoring
```

## Hybrid Approaches

### Local Replication + Distributed Erasure Coding

**S3's actual approach** (simplified):

```
Within AZ:
  - Use 3x replication locally
  - Fast writes (low latency)
  - Good for reads (3 local copies)

Across AZs:
  - Use erasure coding
  - Efficient storage
  - Survive AZ failure

Example:
  AZ-1: 3 replicas of chunks [D1, D2]
  AZ-2: 3 replicas of chunks [D3, D4]
  AZ-3: 3 replicas of chunks [P1, P2]

Total: 18 copies of 6 chunks = 3x overhead
  But: Fast local reads, AZ durability
```

**Tradeoff**: More overhead than pure erasure coding, but better performance.

### Tiered Erasure Coding

**Different schemes for different data**:

```
Hot data (frequently accessed):
  - 6+3 scheme (50% overhead)
  - Optimized for read performance
  - More chunks = more parallel reads

Cold data (archival):
  - 12+4 scheme (33% overhead)
  - Optimized for cost
  - Slower reads acceptable

Critical data:
  - 8+6 scheme (75% overhead)
  - Can survive 6 failures
  - Maximum durability
```

## Implementation Challenges

### Challenge 1: Partial Writes

**Problem**: What if encoding fails midway?

```
Scenario:
  1. Split object into 8 chunks
  2. Encode → generate 4 parity chunks
  3. Write D1-D4 to servers (success)
  4. Write D5 fails (network error)
  5. What now?

Solution: Two-phase commit
  Phase 1: Write all chunks to temp locations
  Phase 2: Atomic commit (update metadata)

  If Phase 1 fails: Rollback (delete temp chunks)
  If Phase 2 fails: Retry or rollback
```

### Challenge 2: Chunk Version Skew

**Problem**: Chunks updated at different times.

```
Scenario:
  1. Write object v1: Chunks D1v1-D8v1, P1v1-P4v1
  2. Overwrite with v2: Update in progress...
  3. D1v2, D2v2, D3v2 written
  4. Crash before finishing
  5. Now have: D1v2, D2v2, D3v2, D4v1, D5v1, ...

Mixed versions! Cannot decode correctly.

Solution: Version numbers + atomic commit
  - Each chunk has version number
  - Metadata tracks current version
  - Only use chunks matching current version
  - Old chunks garbage collected
```

### Challenge 3: Chunk Corruption

**Problem**: What if chunk is silently corrupted?

```
Scenario:
  1. Read 8 chunks to decode
  2. Chunk D3 has bit flip (corrupted)
  3. Decode produces garbage!

Solution: Checksums
  - Each chunk has MD5/SHA checksum
  - Verify before using
  - If corrupted: Treat as unavailable
  - Decode from other chunks
```

### Challenge 4: Alignment

**Problem**: Objects aren't multiples of chunk size.

```
Example:
  Object size: 3.7 MB
  Chunk size: 512 KB (for 8 chunks)
  Total needed: 4 MB

Padding required: 4 MB - 3.7 MB = 0.3 MB

Solutions:
  1. Zero-padding (simple, but wastes space)
  2. Store actual size in metadata (trim on read)
  3. Variable chunk sizes (complex)

S3 approach: Option 2
```

## Key Takeaways

- **Erasure coding saves 50% storage cost** - vs simple replication
- **Reed-Solomon codes provide the math** - Polynomial evaluation
- **8+4 scheme is common** - Good balance of overhead and durability
- **Chunk placement critical** - Distribute across failure domains
- **11 nines achievable** - With proper scheme and placement
- **Performance impact minimal** - CPU encoding fast, network-bound
- **Repair is efficient** - Read 8 chunks, write 1, not copy 3
- **Hybrid approaches best** - Local replication + distributed EC

## What's Next?

We can store trillions of objects durably and efficiently. But how do we manage metadata for all of them? In the next chapter, we'll explore metadata management at scale.

---

**[← Previous: The Storage Backend](./08-storage-backend.md) | [Back to Contents](./README.md) | [Next: Metadata at Scale →](./10-metadata-at-scale.md)**
