# Chapter 13: The Trade-offs

> **"There are no solutions, only trade-offs." - Thomas Sowell**

## The Fundamental Truth

Kafka makes specific trade-offs to achieve its goals. Understanding these trade-offs helps you:
1. **Configure it correctly** for your use case
2. **Avoid surprises** in production
3. **Make informed decisions** about when to use it

Let's be explicit about what you gain and what you give up.

## Trade-off 1: Throughput vs Latency

### The Choice

**High Throughput Configuration:**
```javascript
{
    batchSize: 1000000,        // 1 MB batches
    lingerMs: 100,             // Wait 100ms to fill batch
    compression: 'zstd',       // Compress aggressively
    acks: 1,                   // Leader only
}
```

**Result:**
- Throughput: 100,000+ messages/sec
- Latency: 50-200ms (p99)

**Low Latency Configuration:**
```javascript
{
    batchSize: 1024,           // Small batches
    lingerMs: 0,               // Send immediately
    compression: 'none',       // No compression delay
    acks: 1,                   // Leader only
}
```

**Result:**
- Throughput: 10,000 messages/sec
- Latency: 2-10ms (p99)

### What This Means

You **cannot optimize for both simultaneously**.

**Why?**
- Batching increases throughput (fewer network calls) but adds latency (wait time)
- Compression increases throughput (less data) but adds latency (CPU time)

**Your decision:**
- **Real-time dashboard** → Optimize for latency
- **Data pipeline** → Optimize for throughput
- **Hybrid** → Find middle ground (medium batches, fast compression like lz4)

## Trade-off 2: Durability vs Performance

### The Choice

**Maximum Durability:**
```javascript
{
    acks: 'all',                        // Wait for all replicas
    minInsyncReplicas: 2,               // Require 2 replicas minimum
    replicationFactor: 3,               // 3 copies
    uncleanLeaderElection: false,       // Never allow data loss
}
```

**Result:**
- Data loss: Nearly impossible (requires 2+ brokers to fail simultaneously)
- Latency: 5-15ms (wait for replication)
- Availability: Lower (partition unavailable if only 1 replica up)

**Maximum Performance:**
```javascript
{
    acks: 1,                            // Leader only
    minInsyncReplicas: 1,               // Leader counts
    replicationFactor: 1,               // No replication
    uncleanLeaderElection: true,        // Allow any replica as leader
}
```

**Result:**
- Data loss: Possible (if leader dies before replication)
- Latency: 1-3ms (no replication wait)
- Availability: Higher (always available)

### What This Means

**You must choose what you value more:**

| Use Case | Configuration | Why |
|----------|--------------|-----|
| Financial transactions | Max durability | Cannot lose money |
| Clickstream analytics | Balanced | Losing a few clicks acceptable |
| Real-time game events | Max performance | Speed > completeness |
| Audit logs | Max durability | Regulatory requirement |

**No configuration is "better"—it depends on your requirements.**

## Trade-off 3: Ordering vs Parallelism

### The Choice

**Global Ordering:**
```javascript
// Single partition
await admin.createTopic({
    topic: 'events',
    numPartitions: 1,  // Global order maintained
});
```

**Result:**
- Ordering: Perfect (global order across all messages)
- Parallelism: None (only 1 consumer can read)
- Throughput: Limited to single partition (~50-100K msgs/sec)

**High Parallelism:**
```javascript
// Many partitions
await admin.createTopic({
    topic: 'events',
    numPartitions: 100,  // High parallelism
});
```

**Result:**
- Ordering: Per partition only (no global order)
- Parallelism: Up to 100 concurrent consumers
- Throughput: Very high (millions of msgs/sec)

### What This Means

**Partitions are the unit of parallelism AND ordering.**

**More partitions:**
- ✓ More consumers (higher throughput)
- ✓ Better load distribution
- ✗ No global ordering
- ✗ More operational complexity

**Fewer partitions:**
- ✓ Simpler to manage
- ✓ Better ordering guarantees
- ✗ Less parallelism
- ✗ Lower max throughput

**Your decision depends on:**
- Do you need global order? (Stock trades: YES, Web clicks: NO)
- How much throughput? (Millions/sec → need many partitions)

## Trade-off 4: Pull vs Push Model

### The Choice

