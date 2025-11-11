# Chapter 9: The Producer's Perspective

> **"Sending data is easy. Sending it reliably, quickly, and correctly is an engineering challenge."**

## The Producer's Job

A producer must:
1. **Serialize** messages
2. **Partition** them (decide which partition)
3. **Batch** them (for performance)
4. **Compress** them (optionally)
5. **Send** to the broker
6. **Handle failures** (retries, timeouts)
7. **Guarantee ordering** (if required)

Let's understand each step.

## Serialization

Kafka stores bytes. Your application works with objects. You need serialization.

```javascript
const producer = new KafkaProducer({
    keySerializer: 'string',    // or custom
    valueSerializer: 'json',    // or custom
});

await producer.send({
    topic: 'orders',
    key: '12345',                    // Will be: "12345" (bytes)
    value: { orderId: 12345, ... }  // Will be: JSON bytes
});
```

**Common serializers:**
- **String** - UTF-8 bytes
- **JSON** - Flexible, human-readable, larger size
- **Avro** - Schema-based, compact, versioned
- **Protobuf** - Schema-based, efficient
- **Custom** - Your own binary format

**Schema registries:**

For Avro/Protobuf, use a **Schema Registry** (like Confluent Schema Registry):

```
Producer:
  1. Register schema → Schema Registry → Returns schema ID
  2. Serialize data using schema
  3. Prepend schema ID to bytes
  4. Send to Kafka

Consumer:
  1. Read bytes from Kafka
  2. Extract schema ID
  3. Fetch schema from registry
  4. Deserialize using schema
```

**Benefits:**
- **Schema evolution** - Backward/forward compatibility
- **Validation** - Ensure data matches schema
- **Documentation** - Schema is self-documenting

## Partitioning

How does the producer decide which partition?

### Default Partitioner

```javascript
function partition(key, value, numPartitions) {
    if (key !== null) {
        // Hash-based partitioning
        return hash(key) % numPartitions;
    } else {
        // Round-robin or sticky partitioner
        return roundRobin(numPartitions);
    }
}
```

**With a key:**
```javascript
await producer.send({
    topic: 'orders',
    key: 'user_123',  // Same key → same partition
    value: event
});
```

All events with `key = 'user_123'` go to the same partition (ordering guaranteed).

**Without a key (Kafka 2.4+):**

Uses **Sticky Partitioner**:
- Pick a partition
- Send all messages to it until batch is full
- Switch to another partition

**Why sticky?**
- Better batching (more messages per batch)
- Higher throughput (fewer small batches)

**Old behavior (pre-2.4):** Round-robin every message (poor batching)

### Custom Partitioner

```javascript
class RegionPartitioner {
    partition(key, value, numPartitions) {
        const region = value.userRegion;

        if (region === 'US') return 0;
        if (region === 'EU') return 1;
        if (region === 'ASIA') return 2;

        return hash(key) % numPartitions;
    }
}

const producer = new KafkaProducer({
    partitioner: new RegionPartitioner()
});
```

**Use cases:**
- Geo-based routing
- Priority lanes (important events to partition 0)
- Custom business logic

## Batching

**Batching is critical for performance.**

### How Batching Works

```javascript
const producer = new KafkaProducer({
    batchSize: 16384,      // 16 KB (default)
    lingerMs: 0,           // Default: send immediately when batch full
});

producer.send({ topic: 'orders', value: event1 }); // Queued
producer.send({ topic: 'orders', value: event2 }); // Queued
producer.send({ topic: 'orders', value: event3 }); // Queued
// ... (batch fills up to 16 KB) ...
// Sent as one batch to broker
```

**Two triggers for sending a batch:**
1. **Batch size reached** - `batchSize` bytes accumulated
2. **Time limit reached** - `lingerMs` milliseconds elapsed

### Tuning for Throughput

```javascript
const producer = new KafkaProducer({
    batchSize: 1000000,    // 1 MB (larger batches)
    lingerMs: 100,         // Wait up to 100ms to fill batch
    compression: 'lz4',    // Compress batches
});
```

**Effect:**
- More messages per batch
- Fewer network calls
- Higher compression ratio (more data to compress)
- **10-100x throughput improvement**

**Trade-off:** Higher latency (up to 100ms extra)

### Tuning for Latency

```javascript
const producer = new KafkaProducer({
    batchSize: 1024,       // Small batches
    lingerMs: 0,           // Send immediately
    compression: 'none',   // No compression delay
});
```

