# Chapter 10: The Consumer's World

> **"Consuming data is where the value is realized. Get it wrong, and all the perfect production means nothing."**

## The Consumer's Responsibilities

A consumer must:
1. **Discover** topic metadata and partition assignments
2. **Fetch** data from brokers
3. **Deserialize** messages
4. **Process** messages
5. **Commit** offsets (track progress)
6. **Handle rebalances** (when group membership changes)
7. **Deal with failures** (retries, dead letter queues)

This is more complex than producing. Let's dive deep.

## Consumer Configuration

```javascript
const consumer = new KafkaConsumer({
    groupId: 'order-processor',
    clientId: 'processor-instance-1',
    brokers: ['localhost:9092'],

    // Fetch configuration
    fetchMinBytes: 1,               // Min bytes to wait for
    fetchMaxWaitMs: 500,            // Max time to wait for minBytes
    maxPartitionFetchBytes: 1048576, // 1 MB per partition

    // Session and heartbeat
    sessionTimeoutMs: 10000,        // 10 seconds
    heartbeatIntervalMs: 3000,      // 3 seconds

    // Offset management
    enableAutoCommit: false,        // Manual commit recommended
    autoCommitIntervalMs: 5000,     // If auto-commit enabled

    // Processing
    maxPollRecords: 500,            // Max records per poll()
    maxPollIntervalMs: 300000,      // 5 minutes max between polls
});
```

Let's understand each setting.

## The Fetch Mechanism

### Pull vs Push

**Kafka uses pull** (consumer pulls data from broker):

```
Consumer:               Broker:
  |                       |
  |--- Fetch request ---->|
  |                       | (has data? send. no data? wait)
  |<--- Fetch response ---|
  |                       |
  | Process data          |
  |                       |
  |--- Fetch request ---->|
```

**Why pull?**

1. **Consumer controls rate** - Natural backpressure
2. **Batching** - Consumer can request large batches
3. **Simplicity** - Broker doesn't track consumer state

**Contrast with push:**
- Broker pushes data to consumer
- Risk of overwhelming consumer
- Broker needs to track consumer state

### Fetch Configuration

```javascript
{
    fetchMinBytes: 1024,      // Wait for at least 1 KB
    fetchMaxWaitMs: 500,      // But no more than 500ms
}
```

**How it works:**

```
Consumer sends fetch request with offset 1000

Broker checks:
  - Do I have at least 1024 bytes after offset 1000?
    YES → Send immediately
    NO → Wait up to 500ms for more data to accumulate
         If timeout, send whatever is available
```

**Tuning for throughput:**

```javascript
{
    fetchMinBytes: 1048576,   // Wait for 1 MB
    fetchMaxWaitMs: 1000,     // Up to 1 second
}
```

Larger batches, fewer fetch requests, higher throughput.

**Tuning for latency:**

```javascript
{
    fetchMinBytes: 1,         // Don't wait
    fetchMaxWaitMs: 100,      // Short timeout
}
```

Get data as soon as it's available.

### Max Fetch Size

```javascript
{
    maxPartitionFetchBytes: 1048576,  // 1 MB per partition
}
```

If consumer reads from 10 partitions:
- Max fetch size = 10 × 1 MB = 10 MB

**Important:** If a single message is larger than this, the fetch will still succeed (with just that one message).

## Consumer Groups and Rebalancing

### Group Coordinator

Each consumer group has a **group coordinator** (one of the brokers):

```
Consumer Group "order-processors":
  Coordinator: Broker 2

  Members:
    - consumer-1 (partitions: 0, 1)
    - consumer-2 (partitions: 2, 3)
    - consumer-3 (partitions: 4, 5)
```

Coordinator handles:
- **Member liveness** (heartbeat tracking)
- **Rebalancing** (partition assignment)
- **Offset commits** (stores in `__consumer_offsets`)

### Heartbeat Protocol

```javascript
{
    sessionTimeoutMs: 10000,      // 10 seconds
    heartbeatIntervalMs: 3000,    // 3 seconds
}
```

**Consumer sends heartbeats every 3 seconds.**

If coordinator doesn't receive heartbeat within 10 seconds:
- Consumer is considered dead
- Rebalance triggered

**Important:** Heartbeats are sent in a **background thread**.

But if `poll()` is not called within `maxPollIntervalMs`:
- Consumer is considered stuck (processing too slow)
- Kicked out of group, rebalance triggered

```javascript
{
    maxPollIntervalMs: 300000,  // 5 minutes
}
```

**Rule:** Time between `poll()` calls must be < `maxPollIntervalMs`.

### Rebalance Strategies

When rebalancing, how are partitions assigned?

**1. RangeAssignor (default):**

```
Topic: orders (6 partitions: 0, 1, 2, 3, 4, 5)
Consumers: A, B, C

Assignment:
  A: 0, 1
  B: 2, 3
  C: 4, 5
```

Simple, but can cause imbalance with multiple topics.

**2. RoundRobinAssignor:**

