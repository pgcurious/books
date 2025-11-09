# Inventing the LSM Tree
## A First-Principles Journey to an Elegant Data Structure

---

## Introduction: The Joy of Discovery

Imagine you're sitting at a blank whiteboard. Your task: design a database that can handle millions of writes per second while still serving reads efficiently. You don't know what an "LSM tree" is—the term hasn't been invented yet. You only have your knowledge of basic data structures, an understanding of how computers work, and the ability to reason about trade-offs.

This book is that journey. We'll start with the simplest possible solution and let each problem guide us to the next insight. Along the way, we'll discover one of computing's most elegant data structures—not because someone told us about it, but because we invented it ourselves.

**The Promise**: By the end of this book, you won't just understand LSM trees. You'll understand the *thought process* that creates elegant solutions to complex problems.

Let's begin.

---

## Chapter 1: The Problem Space

### What We're Building

We need a storage system with these requirements:

**Functional Requirements**:
- Store key-value pairs (like a giant hash map)
- Support PUT(key, value) - write or update
- Support GET(key) - retrieve value
- Support DELETE(key) - remove entry

**Performance Requirements**:
- Handle **millions of writes per second**
- Serve reads efficiently (sub-millisecond)
- Store more data than fits in RAM (terabytes)
- Survive crashes (persistence)

**Constraints**:
- RAM is limited (expensive, volatile)
- Disk is large but slow
- Need to run on commodity hardware

### The Performance Reality

Before we start, let's understand our hardware:

**Memory (RAM)**:
- Random access: ~100 nanoseconds
- Sequential access: ~100 nanoseconds
- Capacity: ~100 GB (realistic for one machine)

**Disk (SSD)**:
- Random read: ~100 microseconds (1,000x slower than RAM)
- Sequential read: ~1 microsecond per KB (much faster than random)
- Random write: ~100 microseconds
- Sequential write: ~1 microsecond per KB (100x faster than random write!)
- Capacity: ~10 TB (much larger than RAM)

**The Critical Insight**: Sequential disk access is ~100x faster than random disk access.

This single fact will shape our entire design.

---

## Chapter 2: First Attempt — The In-Memory Hash Table

### The Simplest Solution

Let's start simple. A hash table gives us O(1) operations:

```
class SimpleDB {
    map<string, string> data;

    void put(key, value) {
        data[key] = value;
    }

    string get(key) {
        return data[key];
    }

    void delete(key) {
        data.erase(key);
    }
}
```

**Performance**:
- PUT: O(1), ~100 nanoseconds
- GET: O(1), ~100 nanoseconds
- DELETE: O(1), ~100 nanoseconds

**Analysis**: Perfect! We can handle millions of operations per second easily.

### The Fatal Flaw

**Question**: What happens when the machine crashes?

**Answer**: All data is lost. Everything lives in volatile RAM.

**Question**: What happens when we have 1 TB of data but only 100 GB of RAM?

**Answer**: Doesn't fit. We need disk storage.

**Conclusion**: We need persistence. Back to the drawing board.

---

## Chapter 3: The Persistence Problem

### Attempt 2: Hash Table + Disk File

**Idea**: Keep the hash table, but also write to disk for persistence.

```
class PersistentDB {
    map<string, string> data;
    File diskFile;

    void put(key, value) {
        data[key] = value;

        // Write to disk for persistence
        diskFile.seek(hash(key) % FILE_SIZE);
        diskFile.write(key, value);
    }

    string get(key) {
        if (data.contains(key)) {
            return data[key];
        }

        // Not in memory, read from disk
        diskFile.seek(hash(key) % FILE_SIZE);
        return diskFile.read();
    }
}
```

### The Performance Problem

Let's measure what happens under load:

**Write Test**: 1 million PUTs
- Each PUT requires one random disk write
- Random write: ~100 microseconds
- Total time: 1,000,000 × 100μs = 100 seconds
- **Throughput: 10,000 writes/second**

We needed **millions** per second. We got **thousands**. We're 100-1000x too slow!

### Why So Slow?

**The Problem**: Random disk writes are expensive.

When you write to a random location on disk:
1. Disk head must physically move (seek time)
2. Wait for platter rotation (rotational latency)
3. Write the data

Even on SSDs (no moving parts), random writes are slow because:
- Writing requires erasing entire blocks
- Wear leveling adds overhead
- Garbage collection creates unpredictability

### The Key Question

**Can we avoid random disk writes?**

This question will lead us to our first major insight.

---

## Chapter 4: The Sequential Insight