**Effect:**
- Lower latency (~1-2ms)
- More network calls
- Lower overall throughput

**Use case:** Real-time applications (trading, monitoring)

### The Buffer Memory

Producer has a buffer pool:

```javascript
const producer = new KafkaProducer({
    bufferMemory: 33554432,  // 32 MB (default)
});
```

**What happens when buffer is full?**

1. **Block** (default) - `send()` blocks until space available
2. **Throw exception** - If `maxBlockMs` exceeded

```javascript
const producer = new KafkaProducer({
    bufferMemory: 33554432,
    maxBlockMs: 60000,  // Block up to 60 seconds
});

// If buffer full for > 60s, throws TimeoutException
await producer.send({ ... });
```

**Monitoring:** If you hit buffer limits, you're producing faster than Kafka can handle (increase throughput or add brokers).

## Compression

```javascript
const producer = new KafkaProducer({
    compression: 'lz4'  // 'gzip', 'snappy', 'zstd', 'none'
});
```

**Compression is per batch:**

```
Batch of 1000 messages (1 MB uncompressed)
↓
Compress with lz4
↓
Compressed batch (200 KB)
↓
Send to broker (saved 800 KB on network)
↓
Broker stores compressed (saved 800 KB on disk)
↓
Consumer fetches compressed (saved 800 KB on network)
↓
Consumer decompresses locally
```

### Compression Algorithms

| Algorithm | Speed | Ratio | CPU Use | Recommendation |
|-----------|-------|-------|---------|----------------|
| **none** | N/A | 1x | 0% | Low-latency, already compressed data |
| **lz4** | Fast | 2-3x | Low | **Best default choice** |
| **snappy** | Fast | 2-3x | Low | Alternative to lz4 |
| **gzip** | Slow | 3-5x | High | High compression, slow broker |
| **zstd** | Medium | 3-5x | Medium | **Best throughput/ratio balance** |

**Recommendation: Use `lz4` or `zstd`.**

## Reliability and Retries

### The Acks Setting

```javascript
const producer = new KafkaProducer({
    acks: 'all'  // 0, 1, or 'all'
});
```

- **`acks: 0`** - Fire and forget (no wait for broker)
  - **Latency:** ~0.1ms
  - **Durability:** None (data loss possible)

- **`acks: 1`** - Leader acknowledgment
  - **Latency:** ~2-5ms
  - **Durability:** Lose data if leader fails before replication

- **`acks: 'all'`** - All in-sync replicas
  - **Latency:** ~5-10ms
  - **Durability:** No data loss (if `min.insync.replicas` > 1)

**For critical data: Use `acks: 'all'`.**

### Retries

```javascript
const producer = new KafkaProducer({
    retries: 10,               // Number of retries
    retryBackoffMs: 100,       // Wait 100ms between retries
});
```

**Retriable errors:**
- Network timeouts
- Leader not available (during election)
- Not enough replicas

**Non-retriable errors:**
- Invalid topic
- Message too large
- Authorization failure

**Infinite retries:**

```javascript
const producer = new KafkaProducer({
    retries: Number.MAX_SAFE_INTEGER,  // Retry forever
    deliveryTimeoutMs: 120000,         // But timeout after 2 minutes total
});
```

Producer will retry until:
- Success
- Or `deliveryTimeoutMs` exceeded

**This is the recommended pattern** for critical data.

### Idempotent Producer

**Problem:** Retries can cause duplicates.

```
Producer sends message (offset 100)
↓
Broker receives, writes, sends ACK
↓
ACK lost in network
↓
Producer retries (thinks it failed)
↓
Broker receives again, writes at offset 101
↓
DUPLICATE!
```

**Solution: Idempotent producer**

```javascript
const producer = new KafkaProducer({
    enableIdempotence: true,  // Kafka 3.0+: enabled by default
});
```

**How it works:**

1. Producer gets a unique **producer ID** (PID)
2. Each message gets a **sequence number**
3. Broker tracks `(PID, sequence)` for each partition
4. If duplicate detected (same PID + sequence), broker de-duplicates

```
Producer sends: PID=123, Seq=5
Broker receives, writes (offset 100)
ACK lost
Producer retries: PID=123, Seq=5
Broker sees: "Already have Seq=5 for PID=123, skip write, send ACK"
```

**No duplicate!**

**Requirements for idempotence:**
- `retries` > 0
- `acks: 'all'`
- `maxInflightRequests: 5` or less

