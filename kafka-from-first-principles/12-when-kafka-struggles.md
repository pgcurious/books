# Chapter 12: When Kafka Struggles

> **"Knowing when NOT to use a tool is as important as knowing when to use it."**

## The Wrong Use Cases

Kafka is powerful, but it's not a universal solution. Let's be honest about where Kafka struggles or is simply the wrong choice.

## 1. Request-Response Patterns

### The Anti-Pattern

```
User requests: "Get my order status"
                ↓
Application → Kafka (request message)
                ↓
Order Service consumes, processes
                ↓
Order Service → Kafka (response message)
                ↓
Application consumes response
                ↓
Return to user (after 50-500ms)
```

### Why This Is Bad

1. **High latency** - Multiple network hops, polling for response
2. **Complexity** - Correlation IDs, timeout handling, response topics
3. **Waste** - Response messages stored unnecessarily

**Typical latency:**
- Kafka: 50-200ms
- REST API: 5-20ms

### The Right Tool

**Use HTTP/REST/gRPC** for request-response:

```
User requests: "Get my order status"
                ↓
Application → HTTP GET /orders/123 → Order Service
                ↓
Order Service queries database
                ↓
Response (5-20ms)
```

**When to use what:**

| Pattern | Tool |
|---------|------|
| Request-response | HTTP, gRPC |
| Fire-and-forget | Kafka |
| Query data | Database |
| Stream events | Kafka |

## 2. Small-Scale / Low-Throughput Systems

### The Scenario

You have:
- 3 microservices
- 100 events per day
- Small team (2-3 developers)

**Is Kafka overkill? Yes.**

### The Operational Cost

Running Kafka requires:
- **Minimum 3 brokers** (for production with replication)
- **Monitoring** (consumer lag, broker health, disk usage)
- **Expertise** (understanding rebalancing, partition assignment, etc.)
- **Infrastructure** (VMs/containers, load balancers)

**For 100 events/day, this is massive overkill.**

### Better Alternatives

**Simple queue (RabbitMQ, SQS, Redis):**
```
Order Service → Redis List → Email Service
```

**Or even direct HTTP:**
```
Order Service → HTTP POST → Email Service (with retry)
```

**Rule of thumb:**
- < 1,000 messages/day → Consider simpler tools
- < 10,000 messages/day → Maybe Kafka, maybe not
- \> 100,000 messages/day → Kafka makes sense

**Exception:** Even at low volume, if you need replay/audit, Kafka might still be worth it.

## 3. Low-Latency Queuing (< 1ms)

### The Scenario

You need **sub-millisecond latency** consistently.

Examples:
- High-frequency trading
- Real-time bidding (ad tech)
- Gaming leaderboards

### Why Kafka Struggles

**Kafka's typical latency:**
- p50: 2-5ms
- p99: 10-20ms
- p99.9: 50-100ms

**Sources of latency:**
1. Batching (producer waits for batch)
2. Replication (wait for ISR)
3. Network hops (producer → broker → consumer)
4. Disk writes (even with page cache)

### Better Alternatives

**In-memory systems:**
- **Redis Streams** - In-memory, ~1ms latency
- **Apache Pulsar** - Tiered storage, lower latency
- **NATS** - Ultra-low latency message bus

**Or even:**
- **Direct TCP sockets** - No middleware, <0.1ms

**Trade-off:** These sacrifice durability or persistence for speed.

**When to use what:**
- Need < 1ms consistently → Redis, NATS
- Need 1-10ms + durability → Kafka
- Need < 0.1ms, can lose data → Direct sockets

## 4. Transactional Workflows (ACID)

### The Scenario

You need an operation to update multiple systems atomically:

```
Transfer money:
  1. Deduct from Account A
  2. Add to Account B
  3. Record transaction
  4. Update balance history

All must succeed or all must fail (atomically).
```

### Why Kafka Struggles

Kafka transactions are **Kafka-to-Kafka only**:

```
Can do atomically:
  - Read from Kafka topic A
  - Write to Kafka topic B
  - Commit consumer offset

Cannot do atomically:
  - Read from Kafka
  - Write to PostgreSQL
  - Write to Kafka
```

**Kafka is not a database.** It doesn't support arbitrary transactional operations across external systems.

### The Right Tool

