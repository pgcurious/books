# Chapter 4: The Append-Only Revelation

> **"The simplest data structure is often the most powerful: the humble log."**

## Back to Basics

Let's strip everything away and start with the simplest possible thing: a file.

```
# orders.log
{"event": "ORDER_PLACED", "orderId": 1, "amount": 29.99, "timestamp": "2025-01-15T10:30:00Z"}
{"event": "ORDER_PLACED", "orderId": 2, "amount": 49.99, "timestamp": "2025-01-15T10:31:00Z"}
{"event": "ORDER_PLACED", "orderId": 3, "amount": 19.99, "timestamp": "2025-01-15T10:32:00Z"}
```

That's it. Just append each event as a line in a file.

**This seems too simple to work at scale, right?**

Let's explore why it's actually brilliant.

## The Properties of a Log

An append-only log has beautiful properties:

### 1. Total Order

Events are written in sequence. The position in the file defines the order:

```
Offset 0: {"event": "ORDER_PLACED", "orderId": 1, ...}
Offset 1: {"event": "ORDER_PLACED", "orderId": 2, ...}
Offset 2: {"event": "ORDER_PLACED", "orderId": 3, ...}
```

The **offset** (position) is a natural, monotonically increasing identifier.

No database sequences. No locks. Just: "this came before that."

### 2. Immutability

Once written, events never change. No updates. No deletes (yet).

This means:
- No lock contention on reads and writes
- No complex transaction isolation
- Perfect for concurrent readers

### 3. Sequential I/O

Appending to a file is **incredibly fast**:

**Random I/O (database pattern):**
- Seek to index
- Read index
- Seek to data
- Read data
- **~100-200 IOPS (I/O operations per second)** on HDD
- **~10,000 IOPS** on SSD

**Sequential I/O (log pattern):**
- Just append to end
- **~1000 MB/sec** on HDD
- **~3000 MB/sec** on SSD

Sequential writes are **orders of magnitude faster**.

Even on modern SSDs, sequential I/O beats random I/O by 2-3x. On HDDs (which are still common in data centers for cost), it's 100x or more.

### 4. OS Page Cache is Your Friend

Modern operating systems cache file reads in RAM (page cache). When you read a log sequentially:

1. First read: Loads data into page cache
2. Subsequent reads: Served from RAM (microseconds, not milliseconds)

With multiple readers reading the same recent data, they all benefit from the same cached pages. **Zero-copy optimization** becomes possible (more on this later).

### 5. Simple Retention

Want to keep only the last 7 days of events?

```bash
# Delete log segments older than 7 days
find /logs -name "orders-*.log" -mtime +7 -delete
```

No expensive DELETE queries. No vacuum. Just delete old files.

## The Write Path

Let's implement a simple log writer:

```javascript
class CommitLog {
    constructor(filepath) {
        this.filepath = filepath;
        this.fd = fs.openSync(filepath, 'a'); // Open for append
        this.offset = 0;
    }

    append(message) {
        const messageBuffer = Buffer.from(JSON.stringify(message) + '\n');
        const offset = this.offset;

        // Write to file
        fs.writeSync(this.fd, messageBuffer);

        // Force to disk (durability)
        fs.fsyncSync(this.fd);

        this.offset++;
        return offset;
    }
}
```

**That's it for writes.** Simple, fast, durable.

### The fsync Trade-off

Notice the `fsync()` call. This forces data from the OS buffer to physical disk, ensuring durability.

But `fsync()` is **slow** (~5-10ms per call). If we fsync after every message, we're limited to ~100-200 writes/sec.

**The insight: Batch writes.**

```javascript
class CommitLog {
    constructor(filepath) {
        this.filepath = filepath;
        this.fd = fs.openSync(filepath, 'a');
        this.offset = 0;
        this.buffer = [];
    }

    append(message) {
        const offset = this.offset;
        this.buffer.push(message);
        this.offset++;

        // Return a promise that resolves when flushed
        return new Promise((resolve) => {
            this.pendingWrites.set(offset, resolve);
        });
    }

    // Flush every 10ms or every 1000 messages
    flush() {
        if (this.buffer.length === 0) return;

        const batch = this.buffer.join('\n') + '\n';
        fs.writeSync(this.fd, Buffer.from(batch));
        fs.fsyncSync(this.fd);

        // Resolve all pending promises
        this.pendingWrites.forEach((resolve) => resolve());
        this.pendingWrites.clear();

        this.buffer = [];
    }
}
```

Now we can do **1000 writes with one fsync**. Throughput increases to **100,000 writes/sec** on commodity hardware.

**This is exactly what Kafka does.**

## The Read Path

Reading is even simpler:

