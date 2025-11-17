# Chapter 13: Lifecycle and Storage Classes

> **"Not all data is created equal. Some is hot, some is cold, and some is frozen solid."**

## The Data Temperature Problem

You're reviewing storage costs:

```
Total S3 storage: 10 PB
Monthly cost: $230,000 ($0.023/GB × 10 PB)

Analysis:
  - 1 PB: Accessed daily (hot)
  - 3 PB: Accessed monthly (warm)
  - 6 PB: Never accessed (cold)

Question: Why pay the same price for data that's never touched?
```

**The insight**: Data access patterns vary dramatically.

## Storage Classes

### The Tiered Storage Concept

**Different storage for different access patterns**:

```
┌─────────────────────┬──────────┬──────────┬───────────┐
│ Storage Class       │ Cost/GB  │ Access   │ Use Case  │
├─────────────────────┼──────────┼──────────┼───────────┤
│ STANDARD            │ $0.023   │ Fast     │ Hot data  │
│ STANDARD_IA         │ $0.0125  │ Fast     │ Warm data │
│ GLACIER_IR          │ $0.004   │ Minutes  │ Cool data │
│ GLACIER_FLEXIBLE    │ $0.0036  │ Hours    │ Cold data │
│ GLACIER_DEEP_ARCHIVE│ $0.00099 │ 12 hours │ Archive   │
└─────────────────────┴──────────┴──────────┴───────────┘

Price range: 23x difference!
```

### STANDARD Class

**Default storage class**:

```
Characteristics:
  - Price: $0.023/GB/month
  - Retrieval: Millisecond latency
  - Retrieval cost: Free
  - Minimum storage: None
  - Minimum duration: None
  - Durability: 11 nines
  - Availability: 99.99%

Storage architecture:
  - SSD + HDD mix
  - Multiple AZs
  - Full replication or erasure coding
  - Optimized for frequent access

Use case:
  - Active websites
  - Mobile apps
  - Content distribution
  - Frequently accessed data
```

### STANDARD_IA (Infrequent Access)

**For less-frequently accessed data**:

```
Characteristics:
  - Price: $0.0125/GB/month (46% cheaper!)
  - Retrieval: Millisecond latency (same as STANDARD)
  - Retrieval cost: $0.01/GB (NEW!)
  - Minimum storage: 128 KB (charged even if smaller)
  - Minimum duration: 30 days (charged even if deleted earlier)
  - Durability: 11 nines
  - Availability: 99.9%

When to use:
  ✓ Accessed less than once per month
  ✓ Objects > 128 KB
  ✓ Kept for > 30 days

Example: Backups, old logs, archived documents
```

**Cost comparison**:

```
Scenario: 1 TB stored for 1 year, accessed once

STANDARD:
  Storage: $0.023 × 1000 GB × 12 months = $276
  Retrieval: $0
  Total: $276

STANDARD_IA:
  Storage: $0.0125 × 1000 GB × 12 months = $150
  Retrieval: $0.01 × 1000 GB = $10
  Total: $160

Savings: $116 (42% cheaper)
```

### GLACIER Classes

**For archival data**:

#### GLACIER Instant Retrieval

```
Characteristics:
  - Price: $0.004/GB/month (83% cheaper than STANDARD!)
  - Retrieval: Milliseconds
  - Retrieval cost: $0.03/GB
  - Minimum storage: 128 KB
  - Minimum duration: 90 days
  - Durability: 11 nines
  - Availability: 99.9%

Use case: Rarely accessed data that needs instant retrieval
  Example: Medical images, customer records
```

#### GLACIER Flexible Retrieval

```
Characteristics:
  - Price: $0.0036/GB/month (84% cheaper!)
  - Retrieval: 1-5 hours (Expedited: 1-5 min)
  - Retrieval cost: $0.02/GB (Standard)
  - Minimum storage: 40 KB
  - Minimum duration: 90 days
  - Durability: 11 nines
  - Availability: 99.99%

Retrieval tiers:
  - Expedited: 1-5 minutes, $0.03/GB
  - Standard: 3-5 hours, $0.01/GB
  - Bulk: 5-12 hours, $0.0025/GB

Use case: Long-term backups, disaster recovery
  Example: Annual financial records
```

#### GLACIER Deep Archive

```
Characteristics:
  - Price: $0.00099/GB/month (96% cheaper!)
  - Retrieval: 12-48 hours
  - Retrieval cost: $0.02/GB
  - Minimum storage: 40 KB
  - Minimum duration: 180 days
  - Durability: 11 nines
  - Availability: 99.99%

Use case: Rarely/never accessed archival
  Example: Compliance data (7-10 year retention)
```

### How Glacier Actually Works

**Glacier is not tape!**

```
Common misconception:
  "Glacier = tape storage"
  ✗ Wrong!

Reality:
  - Still disk-based (HDD)
  - Optimized for sequential reads
  - Packed densely
  - Powered down when idle
  - Spun up on retrieval
```

**Storage architecture**:

