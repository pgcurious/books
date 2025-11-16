# Chapter 2: Why Traditional Storage Fails

> **"Understanding why existing solutions fail teaches us what the new solution must do."**

## The Three Traditional Approaches

Let's examine why each traditional storage approach breaks down at our scale.

## Approach 1: File Systems (NFS, CIFS)

### The Design

Traditional file systems were designed for local storage:

```
┌──────────────────────────────┐
│    Application               │
├──────────────────────────────┤
│    File System (ext4, NTFS)  │
├──────────────────────────────┤
│    Block Device Driver       │
├──────────────────────────────┤
│    Physical Disk             │
└──────────────────────────────┘
```

**Networked file systems** (NFS, CIFS) extended this:

```
┌─────────────┐         ┌──────────────┐
│   Client    │ Network │  NFS Server  │
│ Application │◄────────┤  File System │
│             │         │  Local Disk  │
└─────────────┘         └──────────────┘
```

### The Problems at Scale

#### Problem 1: Metadata Bottleneck

**File systems maintain complex metadata**:
```
For each file:
- inode number
- permissions (owner, group, mode)
- timestamps (created, modified, accessed)
- size
- block pointers
- extended attributes
```

**Listing a directory with 1 million files**:
```
ls /large-directory

Operations required:
1. Read directory inode
2. Read directory blocks (contains file names)
3. Read 1M file inodes
4. Sort and format output

Time: Minutes to hours (!)
```

**Our requirement**: Billions of objects per bucket.

**File system reality**: Directories with millions of entries become unusable.

#### Problem 2: Hierarchical Namespace

File systems use trees:
```
/
├── users/
│   ├── alice/
│   │   ├── photos/
│   │   │   ├── 2020/
│   │   │   │   ├── jan/
│   │   │   │   │   └── img1.jpg
```

**Problems**:
- Deep hierarchies are slow to traverse
- Moving files requires updating parent directory
- Renaming directories is expensive
- No natural way to partition across machines

**At our scale**:
- Users create arbitrary directory structures
- Some have millions of files in one directory
- Others have deep nesting (100+ levels)
- No way to predict or optimize

#### Problem 3: POSIX Semantics

POSIX requires:
```
Consistency guarantees:
- Read after write: Must see the latest write
- Open file consistency: All processes see same data
- Atomic operations: Rename, link, etc.

Permission model:
- User/Group/Other rwx bits
- Access control lists (ACLs)
- Special bits (setuid, sticky, etc.)
```

**In a distributed system**:
- Strong consistency is expensive (high latency)
- Permission checks slow every operation
- POSIX semantics limit scalability

**Example: Distributed rename**:
```
mv /data/old-name /data/new-name

Required operations:
1. Lock source directory
2. Lock destination directory
3. Update source directory (remove entry)
4. Update destination directory (add entry)
5. Update inode link count
6. Unlock all

If source and dest are on different servers?
→ Distributed transaction
→ High latency
→ Potential deadlock
```

#### Problem 4: Single Server Limits

**One NFS server can handle**:
- ~100 TB of storage (before performance degrades)
- ~10,000 clients (connection limit)
- ~50,000 IOPS (disk limited)

**Our requirement**: Unlimited scale.

**Options**:
1. **Scale up**: Buy bigger servers (expensive, still has limits)
2. **Federation**: Multiple NFS servers, manual partitioning (complex, breaks namespace)
3. **Clustered FS**: (GFS, Lustre) - Better, but still complex and limited

### The Verdict

```
File Systems:
✗ Metadata doesn't scale to billions of objects
✗ Hierarchical namespace limits distribution
✗ POSIX semantics too expensive at scale
✗ Single server architecture has hard limits
```

## Approach 2: Block Storage (SAN)

### The Design

**Block storage** provides raw disk blocks over network:

```
┌─────────────┐         ┌──────────────┐
│  Server     │  iSCSI  │  SAN Array   │
│  (Initiator)│◄────────┤  (Target)    │
│             │ FibreCh │  Disk Pool   │
└─────────────┘         └──────────────┘
```

Server sees it as a raw disk:
```
/dev/sdb → SAN LUN
```

Then puts a file system on top (ext4, XFS, etc.)

