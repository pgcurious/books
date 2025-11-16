# Chapter 8: The Storage Backend

> **"The magic of cloud storage ultimately comes down to spinning disks and clever software."**

## From HTTP to Disk

You've designed the namespace, placement, and durability model. Now the critical question:

**How do you actually store bytes on disk?**

### The Full Stack

```
┌─────────────────────────────┐
│  HTTP API (GET/PUT/DELETE)  │  ← User-facing
├─────────────────────────────┤
│  Object Manager             │  ← Routing, auth
├─────────────────────────────┤
│  Metadata Index             │  ← Where is object?
├─────────────────────────────┤
│  Data Placement             │  ← Which servers?
├─────────────────────────────┤
│  Storage Backend            │  ← How on disk? (This chapter)
├─────────────────────────────┤
│  File System / Raw Disk     │  ← OS layer
├─────────────────────────────┤
│  Physical Disks (HDD/SSD)   │  ← Hardware
└─────────────────────────────┘
```

This chapter focuses on the **Storage Backend**.

## The Storage Backend Challenge

### Requirements

**Must**:
- Store objects efficiently (minimize overhead)
- Support tiny objects (bytes) to huge objects (terabytes)
- Maximize disk utilization (>90% of raw capacity)
- Handle millions of objects per disk
- Enable fast lookup (milliseconds)
- Support checksumming (integrity)
- Allow efficient deletion (reclaim space)
- Work with commodity disks (HDD and SSD)

**Traditional approaches won't work**:

```
File system:
  ✗ Metadata overhead (~4 KB per file)
  ✗ Inode limits
  ✗ Directory traversal slow

Database:
  ✗ Storage overhead (indexes, logs)
  ✗ Not optimized for large blobs

Raw disk:
  ✓ Maximum control
  ✗ Must implement everything
```

## Storage Backend Design Options

### Option 1: One File Per Object

**Simplest approach**:

```
Bucket: "photos"
Key: "beach.jpg"

Store as: /mnt/disk1/photos/beach.jpg

File system hierarchy:
/mnt/disk1/
  bucket1/
    object1.jpg
    object2.pdf
  bucket2/
    object3.txt
```

**Pros**:
```
✓ Simple (use existing file system)
✓ Easy debugging (can browse with ls)
✓ Standard tools work (cp, rm, etc.)
```

**Cons**:
```
✗ Metadata overhead:
  - Each file needs inode (256 bytes)
  - Directory entry (8 bytes)
  - Overhead for 1 KB file: ~26%

✗ Inode limits:
  - ext4: ~4 billion inodes (by default)
  - One billion files ≈ 256 GB just for inodes!

✗ Directory scaling:
  - Millions of files in directory → slow

✗ Small files waste space:
  - Minimum allocation: 4 KB block
  - 100-byte file wastes 3,900 bytes
  - 97.5% waste!
```

**Verdict**: Not scalable for billions of small objects.

### Option 2: Multiple Objects Per File

**Batch objects into larger files**:

```
Shard file: /mnt/disk1/shard-0001.dat

Contains:
┌────────────────────────────┐
│ Object 1: beach.jpg        │
│ Size: 2048 bytes          │
│ Data: <image bytes>       │
├────────────────────────────┤
│ Object 2: sunset.jpg       │
│ Size: 3072 bytes          │
│ Data: <image bytes>       │
├────────────────────────────┤
│ Object 3: photo.jpg        │
│ Size: 1024 bytes          │
│ Data: <image bytes>       │
└────────────────────────────┘
```

**Pros**:
```
✓ Fewer inodes (one per shard, not per object)
✓ Better disk utilization
✓ Less file system overhead
```

**Cons**:
```
✗ Deletion is complex:
  - Can't just delete file
  - Must track "holes" in shard
  - Fragmentation over time

✗ Updates require rewrite:
  - Can't expand object in place
  - Must copy to new location

✗ Lookup requires index:
  - Which shard?
  - What offset within shard?
```

**Verdict**: Better, but still has issues.

### Option 3: Log-Structured Storage