### A Different Approach

**Observation**: Sequential writes are ~100x faster than random writes.

**Idea**: What if we just... append everything?

Instead of writing to random locations, we write to the end of a file:

```
class AppendOnlyDB {
    map<string, string> memoryCache;
    File log;  // Append-only log file

    void put(key, value) {
        memoryCache[key] = value;

        // Append to end of log (sequential write!)
        log.append(key, value, timestamp);
    }
}
```

### The Performance Win

**Write Test**: 1 million PUTs
- Each PUT appends to log (sequential write)
- Sequential write: ~1 microsecond per entry
- Total time: 1,000,000 × 1μs = 1 second
- **Throughput: 1,000,000 writes/second**

**Success!** We achieved our goal: millions of writes per second.

### The Append-Only Log Principle

**Structure of our log file**:
```
[key1, value1, timestamp1]
[key2, value2, timestamp2]
[key3, value3, timestamp3]
[key1, value1_updated, timestamp4]  # Update: just append
[key2, DELETE, timestamp5]          # Delete: append tombstone
...
```

**Properties**:
- Writes are always fast (sequential)
- Updates don't modify old data, just append new version
- Deletes append a "tombstone" marker
- Immutable: once written, never changed

**The Trade-off**: We've optimized writes. But what about reads?

---

## Chapter 5: The Read Problem

### Reading from the Log

To find a key, we need to scan the log:

```
string get(key) {
    // Check memory first
    if (memoryCache.contains(key)) {
        return memoryCache[key];
    }

    // Scan log from end to beginning (newest first)
    for (entry in log.reverseIterator()) {
        if (entry.key == key) {
            if (entry.isTombstone()) {
                return NOT_FOUND;
            }
            return entry.value;
        }
    }
    return NOT_FOUND;
}
```

### The Performance Disaster

**Read Test**: 1 million GETs (keys not in memory cache)
- Log size: 1 billion entries
- Average scan: 500 million entries
- Each entry read: ~1 microsecond
- Average read time: 500 seconds per GET

**This is catastrophically slow!**

### The Problem Analysis

We optimized writes at the expense of reads:
- **Write performance**: O(1), microseconds ✓
- **Read performance**: O(n), seconds ✗

**Question**: How can we make reads faster without sacrificing write performance?

---

## Chapter 6: The Memory Buffer Solution

### Insight: Recent Data is Hot

**Observation**: In most workloads, recently written data is frequently accessed (temporal locality).

**Idea**: Keep recent writes in memory.

```
class MemTableDB {
    SortedMap<string, string> memTable;  // In-memory sorted structure
    File log;

    void put(key, value) {
        memTable[key] = value;
        log.append(key, value);  // Write-ahead log for durability
    }

    string get(key) {
        if (memTable.contains(key)) {
            return memTable[key];  // O(log n) where n is memTable size
        }

        // Fall back to scanning log (still slow)
        return scanLog(key);
    }
}
```

### Why Sorted?

We use a sorted structure (like a balanced tree) instead of a hash table because:
- We'll need to iterate in order later (spoiler!)
- Still O(log n) lookup (fast enough)
- Enables range queries

**Common implementation**: Skip list or Red-Black tree

### Performance Improvement

**Read Test** (assuming 80% of reads hit memTable):
- 80% of GETs: O(log n), microseconds ✓
- 20% of GETs: Still scanning log, seconds ✗

Better, but not solved. And we have a new problem...

---

## Chapter 7: The Memory Overflow Problem

### The Constraint

**Problem**: MemTable lives in RAM. RAM is limited.

**Question**: What happens when memTable fills up?

**Options**:
1. Stop accepting writes? (Unacceptable)
2. Evict old entries? (Where do they go?)
3. Write memTable to disk? (Promising!)

### Flushing to Disk

**Idea**: When memTable reaches a threshold (e.g., 64 MB), write it to disk and start fresh.

```
class FlushingDB {
    SortedMap memTable;
    File writeAheadLog;
    List<File> diskFiles;  // Flushed memTables

    void put(key, value) {
        writeAheadLog.append(key, value);  // Durability
        memTable[key] = value;

        if (memTable.size() > THRESHOLD) {
            flush();
        }
    }

    void flush() {
        File newFile = createNewFile();

        // Write entire memTable to disk in sorted order
        for ((key, value) in memTable.inOrder()) {
            newFile.append(key, value);
        }

        diskFiles.add(newFile);
        memTable.clear();
        writeAheadLog.clear();
    }
}
```

### The Structure We've Created

