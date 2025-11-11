# Chapter 3: The Database Trap

> **"When you have a hammer, everything looks like a nail. When you have a database, everything looks like a table."**

## The Obvious Solution?

After struggling with message queues, you have an idea: *"We already have a database. It stores data. It's durable. It handles concurrent access. Why not use it for our event log?"*

This seems reasonable. Let's try it.

## Attempt 1: Events Table

You create a simple schema:

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_created_at (created_at)
);
```

Your producer writes events:

```javascript
async function placeOrder(orderData) {
    const order = await db.orders.create(orderData);

    // Write event to events table
    await db.events.create({
        event_type: 'ORDER_PLACED',
        event_data: orderData
    });

    return order;
}
```

Your consumers poll for new events:

```javascript
// Analytics service consumer
setInterval(async () => {
    const events = await db.query(`
        SELECT * FROM events
        WHERE id > $1
        ORDER BY id ASC
        LIMIT 100
    `, [lastProcessedId]);

    for (const event of events) {
        await processEvent(event);
        lastProcessedId = event.id;
    }
}, 1000); // Poll every second
```

## Why This Seems Good

At first glance, this works:

✓ **Ordered** - Auto-incrementing ID ensures order
✓ **Durable** - Database persistence is proven
✓ **Replayable** - Query any historical events
✓ **Multiple readers** - Each service tracks its own lastProcessedId

Your team is happy. You've solved it with familiar technology!

## The Problems Emerge

### Problem 1: Polling is Inefficient

Every consumer is polling the database:

```
Second 1: SELECT * FROM events WHERE id > 12345 ... (0 new rows)
Second 2: SELECT * FROM events WHERE id > 12345 ... (0 new rows)
Second 3: SELECT * FROM events WHERE id > 12345 ... (2 new rows!)
Second 4: SELECT * FROM events WHERE id > 12346 ... (0 new rows)
```

You have 10 consumers, each polling every second. That's **10 queries per second** just to check for new data—most of which return nothing.

**This is wasteful:**
- Database CPU time spent on empty queries
- Network bandwidth for queries that return no data
- Consumer CPU time waiting and checking

And if you decrease the polling interval for lower latency, the waste multiplies.

### Problem 2: The Lock Contention

PostgreSQL is an MVCC database, but it still has to manage:
- Transaction isolation
- Index updates
- WAL (Write-Ahead Log) writes

As you scale to thousands of writes per second, you see:
- **Lock contention** on the index
- **Vacuum** processes slowing things down
- **Bloat** in the table from updates and deletes

Your database is designed for **transactional integrity**, not **high-throughput append-only operations**.

### Problem 3: The Read Amplification

Now you have 10 services all reading the same events. For every event written:
- Analytics reads it
- Fraud detection reads it
- Recommendations read it
- Inventory reads it
- ...10 total reads

That's a **10x read amplification**. Your database is reading the same data from disk 10 times.

With 100,000 events per day and 10 consumers:
- **1 million reads per day** just for events

Your database I/O is becoming a bottleneck.

### Problem 4: The Data Retention Problem

You decide to keep events for 30 days. Your events table now has:
- 100,000 events/day × 30 days = **3 million rows**

And growing. You need to:
- Periodically delete old events (expensive DELETE operations)
- Or archive them (complex process)
- Maintain indexes on a growing table (slower queries)

PostgreSQL's `DELETE` operation:
1. Marks rows as deleted (doesn't free space immediately)
2. Requires VACUUM to reclaim space
3. Still scans deleted rows during queries (until vacuum)

**This is not what databases are optimized for.**

### Problem 5: The Ordering Guarantee is Fragile

Your auto-incrementing ID seems to guarantee order, but what about:

**Race conditions:**
```
Transaction A: Starts, gets ID 100
Transaction B: Starts, gets ID 101, COMMITS
Transaction A: Still processing... then COMMITS

Events written: [101] then [100]
```

In PostgreSQL with concurrent transactions, **sequence numbers don't guarantee commit order**.

You need serializable isolation, which kills performance.

### Problem 6: You Can't Scale Horizontally

As your write throughput grows, you need to scale. But databases are hard to scale horizontally:

- **Read replicas** help with reads but not writes
- **Sharding** is complex and fragile
- **Multi-master** replication has conflict resolution problems

Your single database is a bottleneck.

## The Deeper Problem: Impedance Mismatch

Databases are designed for:
- **Mutable data** (UPDATE, DELETE)
- **Random access** (indexed queries)
- **Transactional integrity** (ACID)
- **Complex queries** (JOINs, aggregations)

But you need:
- **Immutable data** (append-only)
- **Sequential access** (read in order)
- **High throughput** (lots of simple writes)
- **Simple queries** (give me events after offset X)

**This is an impedance mismatch.** You're using the wrong tool for the job.

## The Comparison

| Requirement | Database | What You Actually Need |
|-------------|----------|----------------------|
| Write pattern | ACID transactions, complex | Append-only, simple |
| Read pattern | Indexed queries, random access | Sequential scan |
| Data lifecycle | Mutable, normalized | Immutable, denormalized |
| Scaling | Vertical (bigger machine) | Horizontal (more machines) |
| Latency profile | ~5-10ms per operation | ~1-2ms for batch |
| Throughput | ~10K operations/sec | ~100K+ messages/sec |

## The Realization

You need a storage system that:

1. **Optimizes for append-only writes** - No indexes, no transactions, just append
2. **Optimizes for sequential reads** - Read chunks of data in order
3. **Supports multiple independent readers** - Without read amplification
4. **Scales horizontally** - Add machines to handle more load
5. **Has simple, predictable retention** - Delete old data by time, not by row
6. **Provides ordering guarantees** - Within a partition
7. **Is durable** - Don't lose data

**This is NOT a database. This is a specialized log storage system.**

## The Enlightenment

You start researching and find a pattern that keeps appearing:

> **"The log is the database, inside-out."**

Databases use logs internally (WAL - Write-Ahead Log) for durability and replication. What if we:
- Made the log the PRIMARY abstraction (not hidden internals)
- Optimized everything for log operations
- Let consumers track their own position

This is the core insight that leads to systems like Kafka.

## What We're Building Toward

A system where:

```
[Write] → Append to log (optimized, ~1ms)
          ↓
    [Distributed Log]
          ↓
    [Multiple Readers] (each tracks position)
```

- **Writes are fast** - Just append, no complex operations
- **Reads are fast** - Sequential I/O, OS page cache helps
- **Scale is horizontal** - Partition the log across machines
- **Retention is simple** - Delete old log segments by time

---

## Key Takeaways

- **Databases are not optimized for event streams** - Different access patterns
- **Polling is wasteful** - Empty queries consume resources
- **Read amplification is real** - Same data read multiple times
- **Append-only workloads need specialized systems** - Not general-purpose databases
- **The log as primary abstraction** - Turn database internals inside-out

## The Mental Model

**Database thinking:**
```
Write → Complex transaction → Index updates → Query with WHERE clause
```

**Log thinking:**
```
Write → Append to end → Readers track position → Read sequentially
```

Simplicity enables performance.

---

**[← Previous: The Cracks Appear](./02-the-cracks-appear.md) | [Back to Contents](./README.md) | [Next: The Append-Only Revelation →](./04-append-only-revelation.md)**
