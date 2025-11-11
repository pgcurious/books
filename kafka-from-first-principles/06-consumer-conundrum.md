# Chapter 6: The Consumer Conundrum

> **"Reading data is easy. Reading data correctly, at scale, with failures, is an art."**

## The Problem Space

You've built a distributed log. Producers write to it. Now you need consumers to read from it reliably.

Sounds simple, but consider these scenarios:

### Scenario 1: Slow Consumer

Analytics service processes each event in 100ms. Meanwhile:
- Events arrive at 1000/sec
- Analytics can process 10/sec
- **The consumer falls behind.**

What happens? Does the consumer:
- Block producers? (No, that would break everything)
- Drop events? (No, we need every event)
- Keep falling further behind forever? (Ideally not)

### Scenario 2: Consumer Crash

A consumer is processing events. Mid-batch, it crashes. When it restarts:
- Which events were successfully processed?
- Which need to be retried?
- How does it know where to resume?

### Scenario 3: Multiple Services, Same Data

You have:
- Analytics (wants all events)
- Fraud detection (wants all events)
- Recommendations (wants all events)

But each has different:
- Processing speed
- Processing logic
- Availability (analytics might be down while fraud is up)

**They should not interfere with each other.**

### Scenario 4: Backpressure

A consumer is overwhelmed. It needs to:
- Slow down (not fetch more data)
- Not block producers
- Not lose data

How?

These scenarios reveal the complexity of consumption. Let's solve them step by step.

## Core Concept: Offset Management

The foundational idea:

> **Each consumer tracks its position (offset) in each partition independently.**

```
Partition 0: [e0] [e1] [e2] [e3] [e4] [e5] ...
                         ↑
                    Consumer's offset: 2
                    (has read 0,1,2; next will read 3)
```

**This is stored separately from the log itself.**

### Where to Store Offsets?

**Option 1: Consumer's local storage**
- Simple, fast
- But if consumer crashes and is replaced, new instance doesn't know where to start

**Option 2: External database**
- Durable
- But introduces external dependency, network calls

**Option 3: Kafka itself**
- Kafka has a special internal topic: `__consumer_offsets`
- Consumers commit their offsets to this topic
- **This is what Kafka does**

**Why this is clever:**
- Offset commits are just writes to a Kafka topic
- Use the same infrastructure
- Benefit from Kafka's durability and replication

## Commit Strategies

When should a consumer commit its offset?

### Strategy 1: Auto-commit (Naive)

```javascript
class Consumer {
    constructor(config) {
        this.autoCommit = true;
        this.autoCommitInterval = 5000; // Every 5 seconds
    }

    poll() {
        const messages = this.fetch();

        // Process messages...
        messages.forEach(msg => this.process(msg));

        // Auto-commit happens in background
        return messages;
    }
}
```

**Problem:** What if consumer crashes between fetching and processing?

```
1. Fetch offsets 100-199
2. [Auto-commit writes offset 200]
3. Processing...
4. [Crash at offset 150]
5. Restart: Reads from offset 200
6. Offsets 150-199 are LOST!
```

**This is at-most-once delivery.** (May lose data)

### Strategy 2: Manual Commit After Processing

```javascript
class Consumer {
    constructor(config) {
        this.autoCommit = false;
    }

    async poll() {
        const messages = this.fetch();

        // Process all messages
        for (const msg of messages) {
            await this.process(msg);
        }

        // Only commit after successful processing
        await this.commitSync();
    }
}
```

**Better, but what if:**

```
1. Fetch offsets 100-199
2. Process all messages
3. [Crash before commit]
4. Restart: Reads from offset 100 again
5. Offsets 100-199 are REPROCESSED!
```

**This is at-least-once delivery.** (May process duplicates)

### Strategy 3: Idempotent Processing

Accept that duplicates can happen, make processing idempotent:

```javascript
async function processOrder(event) {
    // Check if already processed
    const exists = await db.orders.findOne({ orderId: event.orderId });

    if (exists) {
        console.log('Already processed, skipping');
        return; // Idempotent!
    }

    // Process the order
    await db.orders.create(event);
}
```

**This is the most common pattern in production.**

### Strategy 4: Transactional Processing

Store offset in the same transaction as processing:

```javascript
async function processOrder(event, offset) {
    const transaction = await db.beginTransaction();

    try {
        // Store the result
        await transaction.orders.create(event);

        // Store the offset
        await transaction.offsets.upsert({
            partition: event.partition,
            offset: offset
        });

        await transaction.commit();
    } catch (error) {
        await transaction.rollback();
    }
}
```

**This gives exactly-once semantics**, but:
- Requires transactional support in destination system
- More complex
- Slower

Kafka supports this with its transactions API, but it's advanced usage.

## Consumer Groups Revisited

Remember: consumer groups enable load balancing.

```
Topic: orders (3 partitions)

Consumer Group "analytics":
  Consumer A: Partition 0, 1
  Consumer B: Partition 2

Consumer Group "fraud-detection":
  Consumer C: Partition 0
  Consumer D: Partition 1
  Consumer E: Partition 2
```

**Key points:**

1. **Each group tracks its own offsets** - Analytics at offset 1000, fraud at offset 500 (slower)
2. **Groups are independent** - Fraud being slow doesn't affect analytics
3. **Within a group, partitions are the unit of parallelism** - Can't have more parallelism than partitions

## Rebalancing Deep Dive