We now have:
1. **MemTable**: Recent data, in memory, sorted
2. **Disk files**: Flushed memTables, on disk, sorted
3. **Write-ahead log**: For crash recovery

Each disk file is **immutable** and **sorted**. This is called a **Sorted String Table (SSTable)**.

---

## Chapter 8: Sorted String Tables (SSTables)

### The SSTable Format

```
SSTable structure:
┌─────────────────────────────┐
│ Data Block 1                │  [key1, value1]
│   [apple, data_a]          │  [key2, value2]
│   [banana, data_b]         │  ...sorted...
│   ...                      │
├─────────────────────────────┤
│ Data Block 2                │
│   [carrot, data_c]         │
│   [date, data_d]           │
│   ...                      │
├─────────────────────────────┤
│ Index Block                 │  Points to data blocks
│   apple → Block 1, offset 0│
│   carrot → Block 2, offset 0│
│   ...                      │
├─────────────────────────────┤
│ Footer                      │  Metadata
└─────────────────────────────┘
```

**Properties**:
- **Sorted**: Keys in order
- **Immutable**: Never modified after creation
- **Indexed**: Fast lookup within file
- **Block-based**: Data organized in blocks for I/O efficiency

### Reading from an SSTable

```
string readFromSSTable(File sstable, key) {
    // 1. Read index (can be cached in memory)
    Index index = sstable.readIndex();

    // 2. Binary search index to find block
    Block targetBlock = index.findBlock(key);

    // 3. Read block from disk (one sequential read!)
    BlockData data = sstable.readBlock(targetBlock);

    // 4. Binary search within block
    return data.binarySearch(key);
}
```

**Performance**:
- One disk read per SSTable
- Binary search (O(log n))
- Much faster than scanning!

---

## Chapter 9: The Multiple Files Problem

### Our Current Read Path

```
string get(key) {
    // 1. Check memTable (newest data)
    if (memTable.contains(key)) {
        return memTable[key];
    }

    // 2. Check each SSTable (oldest to newest)
    for (sstable in diskFiles.reverse()) {
        value = readFromSSTable(sstable, key);
        if (value exists) {
            return value;
        }
    }

    return NOT_FOUND;
}
```

### The Problem

After running for a while:
- We've flushed memTable 1,000 times
- We have **1,000 SSTables** on disk
- Each GET requires checking up to 1,000 files!

**Read Test**:
- 1,000 SSTables to search
- Each SSTable lookup: ~100 microseconds (one disk read)
- Total read time: 1,000 × 100μs = **100 milliseconds**

We needed **sub-millisecond**. We got **100 milliseconds**. We're 100x too slow again!

### The Observation

Many SSTables contain overlapping key ranges:
- SSTable_1: [apple...mango]
- SSTable_2: [banana...orange]
- SSTable_3: [apricot...lemon]

**Question**: Can we merge these files to reduce the number we need to search?

---

## Chapter 10: The Merge Solution — Compaction

### The Merge Insight

**Idea**: Periodically merge multiple SSTables into one larger SSTable.

Since each SSTable is sorted, we can merge them efficiently (like merge sort):

```
function mergeSSTables(files) {
    // Open iterators for each file
    iterators = [file.iterator() for file in files];

    // Create new merged SSTable
    newSSTable = createNewSSTable();

    // Merge using priority queue (min-heap)
    heap = MinHeap(iterators, compareByKey);

    while (!heap.empty()) {
        (key, value, iterator) = heap.extractMin();

        // Skip duplicates, keep only latest version
        if (key != lastKey) {
            newSSTable.append(key, value);
            lastKey = key;
        }

        // Advance iterator and re-insert
        if (iterator.hasNext()) {
            heap.insert(iterator.next(), iterator);
        }
    }

    return newSSTable;
}
```

### Merge Performance

**Merging 10 SSTables** (each 64 MB):
- Input: 640 MB
- Output: ~640 MB (minus duplicates)
- Operation: Sequential read + sequential write
- Time: ~1-2 seconds

**Result**: 10 files → 1 file

### The Benefits

1. **Fewer files to search**: 1,000 files → 100 files (10x fewer reads)
2. **Reclaim space**: Remove old versions and deleted keys
3. **Maintain performance**: As we write, we periodically compact

---

## Chapter 11: The Compaction Strategy Question

### When to Merge?

**Naive approach**: Merge all files periodically

**Problem**:
- Merging 1,000 large files takes hours
- During merge, writes pile up
- System becomes unresponsive

**Better idea**: Merge incrementally

