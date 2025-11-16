# Chapter 1: The Impossible Requirements

> **"The best products start with requirements that seem impossible."**

## The Year is 2006

You're an engineer at Amazon. The company is growing explosively. But there's a problem—actually, many problems.

### The Storage Crisis

Amazon's infrastructure is drowning in data:
- **Product catalog**: Millions of product images, descriptions
- **Customer data**: Orders, reviews, wishlists
- **Analytics**: Clickstream data, logs, metrics
- **Engineering**: Code artifacts, backups, archives

Each team has built their own storage solution:
- The photo team has custom image servers
- The backup team has tape libraries
- The analytics team has Hadoop clusters
- The web team has NFS mounts

**Result**: Chaos. Duplicated effort. Inconsistent reliability. Skyrocketing costs.

## The Vision

Leadership comes with an audacious idea:

> *"What if we built ONE storage service that could handle ALL of Amazon's storage needs—and then sold it to the world?"*

You're assigned to the team. Your first meeting starts with requirements that make your head spin.

## Requirement 1: Infinite Scale

**The Ask**: "The system should store unlimited data."

```
Year 1: 1 billion objects
Year 2: 10 billion objects
Year 5: 100 billion objects
Year 10: Trillions of objects?
```

**Your reaction**: "How do you design for 'unlimited'?"

Traditional systems think in terms of:
- A single machine: Terabytes
- A cluster: Petabytes
- A data center: Maybe an exabyte if you're Google

**This requirement**: No upper bound. Ever.

### The Implications

This rules out almost everything you know:

**Single machine?** Maxes out at ~100 TB
```
❌ Won't work
```

**Traditional SAN/NAS?** Gets exponentially expensive beyond petabytes
```
❌ Won't work
```

**Custom cluster?** Eventually hits architectural limits
```
❌ Might work for a while, but will need redesign
```

**Question**: How do you build a system that NEVER needs to be redesigned as it scales?

## Requirement 2: 99.999999999% Durability

**The Ask**: "If a customer stores 10 million objects, we can lose at most ONE object every 10,000 years."

**You**: "That's 11 nines. Is that even possible?"

**Leadership**: "It's not optional. Customer data is sacred."

### The Math

```
99.999999999% durability = 0.00000000001% annual loss rate

For 10 billion objects:
- Expected loss: 0.1 objects per year
- Acceptable loss: ~1 object per decade
```

### What This Means

If you have 1 million drives:
- Drive failure rate: ~4% per year (typical)
- Expected failures: 40,000 drives per year
- That's 109 drives per day

**You need to survive**:
- Disk failures (constant)
- Server failures (hourly)
- Rack failures (weekly)
- Data center failures (rare but happen)

**Single replication won't cut it.**

**Triple replication?** Maybe, but:
- 3x the cost
- Still vulnerable to correlated failures
- What about bit rot?

## Requirement 3: Millions of Requests per Second

**The Ask**: "Support millions of concurrent users with low latency."

```
Black Friday 2006: 100,000 RPS (requests per second)
Black Friday 2010: 1,000,000 RPS
Black Friday 2020: 10,000,000 RPS?
```

**The Challenge**: Each request needs to:
1. Authenticate the user
2. Check permissions
3. Find the object (among billions)
4. Read from disk
5. Return data

**At 1 million RPS**:
- 1 microsecond per request budget? Impossible.
- 1 millisecond? Tight.
- 10 milliseconds? Achievable, but difficult.

### The Distribution Problem

Users are global:
- US East Coast
- US West Coast
- Europe
- Asia
- South America

**You can't serve them all from one data center.**

Network latency alone:
```
US East → US West: ~60ms
US → Europe: ~100ms
US → Asia: ~150ms
```

**You need**:
- Global distribution
- Low latency everywhere
- Consistency across regions

## Requirement 4: Pennies Per Gigabyte

**The Ask**: "Storage should cost $0.01 per GB per month."

**You**: "But enterprise SAN storage costs $2-5 per GB!"

**Leadership**: "We're not building enterprise storage. We're building commodity storage at internet scale."

### The Cost Breakdown

To hit $0.01/GB/month:

```
Annual revenue per TB: $10/GB × 1000 = $10,000/year
Annual cost budget (40% margin): $6,000/year
```

**Where costs come from**:
- Hardware: Disks, servers, networking
- Data center: Power, cooling, real estate
- Operations: People, bandwidth
- Durability: Replication overhead

**Traditional approach**:
```
High-end disks: $2/GB
RAID controllers: $5,000 each
SAN fabric: $50,000+
-----------------
Total: Impossible to hit $0.01/GB
```

**What you need**:
```
Commodity disks: $0.10/GB (in bulk)
Commodity servers: $2,000 each
Commodity networking: $500/switch
Custom software: Your job!
```