```
STANDARD:
  - Online, always spinning
  - Multiple replicas in fast tier
  - Optimized for random access

GLACIER:
  - Offline, mostly powered down
  - Densely packed (less redundancy overhead)
  - Optimized for sequential access
  - Spin up on demand

Savings:
  - Reduced power costs
  - Reduced cooling costs
  - Higher density (more data per disk)
  - Reduced network overhead
```

**Retrieval process**:

```
GLACIER Flexible Retrieval request:

1. Client: Initiate retrieval
   POST /object?restore&days=1

2. S3: Queue job
   - Find which disks have the data
   - Schedule retrieval

3. Storage: Power up disks
   - Spin up drives
   - Read data (sequential)

4. S3: Stage to STANDARD tier (temp)
   - Available for 1 day
   - Then deleted

5. Client: Download from STANDARD
   GET /object

Time: 3-5 hours (Standard retrieval)
```

## Lifecycle Policies

### What Are Lifecycle Policies?

**Automatic data management rules**:

```
Lifecycle policy = set of rules that:
  - Transition objects between storage classes
  - Delete objects after expiration
  - Clean up old versions
  - Manage multipart upload parts

Applied automatically (no manual work!)
```

### Transition Actions

**Move objects to cheaper storage over time**:

```
Rule: "Transition old logs to cheaper storage"

Transitions:
  - Day 0: Upload to STANDARD
  - Day 30: Transition to STANDARD_IA
  - Day 90: Transition to GLACIER
  - Day 365: Transition to DEEP_ARCHIVE
  - Day 2555: Delete (7 years retention)

┌────────┐  30d  ┌────────┐  60d  ┌────────┐  275d ┌────────┐  2190d
│STANDARD├───────►│IA      ├───────►│GLACIER ├───────►│DEEP_AR ├───────► DELETE
└────────┘       └────────┘       └────────┘       └────────┘
$0.023/GB        $0.0125/GB       $0.0036/GB       $0.00099/GB
```

**Cost savings**:

```
1 TB stored for 7 years (2555 days):

Without lifecycle:
  STANDARD for 7 years: $0.023 × 1000 × 84 months = $1,932

With lifecycle:
  30 days STANDARD: $0.023 × 1000 × 1 = $23
  60 days IA: $0.0125 × 1000 × 2 = $25
  275 days GLACIER: $0.0036 × 1000 × 9.2 = $33
  2190 days DEEP_AR: $0.00099 × 1000 × 73 = $72
  Total: $153

Savings: $1,779 (92% cheaper!)
```

### Expiration Actions

**Automatically delete old data**:

```
Rule: "Delete old log files"

Action:
  - Expire objects older than 90 days
  - Automatically DELETE

Benefits:
  ✓ Reduce storage costs
  ✓ Reduce clutter
  ✓ Compliance (don't keep data longer than required)
```

**Example**: Temporary data

```
Rule: "Expire temp files after 1 day"

Application uploads:
  /temp/session-abc123 (timestamp: 2024-01-15 10:00)

Lifecycle:
  2024-01-16 10:00 → File auto-deleted

No manual cleanup needed!
```

### Version Management

**Clean up old versions**:

```
Rule: "Keep only last 30 days of versions"

Actions:
  - Expire old non-current versions after 30 days
  - Delete expired delete markers

Example:
  Object has 100 versions over 1 year
  Without rule: Store 100 versions ($$$)
  With rule: Store ~30 versions (recent ones)

Savings: 70% reduction in version storage
```

### Incomplete Multipart Uploads

**Clean up abandoned uploads**:

```
Problem:
  User starts multipart upload
  Upload fails halfway
  Parts remain in storage (consuming space)

Rule: "Delete incomplete uploads after 7 days"

Action:
  - Automatically delete abandoned parts
  - Free up space

Typical savings: 1-5% of storage
```

## Lifecycle Policy Examples

### Example 1: Log Aggregation Pipeline

```json
{
  "Rules": [
    {
      "Id": "Log-Lifecycle",
      "Filter": {"Prefix": "logs/"},
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

**Lifecycle**:
```
Day 0-29: STANDARD (fast access for recent logs)
Day 30-89: STANDARD_IA (less frequent access)
Day 90-364: GLACIER_IR (archival, but retrievable quickly if needed)
Day 365-2554: DEEP_ARCHIVE (compliance retention)
Day 2555: DELETE (7-year retention met)
```

### Example 2: Media Archive

```json
{
  "Rules": [
    {
      "Id": "Media-Archive",
      "Filter": {"Prefix": "media/"},
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 60,
          "StorageClass": "GLACIER_FLEXIBLE"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30
      }
    }
  ]
}
```

**Lifecycle**:
```
Day 0-59: STANDARD (active media)
Day 60+: GLACIER (archived, retrieval in hours OK)
Old versions: Delete after 30 days (keep current + 1 month history)
```

### Example 3: Temporary Files

```json
{
  "Rules": [
    {
      "Id": "Temp-Cleanup",
      "Filter": {"Prefix": "temp/"},
      "Status": "Enabled",
      "Expiration": {
        "Days": 1
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 1
      }
    }
  ]
}
```

**Lifecycle**:
```
Day 1: DELETE all temp files
Incomplete uploads: Abort after 1 day
```

## Intelligent-Tiering

### The Auto-Optimization Class

**Problem**: Can't predict access patterns for all objects.

**Solution**: S3 Intelligent-Tiering (auto-moves data based on access)

```
How it works:
  1. Monitors access patterns
  2. Automatically moves objects between tiers
  3. No retrieval fees (!)
  4. Small monthly monitoring fee: $0.0025 per 1000 objects