### Size-Tiered Compaction (First Attempt)

**Strategy**: Merge files of similar size

```
Files on disk:
64 MB, 64 MB, 64 MB, 64 MB  → Merge → 256 MB
256 MB, 256 MB, 256 MB, 256 MB → Merge → 1 GB
1 GB, 1 GB, 1 GB, 1 GB → Merge → 4 GB
```

**Properties**:
- Files organized into size tiers
- Merge when we have N files of similar size
- Each merge creates a file in the next tier

### The Problem with Size-Tiered

**Observation**: Key ranges still overlap across tiers!

To read a key, we might need to check:
- 4 files in 64 MB tier
- 4 files in 256 MB tier
- 4 files in 1 GB tier
- = **12 files total**

Better than 1,000, but can we do even better?

---

## Chapter 12: The Level Insight

### A Different Organization

**Question**: What if we organized files into levels, where each level has **non-overlapping** key ranges?

**The Insight**:
```
Level 0: Files can overlap (recently flushed memTables)
  [a...m] [b...n] [c...o]

Level 1: Files don't overlap (10x size of L0)
  [a...e] [f...j] [k...o] [p...z]

Level 2: Files don't overlap (10x size of L1)
  [a...c] [d...f] [g...i] [j...l] [m...o] [p...r] [s...u] [v...z]
```

**Properties**:
- Level 0: Small, may overlap (fresh from memTable)
- Each subsequent level: 10x larger, no overlap within level
- Files in each level cover the entire key space

### Leveled Compaction

**When Level 0 fills up**:
1. Pick a file from Level 0
2. Find overlapping files in Level 1
3. Merge them
4. Write output to Level 1

**When Level 1 fills up**:
1. Pick a file from Level 1
2. Find overlapping files in Level 2
3. Merge them
4. Write output to Level 2

**And so on...**

---

## Chapter 13: The LSM Tree Structure

### What We've Invented

Congratulations! We've just invented the **Log-Structured Merge Tree (LSM Tree)**!

**Complete structure**:

```
┌─────────────────────────────────────────────────┐
│ MemTable (RAM)                                  │
│ • Sorted map (red-black tree or skip list)     │
│ • Size: ~64 MB                                  │
│ • Accepts all writes                            │
└─────────────────────────────────────────────────┘
                    ↓ (flush when full)
┌─────────────────────────────────────────────────┐
│ Level 0 (Disk) — ~256 MB total                  │
│ • 4 SSTables, may overlap                       │
│ • Immutable, sorted                             │
└─────────────────────────────────────────────────┘
                    ↓ (compact when full)
┌─────────────────────────────────────────────────┐
│ Level 1 (Disk) — ~2.5 GB total                  │
│ • 40 SSTables, non-overlapping                  │
│ • Each SSTable: ~64 MB                          │
└─────────────────────────────────────────────────┘
                    ↓ (compact when full)
┌─────────────────────────────────────────────────┐
│ Level 2 (Disk) — ~25 GB total                   │
│ • 400 SSTables, non-overlapping                 │
└─────────────────────────────────────────────────┘
                    ↓ (continue...)
```

### The Complete Operations

**Write (PUT)**:
```
1. Append to write-ahead log (durability)
2. Insert into memTable
3. If memTable full:
   a. Flush to Level 0 as new SSTable
   b. Clear memTable and log
4. Background: Compact levels when thresholds exceeded
```

**Read (GET)**:
```
1. Check memTable
2. Check Level 0 SSTables (newest first)
3. Check Level 1 (binary search to find right SSTable)
4. Check Level 2 (binary search)
5. Continue down levels until found or exhausted
```

**Delete**:
```
1. Insert tombstone into memTable
2. Tombstone propagates through levels during compaction
3. Eventually, both key and tombstone removed from deepest level
```

---

## Chapter 14: Performance Analysis

### Write Performance

**Single write**:
1. Write to log: ~1 microsecond (sequential)
2. Insert to memTable: ~1 microsecond (tree operation)
3. **Total: ~2 microseconds**

**Throughput**: ~500,000 writes/second on single thread

**Write amplification**:
- Data written once to memTable
- Eventually written to each level during compaction
- Write amplification = ~10x (acceptable for the benefits)

### Read Performance

**Best case** (key in memTable):
- **Latency: ~1 microsecond**

**Average case** (key in Level 1 or 2):
- Check memTable: ~1 microsecond
- Check Level 0: ~4 × 100 microseconds = 400 microseconds
- Check Level 1: 1 × 100 microseconds = 100 microseconds
- **Total: ~500 microseconds = 0.5 milliseconds** ✓

