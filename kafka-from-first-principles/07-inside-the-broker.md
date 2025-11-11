# Chapter 7: Inside the Broker

> **"Performance isn't magic. It's knowing your hardware and not fighting it."**

## What is a Broker?

A Kafka broker is a server that:
1. **Stores log segments** on disk
2. **Serves produce requests** from producers
3. **Serves fetch requests** from consumers
4. **Replicates data** to/from other brokers
5. **Coordinates metadata** with the cluster

It's the workhorse of Kafka. Let's understand how it achieves insane performance.

## The Storage Layer

### Log Segment Files

Remember our log structure?

```
/var/lib/kafka/orders-0/
  ├── 00000000000000000000.log     (1 GB, full, read-only)
  ├── 00000000000000000000.index
  ├── 00000000000000000000.timeindex
  ├── 00000000000001000000.log     (1 GB, full, read-only)
  ├── 00000000000001000000.index
  ├── 00000000000001000000.timeindex
  └── 00000000000002000000.log     (500 MB, active, read-write)
      00000000000002000000.index
      00000000000002000000.timeindex
```

**Each segment has three files:**

1. **`.log` file** - The actual messages
2. **`.index` file** - Offset → file position mapping
3. **`.timeindex` file** - Timestamp → offset mapping

### The .log File Format

Messages are stored in a binary format:

```
[8 bytes: offset]
[4 bytes: message size]
[4 bytes: CRC (checksum)]
[1 byte: magic byte (version)]
[1 byte: attributes (compression, timestamp type)]
[8 bytes: timestamp]
[4 bytes: key length]
[N bytes: key]
[4 bytes: value length]
[M bytes: value]
[4 bytes: headers length]
[X bytes: headers]
```

**Key insights:**

- **Fixed-size header** - Easy to parse
- **CRC checksum** - Detect corruption
- **Compression at batch level** - Producer sends compressed batch, stored as-is
- **Zero-copy possible** - Can send file contents directly to network socket

### The .index File Format

Sparse index: one entry every N messages (default 4KB of log data):

```
[4 bytes: relative offset] [4 bytes: physical position]
[4 bytes: relative offset] [4 bytes: physical position]
...
```

Example:

```
Offset 0 → Position 0
Offset 523 → Position 4096
Offset 1047 → Position 8192
```

**To find offset 750:**
1. Binary search index: Offset 523 is at position 4096
2. Seek to position 4096 in .log file
3. Scan forward until offset 750

**This is fast:**
- Index is tiny (1 entry per 4KB of data)
- Index stays in memory (OS page cache)
- Binary search is O(log N)
- Sequential scan of 4KB is trivial

### The .timeindex File Format

Maps timestamp to offset:

```
[8 bytes: timestamp] [4 bytes: offset]
[8 bytes: timestamp] [4 bytes: offset]
...
```

**Use case:** Consumer wants events from "2025-01-15 10:00:00"

1. Binary search timeindex for timestamp
2. Get corresponding offset
3. Seek to that offset in log

**This enables time-based queries without scanning all data.**

## The Write Path (Producer → Broker)

When a producer sends a message:

### Step 1: Network Receive

```
Producer sends:
  - Topic: orders
  - Partition: 2
  - Messages: [batch of 1000 messages, compressed]
```

Broker's network thread receives this and hands off to request handler thread.

### Step 2: Validation

```java
// Simplified pseudo-code
public void handleProduceRequest(ProduceRequest request) {
    // Validate
    if (!isLeaderForPartition(request.partition)) {
        return ERROR_NOT_LEADER;
    }

    // Check CRC
    if (!request.verifyCRC()) {
        return ERROR_CORRUPT_MESSAGE;
    }

    // Append to log
    long offset = log.append(request.messages);

    // Return success
    return ProduceResponse(offset);
}
```

### Step 3: Append to Log