```
All partitions: [orders-0, orders-1, ..., inventory-0, inventory-1, ...]
Consumers: A, B, C

Assignment (round-robin):
  A: orders-0, inventory-1, ...
  B: orders-1, inventory-2, ...
  C: orders-2, inventory-0, ...
```

More balanced, but can break co-partitioning.

**3. StickyAssignor:**

Minimizes partition movement during rebalance:

```
Initial:
  A: 0, 1, 2
  B: 3, 4, 5

Consumer C joins:

RangeAssignor (all partitions reassigned):
  A: 0, 1
  B: 2, 3
  C: 4, 5

StickyAssignor (minimal movement):
  A: 0, 1
  B: 3, 4
  C: 2, 5
```

**Only moves what's necessary.** Reduces disruption.

**4. CooperativeStickyAssignor (modern, recommended):**

Same as Sticky, but uses **incremental cooperative rebalancing**:
- Only revokes partitions being reassigned
- Other partitions keep processing
- **No stop-the-world pause**

```javascript
const consumer = new KafkaConsumer({
    groupId: 'my-group',
    partitionAssignmentStrategy: 'CooperativeSticky',  // Recommended
});
```

**Use this in Kafka 2.4+.**

### Static Membership

To avoid rebalancing on consumer restart:

```javascript
const consumer = new KafkaConsumer({
    groupId: 'my-group',
    groupInstanceId: 'consumer-1',  // Static ID
});
```

**Benefits:**
- Consumer restarts without rebalancing (as long as it rejoins within session timeout)
- Useful for rolling deploys

**Trade-off:** If consumer truly fails, must wait for session timeout before rebalancing.

## Offset Management

### Commit Strategies Revisited

**Auto-commit (easy, risky):**

```javascript
const consumer = new KafkaConsumer({
    enableAutoCommit: true,
    autoCommitIntervalMs: 5000,
});

while (true) {
    const messages = await consumer.poll();

    for (const msg of messages) {
        await process(msg);
    }

    // Auto-commit happens in background every 5 seconds
}
```

**Risk:** Message loss if consumer crashes after commit but before processing.

**Manual commit after processing (recommended):**

```javascript
const consumer = new KafkaConsumer({
    enableAutoCommit: false,
});

while (true) {
    const messages = await consumer.poll();

    for (const msg of messages) {
        await process(msg);
    }

    // Commit only after all messages processed
    await consumer.commitSync();
}
```

**Risk:** Duplicates if consumer crashes after processing but before commit (at-least-once).

**Manual commit per message (slow but precise):**

```javascript
while (true) {
    const messages = await consumer.poll();

    for (const msg of messages) {
        await process(msg);
        await consumer.commitSync({ offset: msg.offset + 1 });
    }
}
```

**Risk:** Very slow (many commits).

**Balanced approach (batch commit):**

```javascript
const batchSize = 100;
let count = 0;

while (true) {
    const messages = await consumer.poll();

    for (const msg of messages) {
        await process(msg);
        count++;

        if (count >= batchSize) {
            await consumer.commitSync();
            count = 0;
        }
    }
}
```

**Trade-off:** Up to `batchSize` duplicates on failure (still at-least-once, but fewer duplicates).

### Async vs Sync Commit

**Sync commit:**

```javascript
await consumer.commitSync();  // Blocks until commit succeeds
```

**Safe but slow** (waits for broker response).

**Async commit:**

```javascript
await consumer.commitAsync((err, offsets) => {
    if (err) {
        console.error('Commit failed:', err);
    }
});
```