**Kafka uses Pull** (consumer pulls from broker):

```javascript
while (true) {
    const messages = await consumer.poll();  // Consumer controls
    await processMessages(messages);
}
```

**Advantages:**
- Consumer controls rate (natural backpressure)
- Consumer can batch (fetch many messages at once)
- Broker is simpler (no tracking consumer state)

**Disadvantages:**
- Consumer must poll actively (uses CPU even when idle)
- Slightly higher latency (poll interval)
- Consumer must implement retry logic

### Alternative: Push Model (e.g., RabbitMQ)

Broker pushes messages to consumers.

**Advantages:**
- Lower latency (immediate delivery)
- Simpler consumer (just receive and process)

**Disadvantages:**
- Broker must track consumer state (complex)
- Risk of overwhelming consumer (needs backpressure protocol)
- Harder to implement batching

### What This Means

**Kafka chose pull because:**
1. Fits the high-throughput, batch-oriented design
2. Makes brokers simpler (less state)
3. Gives consumers control (backpressure is natural)

**Trade-off:** Slightly more work for consumer implementation, but better scalability.

## Trade-off 5: Partition Count

### The Choice

**Few Partitions (e.g., 3):**

**Advantages:**
- Simple to manage
- Faster rebalancing
- Less memory/file descriptors
- Easier to reason about

**Disadvantages:**
- Limited parallelism (max 3 concurrent consumers)
- Lower max throughput
- Uneven load if data is skewed

**Many Partitions (e.g., 100):**

**Advantages:**
- High parallelism (up to 100 consumers)
- Better load distribution
- Higher max throughput

**Disadvantages:**
- Slower rebalancing (more to redistribute)
- More memory (buffers per partition)
- More file handles (log files per partition)
- More metadata to track

### What This Means

**Partition count is hard to change later** (requires topic recreation or complex migrations).

**Guidelines:**
- Start with: `partitions = expected_max_consumers`
- Typical: 10-50 partitions per topic
- High-throughput: 100+ partitions
- Low-throughput: 1-10 partitions

**Over-partitioning is safer than under-partitioning** (can always use fewer consumers than partitions).

## Trade-off 6: Retention Duration

### The Choice

**Long Retention (e.g., 30 days):**

**Advantages:**
- Can replay far back in history
- Build new consumers from historical data
- Audit trail
- Disaster recovery

**Disadvantages:**
- High disk usage (30 days × daily volume)
- More segments to manage
- Slower seek operations (more files to scan)

**Short Retention (e.g., 1 day):**

**Advantages:**
- Low disk usage
- Fewer files (faster operations)
- Lower costs

**Disadvantages:**
- Limited replay capability
- New consumers must start from recent data
- No historical audit trail

### What This Means