**The realization**: You need to build EVERYTHING yourself.

## Requirement 5: 99.99% Availability

**The Ask**: "Four nines. Less than 1 hour of downtime per year."

```
99.99% uptime = 52.56 minutes of downtime per year
               = 4.38 minutes per month
               = 1.01 minutes per week
```

**But** you're building on commodity hardware that fails constantly:

```
Daily failures (at scale):
- 100+ disk failures
- 10+ server failures
- 1+ switch failure
- 0.1 power distribution unit failure
```

**How do you provide 99.99% availability when your infrastructure provides 90% reliability?**

## Requirement 6: Simple API

**The Ask**: "Developers should be able to use it without reading 500 pages of docs."

**Current state of storage APIs**:
- POSIX: Decades old, complex semantics
- Database: SQL, transactions, schema management
- SAN/NAS: Complex mount procedures, OS dependencies

**What developers want**:
```
PUT /bucket/key → stores object
GET /bucket/key → retrieves object
DELETE /bucket/key → removes object
```

**That's it.**

But behind that simple API, you need to handle:
- Authentication
- Authorization
- Versioning
- Metadata
- Lifecycle
- Encryption
- Compression
- Replication
- Consistency
- Billing

## The Impossible Triangle

You sketch out the challenge:

```
        ┌─────────────┐
        │   SCALE     │
        │ (unlimited) │
        └─────────────┘
             /  \
            /    \
           /      \
          /        \
         /          \
┌──────────┐    ┌──────────┐
│DURABILITY│    │   COST   │
│(11 nines)│    │(pennies) │
└──────────┘    └──────────┘
```

Pick any two is hard. All three seems impossible.

### Historical Perspective

**What exists in 2006**:

**Enterprise SAN/NAS**:
- ✓ Durability (RAID, backups)
- ✓ Performance (fast)
- ✗ Scale (limited)
- ✗ Cost ($$$$)

**Hadoop/HDFS**:
- ✓ Scale (pretty good)
- ✓ Cost (commodity)
- ✗ Durability (3x replication, but not 11 nines)
- ✗ API (complex, batch-oriented)

**Databases**:
- ✓ Durability (ACID)
- ✓ API (SQL)
- ✗ Scale (vertical scaling)
- ✗ Cost (high)

**None of these solve the problem.**

## The First Questions

You sit with your whiteboard and start asking:

### Question 1: What are we actually storing?

Is it:
- Files? (like a file system)
- Blocks? (like a SAN)
- Records? (like a database)
- Something else?

### Question 2: What consistency guarantees do we need?

Can we:
- Have eventual consistency?
- Accept stale reads?
- Allow conflicting writes?

Or do we need:
- Strong consistency?
- Linearizability?
- Transactions?

### Question 3: How do we partition data?

Do we partition by:
- Hash of key?
- Range of keys?
- Geographic location?
- Random placement?

### Question 4: How do we ensure durability?

Options:
- Replication (copies of data)
- Erasure coding (mathematical redundancy)
- Geo-distribution (cross-data-center)
- Backups (separate system)

### Question 5: Where do we spend our resource budget?

We can optimize for:
- Read performance
- Write performance
- Storage efficiency
- Network efficiency

**We can't optimize for everything.**

## The Realization

You realize that the requirements force a specific type of solution:

**Because scale must be unlimited**:
→ Architecture must be horizontally scalable
→ No single bottleneck can exist
→ System must be partition-tolerant

**Because cost must be pennies**:
→ Must use commodity hardware
→ Must accept hardware failures
→ Software must be extremely efficient

**Because durability must be 11 nines**:
→ Multi-layer redundancy required
→ Geographic distribution needed
→ Automated recovery is mandatory

**Because API must be simple**:
→ Can't support complex filesystem semantics
→ Can't provide strong consistency everywhere
→ Must focus on core operations

**These constraints point toward a NEW type of storage**—one that doesn't exist yet.

## The Challenge Ahead

You have impossible requirements and no existing solution.

Time to invent one.

The journey begins with a simple question:

> **"What if we throw away everything we know about storage and start from first principles?"**

---

## Key Takeaways

- **Impossible requirements drive innovation** - Constraints force creative solutions
- **Traditional approaches won't scale** - File systems, databases, and SANs all have limits
- **Trade-offs are inevitable** - You can't optimize for everything
- **The API shapes the architecture** - Simple interfaces hide complex implementations
- **Cost drives design** - Pennies per GB means commodity hardware and custom software

## What's Next?

In the next chapter, we'll explore why traditional storage systems fail at this scale, and what specific limitations we need to overcome.

---

**[← Back to Contents](./README.md) | [Next Chapter: Why Traditional Storage Fails →](./02-why-traditional-fails.md)**
