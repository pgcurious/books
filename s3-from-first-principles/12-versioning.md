# Chapter 12: Versioning and Time Travel

> **"The best backup is one you don't have to think about."**

## The Accidental Delete Problem

Your phone rings at 3 AM. A customer:

> *"I accidentally deleted my entire photo library. 10 years of memories. 50,000 photos. Can you recover them?"*

**Without versioning**: "Sorry, they're gone forever."

**With versioning**: "Give me 5 minutes."

## What is Versioning?

**Versioning keeps every version of every object**:

```
Without versioning:
  PUT /photos/beach.jpg (v1)
  PUT /photos/beach.jpg (v2)  ← Overwrites v1
  DELETE /photos/beach.jpg    ← v2 is gone forever

With versioning:
  PUT /photos/beach.jpg (v1)    ← Stored
  PUT /photos/beach.jpg (v2)    ← v1 still exists!
  DELETE /photos/beach.jpg      ← Both v1 and v2 still exist!
                                  (delete is just a marker)
```

**Key insight**: Nothing is ever truly deleted.

## How Versioning Works

### Version IDs

**Each version gets a unique ID**:

```
PUT /photos/beach.jpg (first upload)
→ Version ID: "abc123xyz"

PUT /photos/beach.jpg (second upload)
→ Version ID: "def456uvw"

PUT /photos/beach.jpg (third upload)
→ Version ID: "ghi789rst"

Storage:
┌────────────────────────────────┐
│ Key: "photos/beach.jpg"        │
├────────────────────────────────┤
│ Version: "ghi789rst" (current) │  ← Latest
│ Size: 2 MB                     │
│ Data: <bytes>                  │
├────────────────────────────────┤
│ Version: "def456uvw"           │  ← Previous
│ Size: 1.8 MB                   │
│ Data: <bytes>                  │
├────────────────────────────────┤
│ Version: "abc123xyz"           │  ← Original
│ Size: 2.1 MB                   │
│ Data: <bytes>                  │
└────────────────────────────────┘
```

**Version ID generation**:

```
Requirements:
  - Globally unique
  - Time-ordered (newer = later)
  - Hard to guess (security)

Implementation:
  - Timestamp (milliseconds) + Random suffix
  - Or: UUID v7 (time-ordered UUID)
  - Or: Snowflake ID (Twitter's approach)

Example:
  "20240115-103045-a3f2b8d4"
  (date-time-random)
```

### Accessing Versions

**Get latest version** (default):

```
GET /photos/beach.jpg
→ Returns version "ghi789rst" (current)
```

**Get specific version**:

```
GET /photos/beach.jpg?versionId=def456uvw
→ Returns version "def456uvw" (previous)

GET /photos/beach.jpg?versionId=abc123xyz
→ Returns version "abc123xyz" (original)
```

**List all versions**:

```
GET /photos/?versions
→ Returns:
  {
    "Versions": [
      {"Key": "beach.jpg", "VersionId": "ghi789rst", "IsLatest": true},
      {"Key": "beach.jpg", "VersionId": "def456uvw", "IsLatest": false},
      {"Key": "beach.jpg", "VersionId": "abc123xyz", "IsLatest": false}
    ]
  }
```

## Deletes with Versioning

### Soft Delete (Delete Marker)

**Delete doesn't remove data**:

```
DELETE /photos/beach.jpg
→ Creates "delete marker" (special version)

Storage after delete:
┌────────────────────────────────┐
│ Key: "photos/beach.jpg"        │
├────────────────────────────────┤
│ DELETE MARKER (current)        │  ← Marks as "deleted"
│ Version: "jkl012mno"           │
├────────────────────────────────┤
│ Version: "ghi789rst"           │  ← Still exists!
│ Size: 2 MB                     │
│ Data: <bytes>                  │
├────────────────────────────────┤
│ Version: "def456uvw"           │
│ ...                            │
└────────────────────────────────┘

GET /photos/beach.jpg
→ 404 Not Found (delete marker is current)

GET /photos/beach.jpg?versionId=ghi789rst
→ Returns the data (version still exists!)
```

**Undelete** (remove delete marker):

```
DELETE /photos/beach.jpg?versionId=jkl012mno
→ Removes the delete marker

Now:
GET /photos/beach.jpg
→ Returns version "ghi789rst" (latest non-delete version)

File is back!
```

### Hard Delete

**Permanently delete a version**:

```
DELETE /photos/beach.jpg?versionId=abc123xyz
→ Permanently removes version "abc123xyz"

Now only "def456uvw" and "ghi789rst" remain.
```

**Delete marker vs hard delete**:

```
DELETE /key              → Creates delete marker (soft delete)
DELETE /key?versionId=X  → Permanently deletes version X (hard delete)

Delete marker: Reversible
Hard delete: Permanent
```

## Storage Implications

### Storage Costs

**Versioning stores all versions**:

