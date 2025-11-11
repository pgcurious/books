# Chapter 5: The Distribution Challenge

> **"A single machine is a single point of failure. Distribution is both the solution and the problem."**

## The Single-Machine Limit

Your append-only log is working great. You're handling 100,000 events per second on a single machine. But your business keeps growing, and you're hitting limits:

### Physical Limits

**Disk space:**
- 100K events/sec × 1KB per event = 100 MB/sec
- 100 MB/sec × 86,400 seconds/day = 8.6 TB/day

Even with a 7-day retention, that's 60TB. One machine can't hold this forever.

**Disk I/O:**
- Sequential writes are fast, but even they have limits
- ~1000 MB/sec on HDD, ~3000 MB/sec on SSD
- You're approaching these limits

**Network:**
- Multiple consumers reading
- If 10 consumers each read at 100 MB/sec = 1 GB/sec network bandwidth
- 1-10 Gbps NIC limits are real

### Operational Limits

**Single point of failure:**
- If the machine dies, your entire event stream is down
- No writes, no reads, total outage

**Deployment and maintenance:**
- Can't upgrade without downtime
- Can't perform maintenance without stopping the world

**You need to distribute the load across multiple machines.**

## The First Idea: Partition by Topic

Your first instinct: "We have different types of events. Let's put them on different machines!"

```
Machine A: orders.log
Machine B: inventory.log
Machine C: users.log
```

**This helps**, but:
- What if orders grow way faster than other topics?
- Machine A becomes a bottleneck while B and C are idle
- You can't scale a single topic beyond one machine

**This doesn't solve the fundamental problem.**

## The Key Insight: Partitioning

What if we split a SINGLE log across multiple machines?

```
Orders Topic:
  Partition 0 → Machine A
  Partition 1 → Machine B
  Partition 2 → Machine C
```

Now when a producer writes an order event, it goes to one of the partitions. Each partition is an independent log.

### The Partitioning Decision

How do you decide which partition an event goes to?

**Option 1: Round-robin**
```
Event 1 → Partition 0
Event 2 → Partition 1
Event 3 → Partition 2
Event 4 → Partition 0
...
```

**Pros:** Balanced distribution
**Cons:** No ordering guarantees

Orders for the same user could be split across partitions and processed out of order.

**Option 2: Hash-based**
```javascript
function selectPartition(event, numPartitions) {
    const key = event.userId; // or orderId, or any key
    const hash = hashFunction(key);
    return hash % numPartitions;
}
```

```
Event {userId: 123} → hash(123) % 3 = 0 → Partition 0
Event {userId: 456} → hash(456) % 3 = 1 → Partition 1
Event {userId: 123} → hash(123) % 3 = 0 → Partition 0
```

**Pros:** All events with the same key go to the same partition (ordering guaranteed per key)
**Cons:** Uneven distribution if keys are skewed

**This is what Kafka does by default.**

### The Ordering Guarantee

**Critical insight:**

> Kafka guarantees order WITHIN a partition, not across partitions.

```
Partition 0: [e1] → [e3] → [e5]  (ordered)
Partition 1: [e2] → [e4] → [e6]  (ordered)
Partition 2: [e7] → [e8] → [e9]  (ordered)

Across all partitions: No global order guarantee
```

**Why this works:**

For most use cases, you care about order within a specific entity:
- All orders for user 123 are in order
- All inventory updates for product 456 are in order

You rarely need GLOBAL order across all events.

## Producing to Partitions

Your producer now needs to know about partitions:

```javascript
class KafkaProducer {
    constructor(topic, brokers) {
        this.topic = topic;
        this.brokers = brokers; // ['broker1:9092', 'broker2:9092', 'broker3:9092']
        this.metadata = this.fetchMetadata(); // Which partitions exist, which broker leads each
    }

    async send(event, key) {
        // Determine partition
        const partition = key ?
            hashCode(key) % this.metadata.numPartitions :
            this.roundRobinPartition();

        // Find the broker that leads this partition
        const broker = this.metadata.partitionLeader[partition];

        // Send to that broker
        await this.sendToBroker(broker, partition, event);
    }
}
```

**The producer needs to know:**
1. How many partitions exist
2. Which broker (machine) leads each partition

This is **metadata** that needs to be discovered and kept up-to-date.

## Consuming from Partitions

Consumers also need to understand partitions:

### Single Consumer

```javascript
class KafkaConsumer {
    constructor(topic, brokers) {
        this.topic = topic;
        this.brokers = brokers;
        this.offsets = {}; // Track offset per partition
    }

    async poll() {
        const messages = [];

        // Fetch from all partitions
        for (const partition of this.getPartitions()) {
            const offset = this.offsets[partition] || 0;
            const batch = await this.fetchFromPartition(partition, offset);

            messages.push(...batch);
            this.offsets[partition] = batch.lastOffset + 1;
        }

        return messages;
    }
}
```

A single consumer reads from ALL partitions of a topic.

### Multiple Consumers (Consumer Group)

But what if processing is slow? You want to parallelize consumption.

**Consumer Group pattern:**

