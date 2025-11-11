# Chapter 8: The Replication Protocol

> **"A system that can't survive failure is a system waiting to fail."**

## The Durability Problem

You've built a fast, distributed log. But there's a critical weakness:

```
Broker 1: orders-0, orders-1
Broker 2: orders-2, users-0
Broker 3: inventory-0

[Broker 1 dies]

Result: orders-0 and orders-1 are GONE. Data loss!
```

**Unacceptable.**

You need **replication**: copies of data on multiple machines.

## The Basic Idea

For each partition, keep N copies (replicas):

```
orders-0:
  Replica 1: Broker 1 (leader)
  Replica 2: Broker 2 (follower)
  Replica 3: Broker 3 (follower)
```

**Replication factor = 3** (common in production)

Now if Broker 1 dies, Broker 2 or 3 can take over. No data loss.

## Leader-Based Replication

**Why not multi-master?**

Multi-master replication (all replicas accept writes) requires complex conflict resolution:

```
Client A writes to Replica 1: value = "foo"
Client B writes to Replica 2: value = "bar"

What's the correct value? How do you resolve conflicts?
```

For a log, this is especially problematic since **order matters**.

**Kafka's choice: Single leader per partition.**

```
orders-0:
  Leader: Broker 1 ← All writes go here
  Follower: Broker 2 ← Replicates from leader
  Follower: Broker 3 ← Replicates from leader
```

**Producers and consumers only talk to the leader.**

Followers are:
- For fault tolerance (take over if leader dies)
- NOT for load balancing reads (this is different from databases)

## The Replication Protocol

### Step 1: Producer Writes to Leader

```
Producer → [orders-0 leader on Broker 1]
Message: {orderId: 123, amount: 99.99}

Leader:
  - Appends to local log
  - Offset 1000 assigned
  - Message at offset 1000 is "uncommitted"
```

### Step 2: Followers Fetch from Leader

Followers continuously send fetch requests:

```
Follower (Broker 2) → Leader (Broker 1):
  "Give me data from offset 1000 onward"

Leader → Follower:
  Returns: [Message at offset 1000, 1001, 1002, ...]

Follower:
  - Appends to its local log
  - Sends ACK back (implicitly, via next fetch request)
```

**Important:** Followers fetch just like consumers (same fetch API).

### Step 3: Leader Tracks In-Sync Replicas (ISR)

Leader maintains a list of replicas that are "caught up":

```
orders-0:
  Leader: Broker 1
  ISR: [Broker 1, Broker 2, Broker 3]
```

A follower is in ISR if:
- It has fetched recently (within `replica.lag.time.max.ms`, default 10 seconds)
- It's not too far behind (within `replica.lag.max.messages`, deprecated in favor of time)

If a follower falls behind:

```
orders-0:
  Leader: Broker 1
  ISR: [Broker 1, Broker 2] ← Broker 3 removed (too slow)
```

### Step 4: Commit Point (High-Water Mark)

**High-Water Mark (HWM):** Highest offset replicated to ALL in-sync replicas.

```
Leader (Broker 1):
  Offset 1000: Written ✓
  Offset 1001: Written ✓
  Offset 1002: Written ✓ ← Leader's local end offset

Follower 1 (Broker 2):
  Offset 1000: Written ✓
  Offset 1001: Written ✓ ← Caught up to here

Follower 2 (Broker 3):
  Offset 1000: Written ✓ ← Caught up to here

High-Water Mark: 1000 (all ISR members have this)
```

**Consumers can only read up to the high-water mark.**

Why? Because offsets above HWM might be lost if the leader fails.

### Step 5: Producer Acknowledgment

Depending on `acks` config:

**`acks=1`** (leader acknowledgment):
- Leader writes to its log
- Immediately responds to producer
- **Faster, but risk of data loss** if leader fails before followers replicate

**`acks=all`** (all in-sync replicas):
- Leader writes to its log
- Waits for all ISR members to replicate
- Responds to producer
- **Slower, but no data loss** (as long as one ISR member survives)

**`acks=0`** (no acknowledgment):
- Producer doesn't wait at all
- **Fastest, highest risk of loss**

## Leader Election

When a leader fails, a new leader must be elected.

### The Old Way: ZooKeeper

Historically, Kafka used Apache ZooKeeper for coordination:

```
ZooKeeper:
  - Stores cluster metadata
  - Tracks which brokers are alive
  - Facilitates leader election
  - Stores consumer group offsets (old versions)
```

**Leader election process (ZooKeeper-based):**

1. Leader dies
2. ZooKeeper detects (no heartbeat)
3. **Controller** (one broker designated as controller) is notified
4. Controller picks a new leader from ISR
5. Controller updates metadata in ZooKeeper
6. All brokers refresh metadata

**Problems with ZooKeeper:**

- External dependency (operational complexity)
- Separate system to maintain
- Scalability limits (metadata size)
- Complex failure modes (ZooKeeper quorum issues)

### The New Way: KRaft (Kafka Raft)

**Starting Kafka 3.3+, ZooKeeper is being removed.** Replaced with **KRaft** (Kafka Raft protocol).

**KRaft overview:**

- Kafka implements its own Raft consensus protocol
- Metadata is stored in a Kafka topic (`__cluster_metadata`)
- No external dependencies
- Cleaner architecture

### How KRaft Works

**Cluster metadata stored as a log:**

```
__cluster_metadata topic:
  [Offset 0] Broker 1 joined
  [Offset 1] Broker 2 joined
  [Offset 2] Partition orders-0: Leader=Broker 1, ISR=[1,2,3]
  [Offset 3] Partition orders-0: Leader=Broker 2, ISR=[2,3] (failover!)
```