### The Problems at Scale

#### Problem 1: Fixed Partitioning

**SAN LUNs are pre-allocated**:
```
LUN 1: 10 TB → Server A
LUN 2: 20 TB → Server B
LUN 3: 50 TB → Server C
```

**Issues**:
- Can't easily rebalance
- Wasted capacity (LUN 1 is full, LUN 3 is empty)
- Requires manual management
- No automatic growth

**Our requirement**: Unlimited, automatic scale.

#### Problem 2: Cost

**Enterprise SAN**:
```
Hardware:
- Dual controllers: $50,000
- Disk enclosures: $10,000 each
- Fiber Channel switches: $20,000 each
- High-end disks: $500 each (with vendor markup)

Total for 1 PB: $2-5 million
Cost per GB: $2-5
```

**Our requirement**: $0.01/GB

**SAN is 200-500x too expensive.**

#### Problem 3: Limited Redundancy

**SAN durability model**:
```
RAID 6: Can survive 2 disk failures
RAID 10: Can survive 1 failure per mirror pair
```

**Calculation**:
```
1000 disks in RAID 6:
- 4% annual failure rate = 40 failures/year
- With hot spares and fast rebuild: 99.99% durability

But not 99.999999999% (11 nines)
```

**To get 11 nines**:
- Need multi-site replication
- Need multiple RAID arrays
- Cost multiplies further

#### Problem 4: Centralized Architecture

**SAN has centralized controllers**:
```
All I/O flows through controller:
┌────────┐
│Server 1├─┐
└────────┘ │
┌────────┐ │    ┌────────────┐    ┌──────┐
│Server 2├─┼───►│ Controller ├───►│ Disk │
└────────┘ │    └────────────┘    └──────┘
┌────────┐ │
│Server 3├─┘
└────────┘
```

**Controller becomes bottleneck**:
- Limited throughput (~10-100 GB/s)
- Limited IOPS (~500k)
- Single point of failure (even with HA)

**Our requirement**: Millions of requests per second.

### The Verdict

```
Block Storage (SAN):
✗ Too expensive (200-500x over budget)
✗ Fixed partitioning doesn't auto-scale
✗ Centralized architecture limits throughput
✗ Can't achieve 11 nines durability cost-effectively
```

## Approach 3: Databases

### The Design

**Could we use a database?**

```
Table: objects
- key VARCHAR PRIMARY KEY
- value BLOB
- created TIMESTAMP
- metadata JSON
```

Simple API:
```sql
INSERT INTO objects VALUES ('key1', <data>, NOW(), '{}');
SELECT value FROM objects WHERE key = 'key1';
DELETE FROM objects WHERE key = 'key1';
```

### The Problems at Scale

#### Problem 1: Storage Overhead

**Database storage**:
```
Row for 1 KB object:
- Key: 100 bytes
- Value: 1 KB
- Row overhead: 50 bytes (headers, alignment)
- Index entry: 150 bytes (B-tree)
- Transaction log: 200 bytes

Total: ~1.5 KB on disk
Overhead: 50%!
```

**For 100 PB of user data**:
- Actual storage: 150 PB
- Extra cost: $500,000/month

**Our requirement**: Minimal overhead.

#### Problem 2: Small Object Performance

**Database optimized for**:
- Transactions (ACID)
- Joins
- Queries
- Indexes

**Not optimized for**:
- Storing millions of 1 KB objects
- Streaming large blobs
- Sequential throughput

**Benchmark** (storing 1 KB objects):
```
PostgreSQL: ~5,000 writes/second
MySQL: ~8,000 writes/second
Cassandra: ~50,000 writes/second (but not ACID)

Our requirement: 1,000,000+ requests/second
```

#### Problem 3: Blob Storage

**Large objects (images, videos)**:
```
Option 1: Store in database
- Bloats database size
- Slows down backups
- Inefficient for streaming

Option 2: Store on filesystem, reference in DB
- Two systems to maintain
- Consistency issues (DB says file exists, but it's deleted)
- Complex backup/restore
```

**Neither option is ideal.**

#### Problem 4: Vertical Scaling Limits