```
Orders Topic (3 partitions):
  Partition 0 → Consumer A
  Partition 1 → Consumer B
  Partition 2 → Consumer C
```

Each consumer in a group reads from a subset of partitions. **Partitions are the unit of parallelism.**

**Important rules:**
1. A partition is assigned to only ONE consumer in a group
2. A consumer can read multiple partitions
3. If you have more consumers than partitions, some consumers sit idle

```
3 partitions, 2 consumers:
  Consumer A reads: Partition 0, Partition 1
  Consumer B reads: Partition 2

3 partitions, 5 consumers:
  Consumer A reads: Partition 0
  Consumer B reads: Partition 1
  Consumer C reads: Partition 2
  Consumer D: idle
  Consumer E: idle
```

**Max parallelism = number of partitions.**

This is why choosing the right number of partitions is important.

## Rebalancing

What happens when a consumer joins or leaves the group?

**Scenario: 3 partitions, 2 consumers**

```
Initial state:
  Consumer A: Partition 0, 1
  Consumer B: Partition 2

Consumer C joins:
  [Rebalance triggered]
  Consumer A: Partition 0
  Consumer B: Partition 1
  Consumer C: Partition 2

Consumer A crashes:
  [Rebalance triggered]
  Consumer B: Partition 0, 1
  Consumer C: Partition 2
```

**Rebalancing ensures:**
- Even distribution of partitions
- Every partition is assigned
- No partition is assigned to multiple consumers

**But rebalancing has costs:**
- Consumers stop processing during rebalance
- Offsets must be committed before reassignment
- Can take seconds for large consumer groups

This is a known pain point in Kafka. We'll discuss optimizations later.

## The Broker Model

Each machine running Kafka is called a **broker**. Brokers:

1. **Store partition data** - Each broker holds some partitions
2. **Serve reads and writes** - Producers write, consumers read
3. **Coordinate** - Share metadata about who leads which partition

```
Cluster:
  Broker 1 (192.168.1.101):
    - orders-0
    - inventory-1
    - users-2

  Broker 2 (192.168.1.102):
    - orders-1
    - inventory-2
    - users-0

  Broker 3 (192.168.1.103):
    - orders-2
    - inventory-0
    - users-1
```

Partitions are distributed across brokers for:
- **Load balancing** - Writes and reads spread out
- **Fault tolerance** - If one broker dies, others still serve partitions

## The Partition Leadership Model

Each partition has:
- **One leader** - Handles all reads and writes
- **Multiple followers** - Replicate data from the leader (next chapter)

```
orders-0:
  Leader: Broker 1
  Followers: Broker 2, Broker 3

orders-1:
  Leader: Broker 2
  Followers: Broker 1, Broker 3

orders-2:
  Leader: Broker 3
  Followers: Broker 1, Broker 2
```

**Why leaders?**
- Simplifies consistency (one source of truth)
- Followers just replicate (no complex consensus on every write)

## The Scale Benefits

Now let's revisit our scaling problem:

**Before (single machine):**
- Disk: 60TB capacity needed
- Write throughput: 100K events/sec
- Read throughput: Limited by single machine's network

**After (3 brokers, 3 partitions):**
- Disk: 20TB per broker (distributed)
- Write throughput: ~300K events/sec (3× parallelism)
- Read throughput: 3× better (consumers read from different brokers)

**And you can keep adding partitions/brokers as needed.**

## The Trade-offs

**More partitions = More parallelism**, but:

1. **More files on disk** - Each partition = log files + index files
2. **More memory** - Each partition has in-memory buffers
3. **Longer rebalances** - More partitions to redistribute
4. **More metadata** - Tracking becomes complex

**Typical guidance:**
- Start with partitions = expected max consumers
- 10-100 partitions per broker is common
- Thousands of partitions on one broker gets expensive

## The Question of Durability

We've distributed the load, but we've introduced a NEW problem:

> **"If Broker 1 dies, we lose all the data in its partitions!"**

This is unacceptable. We need **replication**.

---

## Key Takeaways

- **Partitioning enables horizontal scaling** - Split a topic across machines
- **Partitions are the unit of parallelism** - More partitions = more concurrent consumers
- **Ordering is per-partition only** - Not global across a topic
- **Hash-based partitioning** - Routes events with same key to same partition
- **Consumer groups enable load balancing** - Multiple consumers, each reads from subset of partitions
- **Rebalancing redistributes partitions** - When consumers join/leave
- **Leaders handle all I/O** - Followers replicate (we'll cover this next)
- **More partitions ≠ always better** - Trade-offs exist

## The Mental Model

**Single Log:**
```
[e1] → [e2] → [e3] → [e4] → [e5] → ...
```

**Partitioned Log:**
```
Partition 0: [e1] → [e3] → [e5] → ...
Partition 1: [e2] → [e4] → [e6] → ...
Partition 2: [e7] → [e8] → [e9] → ...
```

Each partition is an independent, ordered log. Together, they form a scalable topic.

---

**[← Previous: The Append-Only Revelation](./04-append-only-revelation.md) | [Back to Contents](./README.md) | [Next: The Consumer Conundrum →](./06-consumer-conundrum.md)**