Tiers:
  - Frequent Access: $0.023/GB (default)
  - Infrequent Access: $0.0125/GB (no access for 30 days)
  - Archive Instant Access: $0.004/GB (no access for 90 days)
  - Archive Access: $0.0036/GB (optional, no access for 90-270 days)
  - Deep Archive: $0.00099/GB (optional, no access for 180-730 days)
```

**Example**:

```
Object: /images/logo.png

Month 1: Accessed 1000 times → Frequent tier ($0.023/GB)
Month 2: Accessed 10 times → Frequent tier ($0.023/GB)
Month 3: Accessed 0 times → Move to Infrequent tier ($0.0125/GB)
Month 6: Still 0 access → Move to Archive tier ($0.004/GB)
Month 12: Still 0 access → Move to Deep Archive ($0.00099/GB)
Month 13: Accessed once → Move back to Frequent tier ($0.023/GB)

All automatic!
```

**When to use**:

```
Use Intelligent-Tiering when:
  ✓ Access patterns unknown or changing
  ✓ Mixed hot/cold data
  ✓ Don't want to manage lifecycle policies

Don't use when:
  ✗ Access patterns predictable (use lifecycle policies instead)
  ✗ Millions of tiny objects (monitoring fee adds up)
```

## Storage Class Selection Guide

### Decision Tree

```
Is data accessed frequently (>1/month)?
  YES → STANDARD
  NO → Continue

Is data accessed occasionally (<1/month but >1/year)?
  YES → STANDARD_IA
  NO → Continue

Is millisecond retrieval required?
  YES → GLACIER_IR
  NO → Continue

Is hour-scale retrieval OK?
  YES → GLACIER_FLEXIBLE
  NO → GLACIER_DEEP_ARCHIVE (retrieval in 12 hours)
```

### Cost Optimization Strategies

**Strategy 1: Analyze access patterns**

```
Use S3 Storage Class Analysis:
  - Monitors access for 30+ days
  - Recommends optimal storage class
  - Shows potential savings

Example output:
  "70% of objects in STANDARD haven't been accessed in 90 days"
  "Recommendation: Transition to STANDARD_IA"
  "Estimated savings: $50,000/month"
```

**Strategy 2: Tag-based policies**

```
Tag objects by business value:

Critical: No lifecycle (keep in STANDARD)
  Tag: critical=true

Important: Transition after 6 months
  Tag: important=true
  Lifecycle: STANDARD → IA (180 days) → GLACIER (365 days)

Normal: Transition after 1 month
  Tag: normal=true
  Lifecycle: STANDARD → IA (30 days) → GLACIER (90 days)

Temporary: Delete after 7 days
  Tag: temp=true
  Lifecycle: DELETE (7 days)
```

**Strategy 3: Use Intelligent-Tiering for unknowns**

```
New application:
  - Don't know access patterns yet
  - Use Intelligent-Tiering for first 6 months
  - Analyze patterns
  - Create custom lifecycle policies
  - Switch to optimized classes
```

## Performance Implications

### Retrieval Latency by Class

```
STANDARD:               1-10ms
STANDARD_IA:            1-10ms (same as STANDARD)
GLACIER_IR:             1-10ms (Instant Retrieval)
GLACIER_FLEXIBLE:
  - Expedited:          1-5 minutes
  - Standard:           3-5 hours
  - Bulk:               5-12 hours
GLACIER_DEEP_ARCHIVE:
  - Standard:           12 hours
  - Bulk:               48 hours
```

### Throughput Considerations

```
STANDARD:
  - Optimized for random access
  - Full throughput available

GLACIER:
  - Optimized for sequential access
  - Throughput limited during retrieval
  - Restored to STANDARD for download

Provisioned capacity:
  - For GLACIER: Can provision retrieval capacity
  - Guarantees retrieval slots
  - Useful for disaster recovery testing
```

## Key Takeaways

- **Storage classes enable cost optimization** - 96% savings possible
- **Lifecycle policies automate management** - No manual work required
- **Intelligent-Tiering for unknown patterns** - Automatic optimization
- **Access patterns drive class selection** - Hot, warm, cold, frozen
- **Retrieval costs matter** - Factor into total cost
- **Minimum storage durations exist** - Early deletion charged full duration
- **Glacier is disk, not tape** - Just optimized differently
- **Monitor and adjust** - Use analytics to optimize

## What's Next?

We've optimized cost through storage classes. But what about performance? In the next chapter, we'll explore performance optimization techniques.

---

**[← Previous: Versioning and Time Travel](./12-versioning.md) | [Back to Contents](./README.md) | [Next: Performance and Optimization →](./14-performance-optimization.md)**