**Append-only design** (S3's approach):

```
All writes are appends to a log:

Write Object 1:
┌────────────────────────────┐
│ Header: {key, size, md5}   │
│ Data: <bytes>              │
└────────────────────────────┘

Write Object 2:
┌────────────────────────────┐
│ Header: {key, size, md5}   │
│ Data: <bytes>              │
└────────────────────────────┘

Disk layout:
[Obj1 Header][Obj1 Data][Obj2 Header][Obj2 Data][Obj3 Header][Obj3 Data]...
```

**Pros**:
```
✓ Writes are sequential (fast on HDD)
✓ No fragmentation
✓ No file system overhead
✓ Simple and predictable
✓ Scales to billions of objects
```

**Cons**:
```
✗ Deletion creates "tombstones"
✗ Requires compaction (garbage collection)
✗ Reads may be random (if objects scattered)
```

**Verdict**: This is the winner for S3-like storage.

## Log-Structured Storage Deep Dive

### The Write Path

**Storing an object**:

```
PUT /bucket/beach.jpg

1. Receive object data
2. Compute MD5 checksum
3. Create entry:
   ┌─────────────────────────┐
   │ Magic: 0xABCD           │  ← Identify entry start
   │ Version: 1              │
   │ Bucket: "photos"        │
   │ Key: "beach.jpg"        │
   │ Size: 2048              │
   │ MD5: a3f2...            │
   │ Timestamp: 1640000000   │
   │ Data: <2048 bytes>      │
   │ Footer: 0xDCBA          │  ← Identify entry end
   └─────────────────────────┘

4. Append to current log file
5. Record in index:
   "photos/beach.jpg" → (disk1, offset=12800, size=2048)

6. Return success
```

**Log file structure**:

```
Log file: /mnt/disk1/data/log-0001.dat

Offset 0:      [Object 1 Entry]
Offset 2048:   [Object 2 Entry]
Offset 5120:   [Object 3 Entry]
Offset 8192:   [Object 4 Entry]
...

Each entry is self-contained:
- Header (metadata)
- Data (bytes)
- Footer (checksum)
```

### The Read Path

**Retrieving an object**:

```
GET /bucket/beach.jpg

1. Query index:
   "photos/beach.jpg" → (disk1, offset=12800, size=2048)

2. Seek to offset:
   lseek(fd, 12800, SEEK_SET)

3. Read entry:
   read(fd, buffer, entry_size)

4. Validate:
   - Check magic numbers
   - Verify MD5 checksum
   - Ensure size matches

5. Extract data from entry

6. Return to client
```

**Read performance**:
```
HDD:
  Seek time: ~5ms
  Read time: 2048 bytes @ 100 MB/s = 0.02ms
  Total: ~5ms per object

SSD:
  Seek time: ~0.1ms
  Read time: 2048 bytes @ 500 MB/s = 0.004ms
  Total: ~0.1ms per object

Sequential reads (batch):
  Much faster (no seeks)
```

### The Delete Path

**Deleting an object**:

```
DELETE /bucket/beach.jpg

Cannot delete from middle of log file!
(Would create holes, complicate reads)

Solution: Tombstone

1. Write tombstone entry:
   ┌─────────────────────────┐
   │ Magic: 0xABCD           │
   │ Type: TOMBSTONE         │  ← Special type
   │ Bucket: "photos"        │
   │ Key: "beach.jpg"        │
   │ Timestamp: 1640001000   │
   └─────────────────────────┘

2. Update index:
   "photos/beach.jpg" → DELETED

3. Return success

4. Actual space reclaimed later (compaction)
```

### Log Rotation

**When to start a new log file?**

```
Triggers:
1. Current log file reaches size limit:
   - Typically: 1-10 GB per log file

2. Time-based:
   - New log every hour/day

3. Compaction:
   - After compaction, new log file

Benefits of rotation:
✓ Smaller files easier to manage
✓ Compaction works on closed files
✓ Can distribute files across disks
✓ Easier to backup/archive old files
```

**Naming convention**:
```
/mnt/disk1/data/
  log-20240115-0001.dat
  log-20240115-0002.dat
  log-20240115-0003.dat
  log-20240115-0004.dat (current, being written)
```

## The Index

### What to Index

**The index maps keys to locations**:

```
Index entry:
  Key: "photos/beach.jpg"
  Value:
    - Disk ID: disk1
    - Log file: log-20240115-0003.dat
    - Offset: 12800
    - Size: 2048
    - MD5: a3f2b8d4...
    - Timestamp: 1640000000
```

### Index Storage Options

**Option 1: In-memory hash table**:

```
map<string, Location> index;

Pros:
  ✓ Fast lookup: O(1)
  ✓ Simple

Cons:
  ✗ RAM intensive
  ✗ Lost on restart (must rebuild)

For 1 million objects:
  Per entry: 100 bytes
  Total: 100 MB

For 1 billion objects:
  Total: 100 GB (!)
  → Too much RAM
```

**Option 2: On-disk B-tree** (like SQLite):

```
B-tree index file: index.db

Pros:
  ✓ Scales to billions
  ✓ Persistent (survives restart)

Cons:
  ✗ Slower (disk I/O)
  ✗ Fragmentation over time

Lookup: O(log N) with disk seeks
  = ~10 disk seeks for 1 billion entries
  = ~50ms on HDD
```

**Option 3: LSM-tree** (like RocksDB):

```
Log-Structured Merge tree:
  - Write buffer (in memory)
  - L0: Small SSTables (sorted)
  - L1: Larger SSTables
  - L2: Even larger
  - ...

Pros:
  ✓ Fast writes (append to log)
  ✓ Scales to billions
  ✓ Good read performance
  ✓ Compaction merges levels

Cons:
  ✗ More complex
  ✗ Write amplification

This is S3's choice.
```

### Index Operations

**Insert**:
```
PUT /photos/beach.jpg

index.put("photos/beach.jpg", {
  disk: "disk1",
  file: "log-0001.dat",
  offset: 12800,
  size: 2048,
  md5: "a3f2...",
  timestamp: 1640000000
})
```

**Lookup**:
```
GET /photos/beach.jpg

location = index.get("photos/beach.jpg")
→ Returns location info
```

**Delete**:
```
DELETE /photos/beach.jpg

index.delete("photos/beach.jpg")
→ Marks as deleted
→ Tombstone remains until compaction
```

**List**:
```
LIST /photos/ with prefix "2024/"

results = index.scan("photos/2024/", "photos/2024/~")
→ Returns all keys in range
```

## Compaction and Garbage Collection

### Why Compaction?

**Problem**: Deleted and overwritten objects leave garbage:

```
Log file before compaction:
┌────────────────────────────┐
│ beach.jpg (v1)    2 KB     │
├────────────────────────────┤
│ sunset.jpg (v1)   3 KB     │
├────────────────────────────┤
│ beach.jpg (v2)    2 KB     │  ← Overwrites v1
├────────────────────────────┤
│ TOMBSTONE(sunset.jpg)      │  ← Deleted
├────────────────────────────┤
│ photo.jpg         1 KB     │
└────────────────────────────┘

Total size: 8 KB
Live data: 3 KB (beach v2 + photo)
Garbage: 5 KB (62.5% waste!)
```

**Compaction reclaims space**.

### Compaction Process

**Steps**:

```
1. Choose log files to compact:
   - Old files (not being written)
   - High garbage ratio (>50%)

2. Scan log file:
   For each entry:
     a. Check index: Is this latest version?
     b. If yes: Copy to new log file
     c. If no (deleted/overwritten): Skip

3. Write new log file:
   ┌────────────────────────────┐
   │ beach.jpg (v2)    2 KB     │
   ├────────────────────────────┤
   │ photo.jpg         1 KB     │
   └────────────────────────────┘

   Total size: 3 KB
   Reclaimed: 5 KB

4. Update index:
   - Point to new locations in new log file

5. Delete old log file:
   - Free disk space
```

### Compaction Scheduling

**When to compact?**

```
Triggers:
1. Garbage ratio > threshold:
   - If >50% garbage, compact soon

2. Disk space low:
   - If disk >80% full, compact aggressively

3. Scheduled (background):
   - Compact during low traffic hours

4. Manual (operations):
   - For maintenance
```

**Compaction rate**:
```
Goal: Balance:
  - I/O cost (reading + writing)
  - Space savings
  - Impact on live traffic

Typical: Compact 1-10% of data per day
  = 1 PB storage → compact 10-100 TB/day
```

## Disk Management

### Multi-Disk Systems

**Typical server**:
```
12 disks × 4 TB = 48 TB raw

How to use all disks?
```

**Option 1: RAID** (don't do this):
```
RAID 6: Combine all disks, lose 2 for parity
  Capacity: 10 × 4 TB = 40 TB
  Durability: Can survive 2 disk failures

Problems:
  ✗ Expensive (RAID controller or CPU overhead)
  ✗ Rebuild storms (stress on remaining disks)
  ✗ Locks in capacity (can't mix disk sizes)
  ✗ Single failure domain

We already have durability at cluster level!
RAID is redundant.
```

**Option 2: JBOD** (Just a Bunch Of Disks):
```
Each disk is independent:
  /mnt/disk1  → 4 TB
  /mnt/disk2  → 4 TB
  ...
  /mnt/disk12 → 4 TB

Each disk:
  - Has own log files
  - Has own index
  - Managed independently

Benefits:
  ✓ Simple
  ✓ No RAID overhead
  ✓ Disk failure = lose only that disk's data
  ✓ Easy to replace/upgrade disks
  ✓ Software RAID at cluster level (better)
```

**S3's choice**: JBOD

### Disk Failure Handling

**When a disk fails**:

```
1. Detection:
   - I/O errors
   - S.M.A.R.T. warnings
   - Latency spikes

2. Mark disk as failed:
   - Stop writing to it
   - Stop reading from it

3. Notify cluster:
   - List objects that were on failed disk
   - Trigger rebuild (from replicas elsewhere)

4. Replace disk:
   - Hot-swap (if supported)
   - Or schedule maintenance

5. Rebuild data:
   - Copy from other replicas
   - Repopulate new disk
```

### Disk Monitoring

**Key metrics**:

```
1. Utilization:
   - % of capacity used
   - Target: 70-80%

2. I/O latency:
   - Average read/write time
   - Alert if >100ms (HDD) or >10ms (SSD)

3. Error rate:
   - I/O errors per billion reads
   - Target: < 1

4. S.M.A.R.T. metrics:
   - Reallocated sectors
   - Pending sectors
   - Temperature
   - Power-on hours

5. Throughput:
   - MB/s read/write
   - Monitor saturation
```

## Storage Efficiency

### Overhead Analysis

**For 1 TB of user data**:

```
User data: 1 TB

Log structure overhead:
  - Entry headers: ~0.5% (128 bytes per object)
  - Padding/alignment: ~0.5%
  - Tombstones (pre-compaction): ~1%
  Total: ~2%

File system overhead (XFS/ext4):
  - Superblocks, inodes: ~1%
  - Fragmentation: ~1%
  Total: ~2%

Total overhead: ~4%

Actual disk usage: 1.04 TB
Efficiency: 96%
```

**Compare to "one file per object"**:
```
User data: 1 TB

File system overhead:
  - Inodes (256 bytes each): ~10% for small files
  - Block allocation: ~5% (4KB blocks for small files)
  - Directory entries: ~2%
  Total: ~17%

Actual disk usage: 1.17 TB
Efficiency: 85%
```

**Log-structured is 11% more efficient!**

### Small Object Handling

**Challenge**: Tiny objects (100 bytes) have high overhead.

**Solutions**:

```
1. Batching:
   - Buffer small writes
   - Flush in batches
   - Reduces overhead per object

2. Compression:
   - Compress small objects together
   - Save space, but cost CPU
   - S3 doesn't do this (users can compress)

3. Inline storage:
   - Store tiny objects in metadata index
   - Avoid disk write for <1KB objects
   - Faster access
```

## Key Takeaways

- **Log-structured storage wins** - Sequential writes, simple, scales
- **Append-only design** - Immutability at the storage layer
- **Index is critical** - Fast lookup from key to disk location
- **Compaction required** - Reclaim space from deletes/overwrites
- **JBOD over RAID** - Simpler, cheaper, cluster-level durability
- **Monitoring essential** - Detect failures early, maintain health
- **Efficiency matters** - 96% vs 85% is huge at exabyte scale

## What's Next?

We have objects stored on disk. But how do we ensure durability across multiple disks and servers? In the next chapter, we'll dive deep into replication and erasure coding.

---

**[← Previous: Data Placement and Partitioning](./07-data-placement.md) | [Back to Contents](./README.md) | [Next: Replication and Erasure Coding →](./09-replication-erasure-coding.md)**