**Metadata controllers:**

A subset of brokers act as **controllers** (typically 3 or 5):

```
Controller Quorum:
  Controller 1 (Leader)
  Controller 2 (Follower)
  Controller 3 (Follower)
```

Controllers use Raft to maintain consistent metadata. Regular brokers are **observers** that replicate metadata.

**Leader election in KRaft:**

1. Leader dies
2. Controllers detect (no heartbeat)
3. Raft leader (controller leader) picks new partition leader from ISR
4. Write new metadata to `__cluster_metadata` log
5. All brokers tail the metadata log and update

**Benefits:**

- **Faster** - No external ZooKeeper calls
- **Simpler** - One system, not two
- **Scales better** - Metadata is a Kafka topic (uses Kafka's own optimizations)
- **Easier ops** - No ZooKeeper cluster to manage

**This is the future of Kafka. All new deployments should use KRaft.**

## Unclean Leader Election

What if ALL ISR members die?

```
orders-0:
  Leader: Broker 1 (died)
  ISR: [Broker 1, Broker 2, Broker 3]
  All brokers died!

Out-of-sync replica:
  Broker 4: Has offsets 0-500 (out of date)
```

**Options:**

1. **Wait for an ISR member to come back** (safe but unavailable)
2. **Elect Broker 4 as leader** (available but data loss)

**Config: `unclean.leader.election.enable`**

- `false` (default, safer): Wait for ISR member
- `true` (prioritize availability): Allow out-of-sync replica to become leader

**Trade-off:** Availability vs. durability

For most systems: Keep `false` (safety over availability)
For real-time systems with less critical data: Consider `true`

## Min In-Sync Replicas

What if you have `acks=all` but only 1 broker in ISR (the leader itself)?

```
orders-0:
  Leader: Broker 1
  ISR: [Broker 1] ← Only leader is in-sync!
```

Producer sends with `acks=all`:
- Leader writes
- No followers to wait for
- Immediately ACKs

**This defeats the purpose of `acks=all`!**

**Config: `min.insync.replicas=2`**

```
If ISR size < min.insync.replicas:
  - Partition becomes unavailable for writes
  - Producer gets NOT_ENOUGH_REPLICAS error
```

**Recommended: `min.insync.replicas = 2` with `replication.factor = 3`**

This ensures:
- At least 2 brokers have the data before ACK
- Can tolerate 1 broker failure (still have 2 in ISR)

## Quorum Writes (KRaft-Specific)

In KRaft, the metadata topic uses Raft quorum:

```
Replication factor = 5
Quorum = 3 (majority)

Write is committed when 3 out of 5 replicas have it.
```

This is different from data topics, which use ISR-based replication.

**Why?**

- Metadata needs strong consistency (Raft provides this)
- Data topics optimize for throughput (ISR is more flexible)

## Follower Fetching (Advanced)

**New in Kafka 2.4+: Follower Fetching**

Consumers can read from nearby followers instead of leader.

```
orders-0:
  Leader: Broker 1 (US East)
  Follower: Broker 2 (US West) ← Consumer can read from here

Consumer in US West reads from Broker 2 instead of Broker 1
```

**Benefits:**

- Reduced cross-region bandwidth
- Lower latency for geographically distributed consumers

**Limitations:**

- Only read up to high-water mark (same as leader)
- Requires `replica.selector.class` configuration

**Why not always use this?**

- Most clusters are in one datacenter (low latency anyway)
- Adds complexity
- Only useful for geo-distributed setups

## Replication Performance

**Replication is background work that doesn't block writes** (with `acks=1`).

But it consumes:
- **Network bandwidth** - Followers fetch data
- **Disk I/O** - Followers write data
- **CPU** - Compression/decompression (if used)

**Config tuning:**

```properties
# Number of fetcher threads per follower
num.replica.fetchers=4

# Replication throughput limit (bytes/sec)
follower.replication.throttled.rate=10000000
```

**Monitoring:**

- **Replica lag** - How far behind are followers?
- **Under-replicated partitions** - Partitions with ISR < replication factor

These are critical metrics for cluster health.

## The Guarantees

With proper configuration:

```properties
replication.factor=3
min.insync.replicas=2
acks=all
unclean.leader.election.enable=false
```

**You get:**

1. **No data loss** - Even if 1 broker dies
2. **High availability** - Can lose 1 broker and keep running
3. **Consistency** - All ISR members have same data up to HWM

**Trade-off:** Higher latency (~2-5ms extra for replication)

---

## Key Takeaways

- **Replication prevents data loss** - Multiple copies across brokers
- **Leader-based replication** - Simple, consistent, no conflicts
- **ISR (In-Sync Replicas)** - Brokers that are caught up
- **High-water mark** - Offset replicated to all ISR members
- **`acks=all` with `min.insync.replicas`** - Safe writes
- **KRaft replaces ZooKeeper** - Metadata as a Kafka topic
- **Unclean leader election** - Availability vs. durability trade-off
- **Follower fetching** - For geo-distributed consumers

## The Evolution

**Old Kafka (pre-3.3):**
```
[Producers] → [Kafka Brokers] ← [ZooKeeper] → [Coordination]
```

**Modern Kafka (KRaft):**
```
[Producers] → [Kafka Brokers + Controllers] → [Metadata as Kafka topic]
```

Simpler, faster, more scalable.

---

**[← Previous: Inside the Broker](./07-inside-the-broker.md) | [Back to Contents](./README.md) | [Next: The Producer's Perspective →](./09-producer-perspective.md)**