When a consumer joins/leaves, rebalancing occurs:

### The Rebalance Protocol

1. **Consumer sends heartbeat** - "I'm alive!"
2. **Coordinator detects change** - New consumer joined, or heartbeat missed
3. **Coordinator triggers rebalance** - Broadcasts to all consumers in group
4. **Consumers stop processing** - Commit current offsets
5. **Coordinator assigns partitions** - Based on strategy (range, round-robin, sticky)
6. **Consumers resume** - From committed offsets on new partitions

### The Stop-the-World Problem

During rebalance:
- **No messages are processed**
- Can take 5-30 seconds for large groups
- Happens whenever:
  - Consumer joins
  - Consumer leaves
  - Consumer crashes (detected after heartbeat timeout)

**This is a major pain point.**

### Optimizations (Modern Kafka)

**Incremental Cooperative Rebalancing (KIP-429):**
- Instead of stopping all consumers, only revoke partitions being reassigned
- Other partitions keep processing
- Reduces rebalance impact

**Static Membership (KIP-345):**
- Assign each consumer a static ID
- If consumer restarts with same ID, no rebalance
- Useful for rolling restarts

These are why using **latest Kafka** matters.

## Backpressure and Flow Control

What if a consumer is overwhelmed?

### Poll-Based Model

Kafka uses a **pull model** (consumer pulls data), not push:

```javascript
while (true) {
    const messages = consumer.poll(timeout = 100);

    // Process at consumer's pace
    for (const msg of messages) {
        await slowProcessing(msg);
    }

    // Consumer controls when to fetch more
}
```

**Benefits:**
- Consumer controls rate
- No buffer overflow in consumer
- Backpressure is natural (just poll less frequently)

**Contrast with push model** (like some message queues):
- Broker pushes data to consumer
- Consumer must handle or drop
- Backpressure requires complex signaling

### Max Poll Records

```javascript
const consumer = new KafkaConsumer({
    maxPollRecords: 500, // Limit batch size
    maxPollInterval: 300000, // 5 minutes max between polls
});
```

- `maxPollRecords`: Controls memory usage (smaller batches)
- `maxPollInterval`: If consumer takes too long to poll again, it's considered dead

## Lag Monitoring

How far behind is a consumer?

```
Partition high-water mark (latest offset): 10,000
Consumer's committed offset: 9,500
Lag: 10,000 - 9,500 = 500 messages
```

**Lag is the most important consumer metric.**

High lag indicates:
- Consumer too slow
- Not enough consumers in group
- Need more partitions for parallelism
- Processing logic has bottleneck

## Multiple Consumer Groups Pattern

Different groups, different use cases:

```
Topic: orders

Group "realtime-analytics" (low latency, can drop):
  - Reads latest
  - If falls behind, skips to recent (reset to latest)
  - Visualization dashboards

Group "financial-audit" (complete accuracy):
  - Reads from beginning
  - Never skips
  - Processes every single event
  - Regulatory compliance

Group "fraud-detection" (medium latency):
  - Reads recent
  - Can tolerate some lag
  - Alerts on suspicious patterns
```

**Each group has different:**
- Offset reset policy (earliest, latest, specific offset)
- Processing guarantees (at-least-once, exactly-once)
- Lag tolerance

## Seeking: Time-Travel in the Log

Consumers can manually seek to any offset:

```javascript
// Seek to beginning
consumer.seekToBeginning();

// Seek to end (skip all existing data)
consumer.seekToEnd();

// Seek to specific offset
consumer.seek(partition, offset);

// Seek to timestamp
consumer.seekToTimestamp(partition, timestamp);
```

**Use cases:**
- Reprocessing after bug fix
- Catching up after downtime
- Testing with historical data

This is the "replay" capability databases don't give you.

## The At-Least-Once vs Exactly-Once Trade-off

**At-least-once** (common, simpler):
- Commit after processing
- Duplicates possible on failures
- Make processing idempotent

**Exactly-once** (complex, slower):
- Use Kafka transactions
- Atomic commit of offsets and results
- Requires transactional producer and consumer
- Only works end-to-end with Kafka (producer → Kafka → consumer)

**Most teams use at-least-once + idempotency** because:
- Simpler to implement
- Better performance
- More flexible (works with any destination)

---

## Key Takeaways

- **Offsets are the consumer's bookmark** - Track position per partition
- **Commit strategy determines delivery guarantees** - Auto (at-most-once), manual (at-least-once), transactional (exactly-once)
- **Consumer groups enable scalability** - Multiple consumers, load balanced
- **Rebalancing is necessary but costly** - Modern Kafka optimizes this
- **Pull model enables natural backpressure** - Consumer controls rate
- **Lag is the key metric** - How far behind is the consumer?
- **Replay is powerful** - Seek to any offset or timestamp

## Design Patterns

**For high-throughput, can tolerate duplicates:**
- At-least-once delivery
- Idempotent processing
- Large batches (high maxPollRecords)

**For critical accuracy:**
- Transactional processing
- Exactly-once semantics
- Smaller batches for faster recovery

**For real-time dashboards:**
- Read from latest
- Can reset to latest on lag
- Low latency over completeness

**For data warehousing:**
- Read from earliest
- Never skip
- Can tolerate lag (batch processing)

---

**[← Previous: The Distribution Challenge](./05-distribution-challenge.md) | [Back to Contents](./README.md) | [Next: Inside the Broker →](./07-inside-the-broker.md)**