```
Example:
  Day 1: Upload 1 GB file (v1)
  Day 2: Modify and upload 1 GB file (v2)
  Day 3: Modify and upload 1 GB file (v3)

Without versioning:
  Storage: 1 GB (only v3)

With versioning:
  Storage: 3 GB (v1 + v2 + v3)
  Cost: 3x higher!
```

**For frequently updated objects**:

```
Log file updated every minute:
  1 MB per update
  1440 updates per day
  Storage per day: 1.44 GB
  Storage per month: ~43 GB
  Cost: $0.43/month (vs $0.01 without versioning)
```

**When versioning is expensive**:

```
Scenarios:
  - Log files (frequent updates)
  - Temporary files
  - Cache data
  - Large files updated often

Better approach:
  - Don't version these
  - Or use lifecycle policies (Chapter 13)
  - Or version only critical data
```

### Managing Version Bloat

**Problem**: Old versions accumulate.

```
File updated 1000 times over 1 year:
  Each version: 10 MB
  Total: 10 GB
  But only latest version matters

Cost: 100x higher than necessary!
```

**Solutions**:

1. **Lifecycle policies** (automatic cleanup):

```
Rule: Delete versions older than 30 days
  v1 (90 days old) → DELETE
  v2 (60 days old) → DELETE
  v3 (20 days old) → KEEP
  v4 (current)     → KEEP

Reduces storage to recent versions only.
```

2. **Transition old versions to cheaper storage**:

```
Rule: Move versions older than 7 days to Glacier
  v1-v10 (old) → Glacier ($0.004/GB)
  v11-v12 (recent) → Standard ($0.023/GB)

Keeps history but reduces cost.
```

3. **Limit version count**:

```
Rule: Keep only last 10 versions
  v1-v90 → DELETE
  v91-v100 → KEEP

Caps storage growth.
```

## Metadata Management

### Storing Version Metadata

**Metadata index entry**:

```
Key: "photos/beach.jpg"
Versions:
  [
    {
      "VersionId": "ghi789rst",
      "IsLatest": true,
      "Size": 2048576,
      "ETag": "a3f2...",
      "LastModified": "2024-01-15T10:30:00Z",
      "Location": [...]
    },
    {
      "VersionId": "def456uvw",
      "IsLatest": false,
      "Size": 1800000,
      "ETag": "b8d4...",
      "LastModified": "2024-01-10T14:20:00Z",
      "Location": [...]
    },
    ...
  ]
```

**Metadata overhead**:

```
Per version: ~500 bytes
1000 versions: 500 KB just for metadata

For 1 trillion objects, avg 10 versions each:
  Metadata: 10 trillion entries × 500 bytes = 5 PB
  (vs 1 PB without versioning)

Challenge: Metadata doesn't scale linearly!
```

### Listing Versions

**List versions API**:

```
GET /bucket?versions&prefix=photos/

Returns:
  - All objects with prefix "photos/"
  - All versions of each object
  - Delete markers
  - Paginated (1000 at a time)

Sorted by:
  1. Key (alphabetical)
  2. Version (newest first per key)
```

**Performance**:

```
1 million objects, 10 versions each = 10 million versions

List first 1000:
  - Scan metadata index
  - Filter by prefix
  - Sort by key + version
  - Return first 1000

Time: 100-500ms (depending on index efficiency)
```

## Use Cases for Versioning

### Use Case 1: Accidental Delete Protection

**Scenario**: User deletes important file.

```
Without versioning:
  User: DELETE /important-doc.pdf
  → Gone forever
  → User: "Help! I didn't mean to delete it!"
  → Support: "Sorry, it's gone."

With versioning:
  User: DELETE /important-doc.pdf
  → Delete marker created
  → User: "Help! I didn't mean to delete it!"
  → Support: "No problem, restoring..."
  → DELETE /important-doc.pdf?versionId=<delete-marker-id>
  → File restored!
```

### Use Case 2: Regulatory Compliance

**Requirement**: Keep all versions for 7 years.

```
Compliance rule:
  "All financial records must be retained for 7 years,
   including all modifications."

With versioning:
  - Every edit creates new version
  - All versions retained
  - Can prove compliance in audit

Lifecycle policy:
  - Delete versions older than 7 years
  - Automatically complies with retention policy
```

### Use Case 3: Rollback

**Scenario**: Bad deployment.

```
Application deployment:
  v1: PUT /config.json (working)
  v2: PUT /config.json (broken config!)
  → Application crashes

Rollback:
  Identify working version: v1
  Restore: Copy v1 to current
  Or: Update app to use ?versionId=v1
  → Application works again!
```

### Use Case 4: Audit Trail

**Track all changes**:

```
Document editing:
  v1: Original (2024-01-01)
  v2: User Alice edited (2024-01-05)
  v3: User Bob edited (2024-01-10)
  v4: User Alice edited again (2024-01-12)

Audit:
  Who changed what when?
  → List versions
  → See timestamp and size for each
  → Download specific version to compare
```

### Use Case 5: A/B Testing