**Retention affects:**
1. Disk requirements: `retention_days × messages_per_day × message_size`
2. Operational costs (storage)
3. Flexibility (can you replay last week's data?)

**Decision factors:**
- **Regulatory/compliance:** May require 7+ years
- **Data pipeline:** 7-30 days typical
- **Real-time only:** 1-3 days sufficient

**Infinite retention?**
- Use log compaction for key-based topics
- Or archive to S3/HDFS and keep recent in Kafka

## Trade-off 7: Consumer Offset Commit Frequency

### The Choice

**Frequent Commits (per message):**

```javascript
for (const msg of messages) {
    await process(msg);
    await consumer.commitSync();  // Every message
}
```

**Advantages:**
- Minimal duplicates on failure (only last message)
- Precise tracking

**Disadvantages:**
- Very slow (network call per message)
- High load on `__consumer_offsets` topic

**Infrequent Commits (per batch):**

```javascript
for (const msg of messages) {
    await process(msg);
}
await consumer.commitSync();  // Once per batch
```

**Advantages:**
- Fast (one commit per batch)
- Lower load on brokers

**Disadvantages:**
- More duplicates on failure (entire batch)
- Less precise tracking

### What This Means

**You must balance:**
- **Performance** (fewer commits)
- **Duplicate tolerance** (more commits)

**Typical approach:**
- Commit per batch (100-1000 messages)
- Make processing idempotent (handle duplicates)
- Use exactly-once semantics if critical

## Trade-off 8: Replication Factor

### The Choice

**Replication Factor = 1:**

**Advantages:**
- Fastest (no replication delay)
- Lowest disk usage (one copy)
- Lowest network usage

**Disadvantages:**
- **Data loss** if broker dies
- No fault tolerance

**Replication Factor = 3:**

**Advantages:**
- Fault tolerant (can lose 2 brokers)
- No data loss (with `acks=all`)
- High availability

**Disadvantages:**
- 3x disk usage
- 3x network traffic (replication)
- Higher latency (wait for replicas)

### What This Means

**Never use `replication.factor=1` in production** unless:
- Data is ephemeral (losing it is acceptable)
- You have backups elsewhere

**Standard: `replication.factor=3`**
- Tolerates 2 broker failures
- `min.insync.replicas=2` for safety

## Trade-off 9: Exactly-Once vs Simplicity

### The Choice

**At-Least-Once (simpler):**

```javascript
// Manual commit after processing
for (const msg of messages) {
    await process(msg);
}
await consumer.commitSync();

// Make processing idempotent
async function process(msg) {
    const exists = await db.findOne({ id: msg.id });
    if (exists) return;  // Already processed

    await db.insert(msg);
}
```

**Advantages:**
- Simple to understand
- Works with any destination (database, API, etc.)
- Good performance

**Disadvantages:**
- Must handle duplicates (idempotency required)
- Possible reprocessing on failures

**Exactly-Once (complex):**

```javascript
// Use transactions
await producer.beginTransaction();
await producer.send(...);
await producer.sendOffsetsToTransaction(offsets, groupId);
await producer.commitTransaction();
```

**Advantages:**
- No duplicates
- Precise semantics

**Disadvantages:**
- Complex to implement
- Only Kafka-to-Kafka
- Higher latency
- More operational complexity

### What This Means

**Most teams use at-least-once + idempotency** because:
- Simpler
- Faster
- More flexible (works with external systems)

**Use exactly-once only when:**
- Cannot tolerate duplicates (financial transactions)
- Kafka-to-Kafka workflow (stream processing)
- Willing to accept complexity

## The Configuration Matrix

Here's how to configure for common scenarios:

| Goal | Config | Trade-off |
|------|--------|-----------|
| **Max Throughput** | Large batches, compression, `acks=1`, many partitions | Higher latency, less durability |
| **Min Latency** | Small batches, no compression, `acks=1`, few partitions | Lower throughput |
| **Max Durability** | `acks=all`, `min.insync.replicas=2`, `replication.factor=3` | Higher latency, lower throughput |
| **Exactly-Once** | Transactions, idempotence | Complexity, higher latency |
| **High Availability** | Many brokers, `replication.factor=3`, `unclean.leader.election=false` | More resources, slightly higher latency |
| **Low Cost** | Fewer brokers, short retention, compression | Less durability, less fault tolerance |

## The Honest Truth

**You cannot have:**
- Infinite throughput
- Zero latency
- Perfect durability
- Zero cost
- Infinite retention
- Global ordering with high parallelism

**You must choose.**

The art of using Kafka is understanding these trade-offs and configuring appropriately for your specific needs.

---

## Key Takeaways

- **Throughput vs latency** - Batching helps one, hurts the other
- **Durability vs performance** - Replication adds safety but latency
- **Ordering vs parallelism** - Partitions give you one or the other
- **Pull model** - Consumer control at cost of complexity
- **Partition count** - Hard to change, choose wisely
- **Retention** - Flexibility vs disk space
- **Commit frequency** - Performance vs duplicates
- **Replication factor** - Safety vs resources
- **Exactly-once** - Precision vs complexity

## The Decision Framework

When configuring Kafka, ask:

1. **What's my throughput requirement?** (msgs/sec)
2. **What's my latency tolerance?** (ms)
3. **Can I tolerate data loss?** (durability)
4. **Can I tolerate duplicates?** (exactly-once)
5. **Do I need global ordering?** (partitions)
6. **How far back must I replay?** (retention)
7. **What's my budget?** (brokers, disk)

Answer these, and your configuration becomes clear.

**There is no "best" Kafka configuration—only the right one for your use case.**

---

**[← Previous: When Kafka Struggles](./12-when-kafka-struggles.md) | [Back to Contents](./README.md) | [Next: The Bigger Picture →](./14-bigger-picture.md)**