**Traditional RDBMS**:
```
Single-node scaling:
1. More RAM: $$$
2. Faster disks: $$$
3. More CPU: $$$

Eventually hits limits:
- Largest instance: ~4 TB RAM, 128 cores
- Cost: $50,000/month (AWS r6g.metal)
- Still limited capacity
```

**Distributed databases** (Spanner, CockroachDB):
```
Better scaling, but:
- Complex setup
- Higher latency (consensus protocols)
- Expensive (distributed transactions)
- Still overhead from ACID guarantees we don't need
```

#### Problem 5: Wrong Consistency Model

**Databases provide**:
- ACID transactions
- Serializability
- Strong consistency

**We need**:
- Simple PUT/GET
- No multi-object transactions
- Eventual consistency is acceptable

**We're paying for features we don't need.**

### The Verdict

```
Databases:
✗ Too much storage overhead (~50%)
✗ Wrong performance profile (optimized for transactions, not blobs)
✗ Vertical scaling limits
✗ Paying for ACID guarantees we don't need
✗ Can't hit millions of requests/second cost-effectively
```

## The Pattern Emerges

Looking at all failures, we see a pattern:

### Traditional Storage Was Designed For:

```
Scale:         Single machine to small cluster
Cost:          Enterprise budgets
Access:        Local or LAN
Semantics:     Rich (POSIX, SQL)
Consistency:   Strong (ACID, synchronous)
Workload:      Mixed (read/write, transactional)
```

### We Need Storage Designed For:

```
Scale:         Unlimited (distributed across data centers)
Cost:          Commodity (pennies per GB)
Access:        Internet (WAN, high latency)
Semantics:     Simple (PUT, GET, DELETE)
Consistency:   Eventual (asynchronous)
Workload:      Object-oriented (immutable writes, streaming reads)
```

## The Core Insights

After studying the failures, you realize:

### Insight 1: Flat Namespace

**Hierarchical bad**:
- Hard to partition
- Expensive operations (rename, move)
- Metadata hotspots

**Flat good**:
- Easy to partition (hash key)
- Simple operations
- Scalable metadata

### Insight 2: Immutability

**Mutable files bad**:
- Requires locking
- Complex consistency
- Difficult caching

**Immutable objects good**:
- No locks needed
- Simple caching
- Easy replication

### Insight 3: Simple Operations

**Rich API bad**:
- POSIX: 50+ system calls
- SQL: Complex query planning
- Slow at scale

**Simple API good**:
- PUT/GET/DELETE
- No transactions
- Predictable performance

### Insight 4: Embrace Eventual Consistency

**Strong consistency bad**:
- Requires coordination
- High latency
- Limits scale

**Eventual consistency good**:
- No coordination needed
- Low latency
- Unlimited scale

### Insight 5: Separate Metadata and Data

**Coupled bad**:
- Metadata operations slow data operations
- Can't optimize separately

**Separated good**:
- Optimize metadata for listings
- Optimize data for throughput
- Scale independently

## The New Direction

You sketch out what the new system needs:

```
┌─────────────────────────────────────┐
│         STORAGE SERVICE             │
│                                     │
│  Properties:                        │
│  • Flat namespace (buckets + keys) │
│  • Immutable objects                │
│  • Simple API (HTTP REST)           │
│  • Eventual consistency             │
│  • Separated metadata/data          │
│  • Horizontal scaling               │
│  • Commodity hardware               │
│  • Automated redundancy             │
└─────────────────────────────────────┘
```

This is starting to look like something new.

Something that doesn't exist yet.

---

## Key Takeaways

- **File systems fail** - Hierarchical namespaces and POSIX semantics don't scale
- **Block storage fails** - Too expensive and centralized
- **Databases fail** - Wrong abstraction and performance profile
- **Traditional assumptions break** - Strong consistency and rich APIs limit scale
- **New primitives needed** - Flat namespace, immutability, eventual consistency

## What's Next?

In the next chapter, we'll explore the cost constraint in detail and understand why it forces us to commodity hardware and custom software.

---

**[← Previous: The Impossible Requirements](./01-impossible-requirements.md) | [Back to Contents](./README.md) | [Next: The Cost Constraint →](./03-cost-constraint.md)**