**Kafka 3.0+ enables this by default.**

## Ordering Guarantees

### Default Ordering

**Within a partition: Kafka guarantees order.**

```javascript
producer.send({ key: 'user_123', value: event1 });
producer.send({ key: 'user_123', value: event2 });
producer.send({ key: 'user_123', value: event3 });

// All go to same partition (same key)
// Stored in order: event1, event2, event3
```

### Ordering with Retries

**Problem:**

```
Producer sends message A
Producer sends message B
Message A fails, retries
Message B succeeds
Message A retry succeeds

Order in Kafka: B, A  (WRONG!)
```

**Solution: `maxInFlightRequestsPerConnection: 1`**

```javascript
const producer = new KafkaProducer({
    maxInFlightRequestsPerConnection: 1  // Only one batch in-flight
});
```

**Guarantees strict ordering, but:**
- Lower throughput (no pipelining)
- Higher latency

**Better solution: Idempotent producer**

```javascript
const producer = new KafkaProducer({
    enableIdempotence: true,              // Enables ordering
    maxInFlightRequestsPerConnection: 5,  // Can pipeline
});
```

Idempotent producer **ensures ordering even with retries and pipelining**.

**Kafka 3.0+: This is the default.**

## Transactions (Exactly-Once Semantics)

For advanced use cases: **produce and consume atomically**.

```javascript
const producer = new KafkaProducer({
    transactionalId: 'my-app-v1',  // Unique per producer instance
    enableIdempotence: true,
});

await producer.initTransactions();

try {
    await producer.beginTransaction();

    // Send multiple messages
    await producer.send({ topic: 'orders', value: order });
    await producer.send({ topic: 'audit', value: auditLog });

    // Commit consumer offsets (if consuming)
    await producer.sendOffsetsToTransaction(offsets, consumerGroupId);

    // Commit transaction
    await producer.commitTransaction();

} catch (error) {
    // Rollback on failure
    await producer.abortTransaction();
}
```

**Guarantees:**
- All messages in transaction are visible together (atomic)
- Or none are visible (if aborted)
- Consumer offsets are committed atomically with messages

**Use case:** Exactly-once stream processing (consume → transform → produce)

**Trade-off:** Higher latency, more complex

## Monitoring Producer Health

**Key metrics:**

```javascript
producer.on('metrics', (metrics) => {
    console.log('Record send rate:', metrics.recordSendRate);
    console.log('Record error rate:', metrics.recordErrorRate);
    console.log('Request latency avg:', metrics.requestLatencyAvg);
    console.log('Buffer available bytes:', metrics.bufferAvailableBytes);
});
```

**Watch for:**
- **High error rate** - Something wrong with broker or config
- **Low send rate** - Bottleneck somewhere
- **High latency** - Network or broker issue
- **Buffer exhaustion** - Producing faster than Kafka can handle

---

## Key Takeaways

- **Serialization matters** - Use schema registry for complex data
- **Partitioning determines parallelism** - Key-based or custom
- **Batching is critical** - Tune `batchSize` and `lingerMs`
- **Compression saves bandwidth and disk** - Use lz4 or zstd
- **`acks: 'all'` for durability** - With `min.insync.replicas`
- **Idempotent producer prevents duplicates** - Enabled by default in Kafka 3.0+
- **Ordering guaranteed per partition** - Idempotence helps with retries
- **Transactions for exactly-once** - Complex, but powerful

## Configuration Cheat Sheet

**For maximum throughput:**
```javascript
{
    batchSize: 1000000,
    lingerMs: 100,
    compression: 'zstd',
    acks: 1,
}
```

**For maximum reliability:**
```javascript
{
    acks: 'all',
    enableIdempotence: true,
    retries: Number.MAX_SAFE_INTEGER,
    maxInFlightRequestsPerConnection: 5,
}
```

**For minimum latency:**
```javascript
{
    batchSize: 1024,
    lingerMs: 0,
    compression: 'none',
    acks: 1,
}
```

**Recommended default (Kafka 3.0+):**
```javascript
{
    acks: 'all',
    enableIdempotence: true,  // Default
    compression: 'lz4',
    batchSize: 16384,
    lingerMs: 0,
}
```

---

**[← Previous: The Replication Protocol](./08-replication-protocol.md) | [Back to Contents](./README.md) | [Next: The Consumer's World →](./10-consumer-world.md)**