**Fast but risky** (doesn't wait, fire-and-forget).

**Best practice:** Use async during normal processing, sync before closing:

```javascript
while (running) {
    const messages = await consumer.poll();
    await processMessages(messages);
    await consumer.commitAsync();  // Fast
}

// On shutdown
await consumer.commitSync();  // Ensure final commit succeeds
await consumer.close();
```

## Error Handling

### Retriable Errors

Some errors are temporary:
- Network timeouts
- Coordinator not available
- Rebalance in progress

**Consumer automatically retries these.**

### Non-Retriable Errors

Some errors require intervention:
- Deserialization failure (malformed data)
- Processing logic error

**You need to handle these:**

```javascript
while (true) {
    const messages = await consumer.poll();

    for (const msg of messages) {
        try {
            await process(msg);
        } catch (error) {
            if (error.retriable) {
                // Retry with backoff
                await sleep(1000);
                await process(msg);
            } else {
                // Log and skip, or send to dead letter queue
                await sendToDeadLetterQueue(msg, error);
            }
        }
    }

    await consumer.commitSync();
}
```

### Dead Letter Queue (DLQ)

For messages that can't be processed:

```javascript
async function handleFailedMessage(msg, error) {
    await producer.send({
        topic: 'orders-dlq',  // Dead letter queue topic
        value: {
            originalMessage: msg.value,
            error: error.message,
            timestamp: Date.now(),
        },
    });

    // Log for monitoring
    logger.error('Message sent to DLQ', { msg, error });
}
```

**Separate process can:**
- Alert on DLQ messages
- Manually inspect and fix
- Retry after fixing underlying issue

## Exactly-Once with Transactions

For exactly-once end-to-end (read from Kafka, process, write to Kafka):

```javascript
// Consumer
const consumer = new KafkaConsumer({
    groupId: 'processor',
    isolationLevel: 'read_committed',  // Only read committed transactions
});

// Producer
const producer = new KafkaProducer({
    transactionalId: 'processor-v1',
    enableIdempotence: true,
});

await producer.initTransactions();

while (true) {
    const messages = await consumer.poll();

    await producer.beginTransaction();

    try {
        // Process and produce
        for (const msg of messages) {
            const result = await process(msg);
            await producer.send({ topic: 'results', value: result });
        }

        // Commit offsets as part of transaction
        await producer.sendOffsetsToTransaction(
            consumer.getOffsets(),
            consumer.groupId
        );

        await producer.commitTransaction();

    } catch (error) {
        await producer.abortTransaction();
    }
}
```

**Guarantees:**
- Consume → process → produce happens atomically
- No duplicates, no lost data
- **Exactly-once semantics**

**Trade-off:**
- More complex
- Higher latency
- Only works Kafka-to-Kafka

## Seeking and Replay

### Manual Offset Control

```javascript
// Seek to specific offset
consumer.seek({ topic: 'orders', partition: 0, offset: 1000 });

// Seek to beginning
consumer.seekToBeginning();

// Seek to end (skip everything)
consumer.seekToEnd();

// Seek by timestamp
consumer.seekToTimestamp({ topic: 'orders', partition: 0, timestamp: Date.now() - 86400000 });
```

**Use cases:**
- Reprocess after bug fix
- Start from specific point in time
- Skip ahead after incident

### Offset Reset Policy

What if consumer starts for first time, or committed offset is invalid?

```javascript
const consumer = new KafkaConsumer({
    autoOffsetReset: 'earliest',  // or 'latest' or 'none'
});
```

- **`earliest`**: Start from beginning (read all history)
- **`latest`**: Start from end (only new messages)
- **`none`**: Throw exception (force explicit handling)

**Typical:**
- Analytics: `earliest` (want all data)
- Real-time alerts: `latest` (only care about new events)

## Consumer Lag

**Lag = (Partition high-water mark) - (Consumer committed offset)**

```
Partition high-water mark: 10,000 (latest message)
Consumer committed offset: 9,500 (last processed)
Lag: 500 messages
```

**Lag is the most critical consumer metric.**

**Healthy:** Lag < 1000 messages (or < 1 minute of data)
**Unhealthy:** Lag > 100,000 messages (or > 1 hour)

**Causes of high lag:**
1. Consumer too slow (processing bottleneck)
2. Not enough consumers (need more parallelism)
3. Not enough partitions (can't add more consumers)
4. Spike in traffic (temporary)

**Solutions:**
1. Optimize processing logic
2. Add more consumers (up to number of partitions)
3. Increase partitions (requires rebalancing, careful!)
4. Tune fetch settings (larger batches)

## Graceful Shutdown

```javascript
let running = true;

process.on('SIGTERM', () => {
    console.log('Shutting down gracefully...');
    running = false;
});

while (running) {
    const messages = await consumer.poll({ timeout: 1000 });

    if (messages.length === 0) continue;

    await processMessages(messages);
    await consumer.commitSync();
}

// Graceful shutdown
await consumer.commitSync();  // Final commit
await consumer.close();       // Leave group cleanly
console.log('Shutdown complete');
```

**Avoids:**
- Lost messages (uncommitted offsets)
- Unnecessary rebalancing (clean group exit)

---

## Key Takeaways

- **Pull model enables backpressure** - Consumer controls rate
- **Fetch tuning affects latency vs throughput** - `fetchMinBytes` and `fetchMaxWaitMs`
- **Heartbeats vs poll interval** - Two failure detection mechanisms
- **Cooperative rebalancing reduces disruption** - Use CooperativeStickyAssignor
- **Manual commit after processing** - For at-least-once (recommended)
- **Transactions for exactly-once** - Kafka-to-Kafka only
- **Monitor consumer lag** - Key health indicator
- **Graceful shutdown matters** - Clean exit, final commit

## Configuration Cheat Sheet

**For high throughput:**
```javascript
{
    fetchMinBytes: 1048576,        // 1 MB
    fetchMaxWaitMs: 1000,          // 1 second
    maxPartitionFetchBytes: 10485760,  // 10 MB
    maxPollRecords: 10000,
}
```

**For low latency:**
```javascript
{
    fetchMinBytes: 1,
    fetchMaxWaitMs: 100,
    maxPollRecords: 100,
}
```

**For reliability:**
```javascript
{
    enableAutoCommit: false,       // Manual commit
    isolationLevel: 'read_committed',  // With transactions
    partitionAssignmentStrategy: 'CooperativeSticky',
}
```

---

**[← Previous: The Producer's Perspective](./09-producer-perspective.md) | [Back to Contents](./README.md) | [Next: When Kafka Shines →](./11-when-kafka-shines.md)**