**Worst case** (key doesn't exist):
- Must check all levels
- ~1 millisecond

### Space Efficiency

**For 1 TB of data**:
- Level 0: ~0.25 GB
- Level 1: ~2.5 GB
- Level 2: ~25 GB
- Level 3: ~250 GB
- Level 4: ~750 GB (actual data)
- **Total: ~1.03 TB** (3% overhead)

Old versions and tombstones are cleaned up during compaction.

---

## Chapter 15: Optimizations

### The Bloom Filter Insight

**Problem**: Reading from multiple SSTables is expensive.

**Question**: Can we quickly determine if a key is **definitely not** in an SSTable without reading it?

**Solution**: Bloom filters!

**Bloom Filter**:
- Probabilistic data structure
- Small (a few KB per SSTable)
- Can answer: "Definitely not present" or "Might be present"
- No false negatives, rare false positives

```
Each SSTable has a Bloom filter:

┌─────────────────────────────┐
│ SSTable                     │
│ • Data blocks               │
│ • Index                     │
│ • Bloom filter (10 KB)     │ ← Loaded in memory
└─────────────────────────────┘

Before reading SSTable:
if (!bloomFilter.mightContain(key)) {
    return NOT_FOUND;  // Skip disk read!
}
```

**Impact**: Reduces disk reads by 90%+ for keys that don't exist.

### Read Path with Bloom Filters

```
string get(key) {
    // Check memTable
    if (memTable.contains(key)) {
        return memTable[key];
    }

    // Check Level 0
    for (sstable in level0.reverse()) {
        if (sstable.bloomFilter.mightContain(key)) {
            value = sstable.get(key);
            if (value exists) return value;
        }
    }

    // Check deeper levels
    for (level in [level1, level2, ...]) {
        sstable = level.findOverlappingSSTable(key);
        if (sstable.bloomFilter.mightContain(key)) {
            value = sstable.get(key);
            if (value exists) return value;
        }
    }

    return NOT_FOUND;
}
```

**New average read latency**: ~100-200 microseconds ✓

### Other Optimizations

**Compression**:
- Compress SSTable blocks
- Save disk space (2-5x compression typical)
- Trade CPU for I/O (usually worth it)

**Block Cache**:
- Cache frequently accessed SSTable blocks in RAM
- LRU eviction policy
- Dramatically improves read performance for hot data

**Compaction Strategies**:
- **Leveled**: What we described (optimizes reads)
- **Size-tiered**: Merge similar-sized files (optimizes writes)
- **Hybrid**: Combine strategies for different levels

**Partitioning**:
- Split key space across multiple LSM trees
- Parallelize operations
- Scale horizontally

---

## Chapter 16: The Complete Picture

### The LSM Tree Design

Let's review what we invented and why:

**Problem 1**: Need fast writes
→ **Solution**: Append-only log (sequential I/O)

**Problem 2**: Slow reads from log
→ **Solution**: Keep recent data in memory (memTable)

**Problem 3**: Memory fills up
→ **Solution**: Flush to disk as sorted files (SSTables)

**Problem 4**: Too many files to search
→ **Solution**: Merge files (compaction)

**Problem 5**: Merging is expensive
→ **Solution**: Organize into levels

**Problem 6**: Still checking multiple files
→ **Solution**: Bloom filters

**Each problem naturally led to the next insight.**

### The Trade-offs

**What we optimized for**:
- ✓ Write throughput (millions/second)
- ✓ Write latency (microseconds)
- ✓ Space efficiency (minimal overhead)
- ✓ Scalability (terabytes of data)

**What we traded away**:
- ✗ Read latency (worse than B-trees for point queries)
- ✗ Write amplification (data written multiple times)
- ✗ Complexity (many moving parts)

**When to use LSM trees**:
- Write-heavy workloads
- Time-series data
- Logging systems
- Analytics databases
- Sequential scan workloads

**When NOT to use LSM trees**:
- Read-heavy point queries (use B-trees)
- Need guaranteed read latency
- Strong consistency requirements (more complex with LSM)

---

## Chapter 17: Real-World Implementations

### The LSM tree pattern powers many systems:

**LevelDB** (Google):
- Embedded key-value store
- Used in Chrome browser (IndexedDB)
- Leveled compaction strategy

**RocksDB** (Facebook):
- Based on LevelDB
- Heavily optimized for SSDs
- Used by MySQL, MongoDB, and others
- Powers Facebook's infrastructure

**Apache Cassandra**:
- Distributed database
- Size-tiered compaction
- Multi-datacenter replication

**Apache HBase**:
- Built on Hadoop
- Massive scale (petabytes)
- Used by Facebook, Twitter

**ScyllaDB**:
- Cassandra-compatible
- Written in C++ for performance
- Microsecond latencies

### Variations and Innovations

**WiscKey** (research):
- Separate keys and values
- Keys in LSM tree, values in log
- Reduces write amplification

**PebblesDB** (research):
- "Skip-list" style levels
- Reduces compaction overhead
- Novel guard concept

**Bw-Tree** (Microsoft):
- Lock-free LSM tree
- Used in SQL Server Hekaton

---

## Chapter 18: The Thought Process

### What We Learned About Problem-Solving

**Pattern 1: Start Simple**
- Begin with the simplest solution
- Understand why it fails
- Each limitation guides the next refinement

**Pattern 2: Understand Constraints**
- Sequential I/O vs. random I/O
- RAM vs. disk trade-offs
- These constraints shaped our design

**Pattern 3: Embrace Trade-offs**
- No perfect solution exists
- Optimize for your workload
- Accept the costs of your choices

**Pattern 4: Solve One Problem at a Time**
- Don't try to solve everything at once
- Each solution introduces new challenges
- That's okay—it's progress

**Pattern 5: Look for Invariants**
- Immutability simplified our design
- Sorted order enabled binary search
- Sequential writes enabled high throughput

### Applying This to Other Problems

This thought process works beyond LSM trees:

**When facing a complex problem**:

1. **Start with the naive solution**
   - What's the simplest thing that could work?

2. **Measure and understand bottlenecks**
   - Where does it fail?
   - What are the constraints?

3. **Attack one bottleneck at a time**
   - Don't try to optimize everything
   - Focus on the biggest pain point

4. **Accept trade-offs**
   - What are you optimizing for?
   - What are you willing to sacrifice?

5. **Iterate**
   - Each solution reveals new problems
   - That's the creative process

**The LSM tree wasn't invented in one flash of insight. It emerged from solving one problem at a time.**

---

## Chapter 19: Deep Dives

### Write Amplification Analysis

**Write Amplification Factor (WAF)**: How many times data is written to disk relative to user writes.

**In LSM trees**:

```
User writes 1 GB of data:
1. Write to memTable/log: 1 GB
2. Flush to Level 0: 1 GB
3. Compact to Level 1: 1 GB
4. Compact to Level 2: 1 GB
5. Compact to Level 3: 1 GB

Total written: 5 GB
WAF: 5x
```

**Factors affecting WAF**:
- Number of levels: More levels = higher WAF
- Level size ratio: Larger ratio = lower WAF (but more L0 files)
- Compaction strategy: Size-tiered has different WAF than leveled

**Why it's acceptable**:
- Sequential writes are cheap
- Write throughput remains high
- Modern SSDs can handle it (with wear leveling)

**Mitigation strategies**:
- Increase level size ratio (e.g., 10x → 20x)
- Use compression (write less data)
- Separate hot/cold data

### Read Amplification Analysis

**Read Amplification**: How many disk reads needed for one query.

**In LSM trees**:
```
Point query for key that doesn't exist:
1. Check Level 0: 4 reads (4 SSTables)
2. Check Level 1: 1 read
3. Check Level 2: 1 read
4. Check Level 3: 1 read

Total: 7 reads (worst case without Bloom filters)
With Bloom filters: ~1 read (false positive rate)
```

**Mitigation strategies**:
- Bloom filters (primary technique)
- Block cache (cache hot SSTables)
- Fence pointers (skip SSTables outside key range)
- Partition filtering

### Space Amplification

**Space Amplification**: Extra space used beyond actual data.

**Sources of extra space**:
1. **Old versions**: During compaction, old and new versions coexist
2. **Tombstones**: Deleted keys remain until compacted away
3. **Overhead**: Indices, Bloom filters, metadata

**Typical space amplification**: 1.1x - 1.5x (10-50% overhead)

**Mitigation**:
- More aggressive compaction
- Time-to-live (TTL) for old versions
- Garbage collection policies

---

## Chapter 20: Advanced Topics

### Compaction Scheduling

**Challenge**: Compaction competes with user requests for I/O bandwidth.

**Strategies**:
1. **Throttling**: Limit compaction rate based on write rate
2. **Priorities**: User reads > user writes > compaction
3. **Time-based**: Run heavy compaction during off-peak hours
4. **Adaptive**: Monitor I/O latency, adjust compaction dynamically

### Crash Recovery

**Problem**: What happens if we crash mid-operation?

**Write-Ahead Log (WAL)**:
```
Every write:
1. Append to WAL (durable sequential write)
2. Insert into memTable (volatile)
3. Return success

On crash recovery:
1. Replay WAL entries
2. Rebuild memTable
3. Resume normal operation
```

**WAL properties**:
- Sequential writes (fast)
- Synced to disk (durable)
- Discarded after memTable flush

### Concurrency Control

**Challenges**:
- Multiple threads writing
- Reads during compaction
- Flushes during writes

**Solutions**:

**MemTable**:
- Concurrent data structure (skip list supports concurrent reads/writes)
- Or single-writer, multi-reader with locks

**SSTables**:
- Immutable = no locks needed for reads
- Reference counting for file lifetime management

**Compaction**:
- Background threads
- No locks on SSTables (work on copies)
- Atomic metadata updates

### Snapshots

**Use case**: Consistent point-in-time views

**Implementation**:
```
Creating snapshot:
1. Record current memTable version
2. Record set of SSTables
3. Prevent their deletion during compaction

Reading snapshot:
1. Read from frozen memTable version
2. Read from frozen SSTable set
3. Ignore newer writes
```

**Cost**: Minimal (just metadata)

---

## Conclusion: What We've Discovered

### The Journey

We started with a simple problem: store key-value pairs efficiently.

We tried the obvious solution: in-memory hash table.

We hit limitations: persistence, scale.

We solved them one by one:
- Append-only log for fast writes
- MemTable for fast recent reads
- SSTables for persistent sorted data
- Compaction for managing file count
- Levels for organizing efficiently
- Bloom filters for read optimization

**We invented the LSM tree.**

### The Meta-Lesson

**The LSM tree isn't just a data structure. It's a case study in problem-solving.**

Key insights:
1. **Constraints drive creativity**: Sequential I/O performance shaped everything
2. **Trade-offs are fundamental**: We chose write speed over read speed
3. **Simplicity compounds**: Each piece is simple; elegance emerges from composition
4. **Iteration works**: We didn't need the perfect solution upfront
5. **Understanding beats memorization**: Now you know *why*, not just *what*

### The Broader Pattern

This thought process applies to all engineering:

**Building a cache?**
- Start simple (hash table)
- Hit capacity limits
- Add eviction (LRU)
- Optimize for your access pattern

**Designing an API?**
- Start with basic CRUD
- Identify common patterns
- Abstract them
- Iterate based on usage

**Creating a distributed system?**
- Start with single machine
- Identify bottlenecks
- Partition carefully
- Handle failures incrementally

**The pattern**: naive → measure → bottleneck → solution → iterate

### Your Next Steps

**Implement it**:
- Build a simple LSM tree
- Feel the trade-offs directly
- Experiment with parameters

**Study implementations**:
- Read LevelDB source code
- Compare RocksDB optimizations
- Understand production lessons

**Apply the mindset**:
- Approach new problems from first principles
- Start simple, iterate
- Embrace trade-offs
- Build intuition through experimentation

### Final Thoughts

The LSM tree is beautiful not because it's complex, but because it's *necessarily complex*. Each piece exists for a reason. Each layer solves a specific problem.

**You now understand not just how LSM trees work, but how to think about complex systems.**

When you encounter a new data structure, a new algorithm, a new system—don't just memorize it. Ask:
- What problem does it solve?
- What constraints does it face?
- What trade-offs does it make?
- Could I have invented this?

**Often, you'll find you could have. And that's the point.**

---

## Appendix A: Implementation Sketch

Here's a simplified implementation to solidify concepts:

```python
from sortedcontainers import SortedDict
from pathlib import Path
import pickle

class SimpleLSM:
    def __init__(self, data_dir):
        self.data_dir = Path(data_dir)
        self.memtable = SortedDict()
        self.wal = open(self.data_dir / "wal.log", "ab")
        self.sstables = []
        self.memtable_threshold = 1000

    def put(self, key, value):
        # Write to WAL for durability
        self.wal.write(pickle.dumps(("PUT", key, value)))
        self.wal.flush()

        # Write to memtable
        self.memtable[key] = value

        # Flush if threshold exceeded
        if len(self.memtable) >= self.memtable_threshold:
            self._flush_memtable()

    def get(self, key):
        # Check memtable first
        if key in self.memtable:
            return self.memtable[key]

        # Check SSTables (newest to oldest)
        for sstable in reversed(self.sstables):
            if key in sstable:
                return sstable[key]

        return None  # Not found

    def _flush_memtable(self):
        # Create new SSTable from memtable
        sstable_path = self.data_dir / f"sstable_{len(self.sstables)}.db"

        with open(sstable_path, "wb") as f:
            pickle.dump(dict(self.memtable), f)

        # Load it back (in real impl, would keep on disk)
        with open(sstable_path, "rb") as f:
            sstable = pickle.load(f)

        self.sstables.append(sstable)

        # Clear memtable and WAL
        self.memtable.clear()
        self.wal.close()
        self.wal = open(self.data_dir / "wal.log", "wb")

        # Trigger compaction if too many SSTables
        if len(self.sstables) > 4:
            self._compact()

    def _compact(self):
        # Simple compaction: merge all SSTables
        merged = {}
        for sstable in self.sstables:
            merged.update(sstable)

        # Write merged SSTable
        sstable_path = self.data_dir / f"sstable_compacted.db"
        with open(sstable_path, "wb") as f:
            pickle.dump(merged, f)

        # Replace old SSTables
        self.sstables = [merged]

# Usage
lsm = SimpleLSM("/tmp/lsm_data")
lsm.put("apple", "red fruit")
lsm.put("banana", "yellow fruit")
print(lsm.get("apple"))  # "red fruit"
```

**This is simplified but demonstrates**:
- MemTable (SortedDict)
- Write-ahead log
- Flushing to SSTables
- Basic compaction

**Missing from this sketch**:
- Proper SSTable format with blocks and indices
- Bloom filters
- Leveled compaction
- Concurrency control
- Efficient merging
- Error handling

**For production implementation, study**: RocksDB, LevelDB source code

---

## Appendix B: Further Reading

### Academic Papers

**Original LSM Tree Paper**:
- O'Neil, P., et al. (1996). "The Log-Structured Merge-Tree (LSM-Tree)." *Acta Informatica*, 33(4), 351-385.
  - The foundational paper

**Modern Variations**:
- Lu, L., et al. (2016). "WiscKey: Separating Keys from Values in SSD-conscious Storage." *FAST*.
- Raju, P., et al. (2017). "PebblesDB: Building Key-Value Stores using Fragmented Log-Structured Merge Trees." *SOSP*.

### Books

**Database Internals** by Alex Petrov:
- Chapter 7: Log-Structured Storage
- Comprehensive coverage with diagrams

**Designing Data-Intensive Applications** by Martin Kleppmann:
- Chapter 3: Storage and Retrieval
- Practical perspective on LSM trees vs B-trees

### Source Code

**LevelDB**:
- https://github.com/google/leveldb
- ~10,000 lines of clear C++
- Great for learning

**RocksDB**:
- https://github.com/facebook/rocksdb
- Production-hardened
- Extensive documentation

### Online Resources

**RocksDB Wiki**:
- https://github.com/facebook/rocksdb/wiki
- Tuning guides
- Architecture explanations

**Database Internals Blog**:
- http://www.databass.dev
- Deep dives into storage engines

---

## Appendix C: Glossary

**Bloom Filter**: Probabilistic data structure for set membership testing

**Compaction**: Process of merging multiple SSTables into fewer SSTables

**Memtable**: In-memory sorted buffer for recent writes

**SSTable**: Sorted String Table; immutable sorted file on disk

**Tombstone**: Marker indicating a key has been deleted

**WAL**: Write-Ahead Log; durable log for crash recovery

**Write Amplification**: Ratio of data written to disk vs. data written by user

**Read Amplification**: Number of disk reads required for one query

**Space Amplification**: Extra space used beyond actual data size

**Level**: Tier in the LSM tree hierarchy, each 10x larger than the previous

**Flush**: Writing memtable to disk as an SSTable

**Minor Compaction**: Flushing memtable to Level 0

**Major Compaction**: Merging SSTables between levels

---

## About This Book

This book was created as an experiment in **first-principles learning**: teaching complex topics by deriving them from scratch rather than explaining them top-down.

The goal: help students develop **problem-solving intuition** that transfers beyond this specific data structure.

**Feedback**: If this approach helped you learn, or if you have suggestions for improvement, please share your thoughts.

**Version**: 1.0
**Date**: November 2025
**License**: Creative Commons BY-SA 4.0

---

*"The best way to understand something is to try to build it yourself."*

*"Genius is not about having the answer, but about asking the right next question."*

**Thank you for taking this journey. Now go build something amazing.**