**Test different file versions**:

```
Scenario: Test two versions of a web page

Upload:
  v1: PUT /index.html (control)
  v2: PUT /index.html (test variation)

Serve:
  50% of users: GET /index.html (latest = v2)
  50% of users: GET /index.html?versionId=v1 (control)

Measure which performs better.
```

## Implementation Challenges

### Challenge 1: Listing Performance

**Problem**: Listing versions is slow with many versions.

```
1 million objects, 100 versions each:
  Total entries: 100 million
  List all: Hours!

Solution: Pagination + prefix filtering
  List 1000 at a time
  Filter by prefix
  Only load metadata needed
```

### Challenge 2: Storage Efficiency

**Problem**: Small changes create full copies.

```
1 GB file, change 1 byte:
  v1: 1 GB
  v2: 1 GB (entire file copied!)
  Storage: 2 GB for 1 byte change

Inefficient!
```

**Possible optimization** (S3 doesn't do this, but could):

```
Delta storage:
  v1: 1 GB (full)
  v2: 1 byte delta from v1
  Storage: 1 GB + 1 byte

Complexity:
  ✗ Must reconstruct versions (slower reads)
  ✗ Delta computation (CPU cost)
  ✗ Can't easily delete old versions

S3 choice: Simple full copies (prioritize read speed)
```

### Challenge 3: Version ID Collision

**Problem**: Version IDs must be unique globally.

```
With random UUIDs:
  Probability of collision ≈ 0 (for practical purposes)
  UUID space: 2^122 (340 undecillion)

With timestamp-based:
  Two writes at exact same millisecond?
  → Add random suffix to guarantee uniqueness
```

**Generation**:

```
Version ID = timestamp + "-" + random(128 bits)

Example:
  "20240115103045123-a3f2b8d4c1e5f6a7"
  (millisecond precision + random suffix)

Collision probability: Negligible
```

### Challenge 4: Delete Marker Accumulation

**Problem**: Delete markers accumulate.

```
Scenario:
  Create file, delete, create, delete, ... (1000 times)

Result:
  1000 delete markers
  + 1000 versions
  = 2000 entries for one key!

Listing performance degrades.
```

**Solution**: Expired delete marker cleanup.

```
Lifecycle rule:
  "Delete delete markers with no non-delete versions"

Example:
  [DELETE MARKER] ← Only version
  → Can safely delete (no data to recover)
  → Reduces clutter
```

## Versioning vs Backups

**Versioning is NOT a backup**:

```
Versioning protects against:
  ✓ Accidental overwrites
  ✓ Accidental deletes
  ✓ Application bugs

Versioning does NOT protect against:
  ✗ Account compromise (attacker deletes all versions)
  ✗ Ransomware (encrypts all versions)
  ✗ Regional failures (if single region)

Need both:
  - Versioning: For quick recovery from mistakes
  - Backups: For disaster recovery
    (Cross-region replication, offline backups)
```

**Best practice**:

```
Defense in depth:
  1. Enable versioning (protect against mistakes)
  2. Cross-region replication (protect against regional failure)
  3. Periodic backups to separate account (protect against compromise)
  4. Lifecycle policies (manage costs)
  5. Object Lock (prevent deletion for compliance)
```

## Object Lock (MFA Delete)

**Prevent version deletion**:

```
Object Lock modes:

1. Governance mode:
   - Versions can't be deleted
   - Except by users with special permission
   - Use case: Prevent accidents

2. Compliance mode:
   - Versions can NEVER be deleted until retention period expires
   - Even root user can't delete!
   - Use case: Regulatory compliance

MFA Delete:
   - Require multi-factor auth to delete versions
   - Extra protection against compromise
```

**Configuration**:

```
Enable Object Lock on bucket creation:
  PUT /bucket?object-lock
  (Cannot be enabled later!)

Set retention:
  PUT /bucket/object?retention
  {
    "Mode": "COMPLIANCE",
    "RetainUntilDate": "2031-01-01"
  }

Try to delete before 2031:
  DELETE /bucket/object?versionId=X
  → 403 Forbidden (retention period not expired)
```

## Key Takeaways

- **Versioning keeps all versions** - Nothing is truly deleted
- **Delete markers enable soft delete** - Deletions are reversible
- **Version IDs are unique** - Timestamp + random for uniqueness
- **Storage costs multiply** - Each version consumes space
- **Lifecycle policies essential** - Manage version bloat
- **Not a replacement for backups** - Complement, don't replace
- **Object Lock for compliance** - Prevent deletion when required
- **Trade-off: durability vs cost** - Keep what you need, not everything

## What's Next?

Versioning helps manage individual object lifecycles. But what about managing billions of objects at scale? In the next chapter, we'll explore lifecycle policies and storage classes.

---

**[← Previous: Consistency Models](./11-consistency-models.md) | [Back to Contents](./README.md) | [Next: Lifecycle and Storage Classes →](./13-lifecycle-storage-classes.md)**