```java
public long append(byte[] messages) {
    // Allocate offset
    long baseOffset = nextOffset;
    nextOffset += countMessages(messages);

    // Write to active segment
    activeSegment.write(messages);

    // Update index (asynchronously, every 4KB)
    if (bytesWrittenSinceIndexUpdate > 4096) {
        index.append(baseOffset, activeSegment.position());
    }

    return baseOffset;
}
```

**Key optimizations:**

1. **Batching** - Producer sends batch, broker writes once
2. **No fsync by default** - Relies on OS flush (configurable)
3. **Append-only** - No seeks, pure sequential write

### Step 4: Replication (Background)

Leader writes to its log, then followers fetch and replicate (we'll cover this in detail next chapter).

### Step 5: Ack to Producer

Depending on `acks` config:

- `acks=0`: Don't wait for broker (fire and forget)
- `acks=1`: Wait for leader write (default)
- `acks=all`: Wait for all in-sync replicas (most durable)

## The Read Path (Consumer → Broker)

When a consumer fetches:

### Step 1: Fetch Request

```
Consumer sends:
  - Topic: orders
  - Partition: 2
  - Offset: 1,000,000
  - MaxBytes: 1 MB (fetch up to 1 MB of data)
```

### Step 2: Locate Data

```java
public byte[] fetch(long offset, int maxBytes) {
    // Find segment containing this offset
    LogSegment segment = findSegmentContainingOffset(offset);

    // Use index to find position
    int position = segment.index.lookup(offset);

    // Read from log file
    return segment.log.read(position, maxBytes);
}
```

### Step 3: Zero-Copy Transfer

**This is where Kafka's performance shines.**

Traditional read:

```
1. Read file from disk → OS buffer (kernel space)
2. Copy from OS buffer → Application buffer (user space)
3. Copy from Application buffer → Socket buffer (kernel space)
4. Send from Socket buffer → NIC

Total: 4 copies (2 in kernel, 2 in user space)
```

**Kafka uses `sendfile()` system call (zero-copy):**

```
1. Read file from disk → OS buffer (kernel space)
2. Send directly from OS buffer → NIC

Total: 2 copies (all in kernel space)
```

**Performance impact:**

- **70% reduction in CPU usage**
- **2-3x higher throughput**
- No garbage collection in JVM (no user-space buffers)

```java
// Pseudo-code using Java NIO
public void sendToConsumer(FileChannel fileChannel, SocketChannel socket) {
    fileChannel.transferTo(position, count, socket);
    // This uses sendfile() under the hood
}
```

### Step 4: Page Cache Magic

**The OS page cache is Kafka's secret weapon.**

When you read data:
1. First read: Disk → Page cache → Consumer (slow, ~10ms)
2. Subsequent reads: Page cache → Consumer (fast, ~0.1ms)

If you have 10 consumers reading the same recent data:
- First consumer causes disk read
- Next 9 consumers hit page cache

**This is why Kafka scales reads so well.**

Memory usage:

```
System: 128 GB RAM
JVM heap: 6 GB
Page cache: ~120 GB (remaining)
```

Kafka uses **small JVM heap** and lets OS manage page cache. This is counterintuitive for Java apps, but brilliant for Kafka's access pattern.

## Compression

Producers can compress batches:

```javascript
const producer = new KafkaProducer({
    compression: 'gzip' // or 'snappy', 'lz4', 'zstd'
});
```

**Compression happens at batch level:**

```
Producer:
  - Batch 1000 messages
  - Compress batch (1 MB → 200 KB)
  - Send compressed batch

Broker:
  - Receives compressed batch
  - Writes to disk AS-IS (still compressed)
  - No decompression!

Consumer:
  - Fetches compressed batch
  - Decompresses locally
```

**Benefits:**

- **5-10x reduction in disk usage**
- **5-10x reduction in network bandwidth**
- **Broker CPU saved** (no compression work)

**Trade-offs:**

- Producer CPU usage (compress)
- Consumer CPU usage (decompress)
- Slightly higher latency (compression time)

**Recommended:** Use `lz4` or `zstd` (fast, good compression ratio)

## Batching

Kafka batches aggressively:

### Producer Batching

```javascript
const producer = new KafkaProducer({
    batchSize: 16384, // 16 KB
    lingerMs: 10 // Wait up to 10ms to fill batch
});

// Send messages
producer.send({ topic: 'orders', value: event1 });
producer.send({ topic: 'orders', value: event2 });
// ... (many more)

// After 10ms or 16KB, whichever comes first, send batch
```

**Impact:**

- Single message: 1 network roundtrip = 1 message
- Batched: 1 network roundtrip = 1000 messages

**Throughput increase: 1000x**

### Broker Batching

Broker never writes one message at a time. Even a "single" produce request is treated as a batch.

### Consumer Batching

Consumer fetches in batches:

```javascript
const messages = consumer.poll(); // Returns 500-1000 messages typically
```

One network roundtrip fetches many messages.

## Log Compaction

For some topics, you don't want retention by time. You want **latest value per key**.

### Use Case: User Profile Updates

```
Messages:
  user_123 → {name: "Alice", email: "alice@old.com"}
  user_123 → {name: "Alice", email: "alice@new.com"}
  user_123 → {name: "Alice Smith", email: "alice@new.com"}
```

You only care about the latest state of `user_123`.

### Compaction Process

```
Before compaction:
[offset 0] user_123 → v1
[offset 1] user_456 → v1
[offset 2] user_123 → v2
[offset 3] user_123 → v3
[offset 4] user_789 → v1

After compaction:
[offset 1] user_456 → v1
[offset 3] user_123 → v3  (latest for user_123)
[offset 4] user_789 → v1
```

**Compaction preserves:**
- At least one value per key
- The latest value per key
- Offsets (no rewriting)

**Use cases:**
- Changelog topics (database CDC)
- Cache rebuilding
- State stores

**Config:**

```javascript
await admin.createTopic({
    topic: 'user-profiles',
    config: {
        'cleanup.policy': 'compact' // Instead of 'delete'
    }
});
```

## Performance Numbers (Real-World)

On commodity hardware (12 cores, 64GB RAM, 8x7200 RPM HDDs in RAID-10):

**Writes:**
- 800,000 messages/sec
- ~80 MB/sec (with compression)

**Reads (single consumer):**
- 900,000 messages/sec
- ~90 MB/sec

**Reads (multiple consumers, same recent data):**
- 3,000,000+ messages/sec (page cache hits)

**Latency:**
- p50: 2 ms
- p99: 10 ms
- p99.9: 30 ms

**Why so fast?**

1. Sequential I/O (100x faster than random)
2. Batching (1000x fewer syscalls)
3. Zero-copy (2-3x less CPU)
4. Page cache (10-100x faster for recent data)
5. Compression (5-10x less I/O)

---

## Key Takeaways

- **Append-only log segments** - Simple, fast storage
- **Sparse indexes** - Small, in-memory, O(log N) lookup
- **Zero-copy transfers** - Bypass user space, save CPU
- **Page cache is critical** - Most reads served from RAM
- **Compression at batch level** - Broker never decompresses
- **Batching everywhere** - Producer, broker, consumer
- **Sequential I/O wins** - Don't fight your hardware
- **Small JVM heap** - Let OS manage memory

## Why Kafka is Fast (Summary)

| Optimization | Impact |
|-------------|--------|
| Sequential I/O | 100x faster than random I/O |
| Batching | 1000x fewer network calls |
| Zero-copy | 70% less CPU, 2-3x throughput |
| Page cache | 10-100x faster for recent data |
| Compression | 5-10x less I/O |
| No data copying in broker | Low GC pressure |

**Combined effect: Several orders of magnitude faster than naive implementations.**

---

**[← Previous: The Consumer Conundrum](./06-consumer-conundrum.md) | [Back to Contents](./README.md) | [Next: The Replication Protocol →](./08-replication-protocol.md)**