**Database with ACID transactions:**

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
  INSERT INTO transactions (from, to, amount) VALUES ('A', 'B', 100);
COMMIT;
```

**Or distributed transaction coordinator:**
- **Two-phase commit (2PC)** - Coordinator + participants
- **Saga pattern** - Choreographed compensating transactions

**Kafka's role:** After the transaction, publish an event:

```
Database transaction completes
  ↓
Publish TRANSFER_COMPLETED to Kafka
  ↓
Other systems react (notifications, analytics)
```

**Don't try to replace databases with Kafka.**

## 5. Large Message Sizes (> 1 MB)

### The Scenario

You need to send:
- Video files (10-100 MB)
- Large documents (5-50 MB)
- Big binary blobs

### Why Kafka Struggles

**Kafka is optimized for many small messages**, not few large messages:

1. **Memory pressure** - Large messages in buffers
2. **Rebalancing delays** - Large messages slow down rebalancing
3. **Consumer lag** - One slow consumer blocks partition
4. **Network saturation** - Large messages hog bandwidth

**Performance degrades significantly > 1 MB per message.**

### Better Alternatives

**Object storage + Kafka reference:**

```
Producer:
  1. Upload video to S3
  2. Send S3 URL to Kafka (small message)

Consumer:
  1. Read S3 URL from Kafka
  2. Download video from S3
```

**Benefits:**
- Kafka messages stay small (fast)
- S3 handles large files (that's what it's for)
- Multiple consumers don't duplicate large downloads

**Other options:**
- **GridFS** (MongoDB) for large files
- **Blob storage** (Azure, GCS)

**Rule:** If message > 100 KB, consider external storage + reference pattern.

## 6. Strong Consistency Across Partitions

### The Scenario

You need **global ordering** or **atomic updates across multiple partitions**.

Example:
```
Stock trading: Order events must be globally ordered
  Event 1: BUY AAPL (partition 0)
  Event 2: SELL AAPL (partition 1)

If consumed out of order → incorrect stock calculation
```

### Why Kafka Struggles

**Kafka guarantees order ONLY within a partition**, not across partitions.

With multiple partitions:
- Partition 0: [e1] → [e3] → [e5]
- Partition 1: [e2] → [e4] → [e6]

**No guarantee that e1 is processed before e2 globally.**

### Workarounds (with trade-offs)

**1. Single partition:**
```
Use 1 partition → Global ordering guaranteed
```

**Trade-off:** No parallelism (only 1 consumer), limited throughput.

**2. Partition by entity:**
```
All AAPL trades → Partition 0
All MSFT trades → Partition 1
```

**Trade-off:** Ordering per stock, not globally.

### Better Alternatives

For **strict global ordering**:
- **Database with sequential ID** - Single source of truth
- **Single-leader queue** - RabbitMQ, Redis List
- **Raft-based log** - etcd, Consul

**Kafka is designed for partitioned, high-throughput use cases, not strict global ordering.**

## 7. Complex Queries and Aggregations

### The Scenario

You need:
- SQL queries (JOINs, GROUP BY, WHERE clauses)
- Ad-hoc analysis
- Historical aggregations

Example:
```sql
SELECT user_id, COUNT(*) as order_count
FROM orders
WHERE created_at > '2025-01-01'
GROUP BY user_id
HAVING COUNT(*) > 10;
```

### Why Kafka Struggles

Kafka is **not a query engine**:
- No SQL interface (directly)
- No indexes for fast lookups
- Must scan entire topic (slow)

**Kafka Streams** can do some aggregations, but:
- Real-time only (not historical ad-hoc queries)
- Limited query capabilities
- Requires code (not SQL)

### The Right Tool

**Databases:**

```
Kafka → Stream → Database (PostgreSQL, ClickHouse)
                 ↓
            Run SQL queries
```

**Or analytics platforms:**
- **Snowflake** - Data warehouse
- **BigQuery** - Analytics
- **Druid** - Real-time analytics

**Kafka's role:** Feed data into these systems, don't replace them.

## 8. Tiny Messages, Ultra-High Throughput (Millions/sec)

### The Scenario

You need:
- 10+ million messages/sec
- Each message is 10-50 bytes
- Ultra-low overhead

### Why Kafka Can Struggle

Even though Kafka is fast, each message has **overhead**:

```
Per message overhead:
  - 8 bytes: offset
  - 4 bytes: size
  - 4 bytes: CRC
  - 1 byte: magic
  - 1 byte: attributes
  - 8 bytes: timestamp
  = ~26 bytes overhead + message size