```javascript
class LogReader {
    constructor(filepath) {
        this.filepath = filepath;
        this.currentOffset = 0;
    }

    read(fromOffset, maxMessages = 100) {
        const messages = [];
        const lines = fs.readFileSync(this.filepath, 'utf8')
                        .split('\n')
                        .slice(fromOffset, fromOffset + maxMessages);

        for (let i = 0; i < lines.length; i++) {
            if (lines[i].trim()) {
                messages.push({
                    offset: fromOffset + i,
                    data: JSON.parse(lines[i])
                });
            }
        }

        return messages;
    }
}
```

Each reader:
- Tracks its own offset
- Reads from that offset
- Updates its offset as it processes

**Multiple readers, same log, different offsets. Perfect.**

## The Index Problem

Wait, there's an issue. To read from offset 1,000,000, we need to scan the entire file?

**Yes, that would be slow.**

**Solution: Sparse index.**

Every N messages (say, every 1000), write an index entry:

```
# orders.index
Offset 0 → File Position 0
Offset 1000 → File Position 87432
Offset 2000 → File Position 174856
Offset 3000 → File Position 262391
```

To read from offset 1,500:
1. Check index: Offset 1000 is at file position 87432
2. Seek to position 87432
3. Scan forward 500 messages

**This is fast.** Index lookup is O(log N), scanning 500 messages is trivial.

The index is small (one entry per 1000 messages) and stays in memory.

**Kafka does exactly this.**

## Log Segments

A single file growing forever has problems:
- Can't delete old data efficiently
- File system limits
- Can't parallelize operations

**Solution: Log segments.**

Split the log into segments:

```
orders-00000000000000000000.log  (offsets 0-999)
orders-00000000000000001000.log  (offsets 1000-1999)
orders-00000000000000002000.log  (offsets 2000-2999)
orders-00000000000000003000.log  (offsets 3000-3999) ← Active
```

Each segment has a corresponding index:

```
orders-00000000000000000000.index
orders-00000000000000001000.index
orders-00000000000000002000.index
orders-00000000000000003000.index
```

**Benefits:**

1. **Retention**: Delete old segments (just delete files)
2. **Compaction**: Compress old segments
3. **Parallelism**: Different segments can be accessed independently

**This is the Kafka log structure.**

## Putting It Together

Your event log now looks like:

```
/data/orders/
  ├── 00000000000000000000.log
  ├── 00000000000000000000.index
  ├── 00000000000000001000.log
  ├── 00000000000000001000.index
  ├── 00000000000000002000.log (active segment)
  └── 00000000000000002000.index
```

**Producer:**
- Appends to active segment
- Flushes batch periodically
- Rotates to new segment when size limit reached

**Consumer:**
- Tracks its offset
- Reads from appropriate segment
- Benefits from OS page cache

**Retention:**
- Delete segments older than N days
- Or keep only last X GB of data

## Performance Characteristics

Let's compare to what we had before:

| Operation | Database | Message Queue | Append-Only Log |
|-----------|----------|---------------|-----------------|
| Write latency | 5-10ms | 2-5ms | 0.1-1ms (batched) |
| Write throughput | ~10K/sec | ~50K/sec | ~100K+/sec |
| Read latency | 5-10ms | 2-5ms | 0.1-1ms (cached) |
| Multiple readers | Read amplification | Complex routing | Native support |
| Replay capability | Query history | No | Yes, trivial |
| Retention | DELETE operations | No history | Delete files |
| Scalability | Vertical | Limited | Horizontal (next chapter) |

**The append-only log wins on almost every dimension.**

## But Wait, There's More

We've solved the single-machine problem. But what about:

1. **Scale beyond one machine** - What if one server can't handle the write load?
2. **Durability** - What if the machine crashes?
3. **High availability** - What if we need zero downtime?

These questions lead us to **distribution**, which we'll tackle next.

---

## Key Takeaways

- **Append-only logs are deceptively powerful** - Simple but high-performance
- **Sequential I/O is orders of magnitude faster** - Than random I/O
- **Batching enables high throughput** - One fsync for many writes
- **OS page cache is a free cache layer** - Leverage it for reads
- **Sparse indexes make seeking fast** - O(log N) lookup
- **Segments enable efficient retention** - Delete old files, not rows
- **Each reader tracks its own offset** - No coordination needed

## The Core Innovation

Kafka's genius isn't inventing the log (logs are ancient). It's recognizing that:

> **"A distributed, replicated, partitioned commit log can be the central abstraction for data integration."**

We've built the log. Next, we'll distribute it.

---

**[← Previous: The Database Trap](./03-the-database-trap.md) | [Back to Contents](./README.md) | [Next: The Distribution Challenge →](./05-distribution-challenge.md)**