```

For a 10-byte message:
- Payload: 10 bytes
- Overhead: 26 bytes
- **72% overhead!**

### Better Alternatives

**In-memory streaming:**
- **Apache Pulsar** - Lower per-message overhead
- **Redis Streams** - Minimal overhead
- **Chronicle Queue** - Ultra-fast, local-only

**Or batch more aggressively:**
- Send arrays of messages (1000 messages in one Kafka message)
- Reduces per-message overhead

## 9. Simple Pub/Sub (No Persistence Needed)

### The Scenario

You need:
- Broadcast messages to multiple subscribers
- Don't need persistence (ephemeral)
- Don't need replay

Example: Live notifications (user is online, typing indicators)

### Why Kafka Is Overkill

Kafka's strengths (persistence, replay, durability) are **wasted** here:
- Messages written to disk (unnecessary)
- Replication (unnecessary)
- Consumer offset tracking (unnecessary)

### Better Alternatives

**Lightweight pub/sub:**
- **Redis Pub/Sub** - In-memory, no persistence
- **NATS** - Lightweight, fast
- **WebSockets** - Direct client-to-client

**Use case:**
```
User typing indicator:
  User A types → Redis Pub/Sub → User B sees indicator

No need to persist this!
```

**If you need persistence and replay, use Kafka. If not, don't.**

## 10. Geographically Distributed (Multi-Region) as Primary

### The Scenario

You need:
- Active-active multi-region setup
- Writes in multiple regions
- Strong consistency across regions

### Why Kafka Struggles

Kafka's replication is **within-cluster only**:
- Replication is fast (same datacenter, low latency)
- Cross-region replication adds latency (50-200ms)

**MirrorMaker 2** can replicate across regions, but:
- Eventual consistency (not immediate)
- Conflict resolution complex
- Adds latency

### Better Alternatives

**Globally distributed databases:**
- **CockroachDB** - Multi-region ACID
- **Google Spanner** - Global consistency
- **Cosmos DB** - Multi-master

**Or regional Kafka clusters:**
- Each region has its own Kafka cluster
- Replicate asynchronously between regions
- Accept eventual consistency

**Kafka is not designed for multi-region strong consistency.**

---

## Summary: When NOT to Use Kafka

| Scenario | Why Kafka is Wrong | Better Alternative |
|----------|-------------------|-------------------|
| Request-response | High latency, complexity | HTTP, gRPC |
| Low throughput | Operational overhead | RabbitMQ, SQS, Redis |
| Sub-ms latency | Batching, disk writes | Redis, NATS |
| ACID transactions | Kafka-to-Kafka only | Database |
| Large messages (>1 MB) | Memory, network issues | S3 + Kafka reference |
| Global ordering | Partition-level only | Single partition or database |
| Ad-hoc queries | Not a query engine | Database, data warehouse |
| Ephemeral pub/sub | Unnecessary persistence | Redis Pub/Sub, NATS |
| Multi-region active-active | Within-cluster replication | CockroachDB, Spanner |

---

## Key Takeaways

- **Kafka is not a database** - Don't use it for queries or ACID transactions
- **Kafka is not a universal queue** - Not always the right choice
- **Kafka has operational cost** - Only worth it at scale
- **Know your latency requirements** - Sub-ms? Not Kafka
- **Message size matters** - Keep messages small (<100 KB)
- **Ordering is per-partition** - Not global
- **Persistence is a feature, not always needed** - Don't over-engineer

## The Honest Assessment

Kafka is **incredibly powerful** for:
- High-throughput event streaming
- Multiple consumers
- Replay and durability

But it's **not the right tool** for:
- Request-response APIs
- Low-latency queuing (< 1ms)
- Transactional workflows
- Ad-hoc queries
- Small-scale systems

**Choose wisely.**

---

**[← Previous: When Kafka Shines](./11-when-kafka-shines.md) | [Back to Contents](./README.md) | [Next: The Trade-offs →](./13-the-tradeoffs.md)**
