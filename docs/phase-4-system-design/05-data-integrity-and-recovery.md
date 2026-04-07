# 5. Data Integrity & Recovery — Never Lose Data

---

## ELI5

Imagine writing a diary. What if your computer crashes mid-sentence? You need:
- A way to recover what you already wrote (backups)
- A way to know where you stopped (write-ahead log)
- A way to check nothing got corrupted (checksums)
- A plan for when things go really wrong (disaster recovery)

---

## Write-Ahead Log (WAL) Deep Dive

### How WAL Works

Every write goes to the WAL BEFORE it's applied to the main data. If the system crashes, it replays the WAL to recover.

```
Write request arrives
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 1. Write to  │────▶│ 2. ACK to    │────▶│ 3. Apply to  │
│    WAL (disk) │     │    client    │     │    segment   │
│    (fsync)   │     │              │     │   (async)    │
└──────────────┘     └──────────────┘     └──────────────┘

If crash happens after step 1 but before step 3:
  → On restart, read WAL, replay unapplied operations
  → Data is recovered ✓

If crash happens before step 1:
  → Write is lost, but client never got ACK
  → Client retries → no data loss if writes are idempotent ✓
```

### WAL Structure

```
WAL file (sequential, append-only):
┌─────┬─────┬─────┬─────┬─────┬─────┐
│ Op1 │ Op2 │ Op3 │ Op4 │ Op5 │ ... │
└─────┴─────┴─────┴─────┴─────┴─────┘

Each operation entry:
┌──────────┬───────────┬──────────┬──────────────┬──────────┐
│ Sequence │ Operation │ Point ID │ Data         │ Checksum │
│ Number   │ Type      │          │ (vector+     │ (CRC32)  │
│ (uint64) │ (upsert/  │          │  payload)    │          │
│          │  delete)  │          │              │          │
└──────────┴───────────┴──────────┴──────────────┴──────────┘
```

**Key properties:**
- **Append-only**: Never modifies existing entries (fast, safe)
- **Sequential writes**: Much faster than random writes on disk
- **fsync**: Flush to disk (not just OS buffer) before acknowledging
- **Checksum**: Detect corruption from partial writes or disk errors

### WAL Configuration in Qdrant

```yaml
# qdrant config
storage:
  wal:
    wal_capacity_mb: 32      # Max WAL size before compaction
    wal_segments_ahead: 0    # Pre-allocated WAL segments
```

### WAL Compaction

The WAL can't grow forever. After operations are applied to segments, old WAL entries are removed:

```
Before compaction:
  WAL: [Op1, Op2, Op3, Op4, Op5, Op6, Op7, Op8, Op9, Op10]
  Applied to segments: Op1-Op7
  Not yet applied: Op8-Op10

After compaction:
  WAL: [Op8, Op9, Op10]   ← only unapplied ops kept
  Freed space: Op1-Op7 removed
```

---

## Snapshot Backups

### Full Snapshots

A complete copy of a collection at a point in time.

```python
# Create snapshot
snapshot = client.create_snapshot(collection_name="documents")
print(f"Snapshot created: {snapshot.name}")

# List snapshots
snapshots = client.list_snapshots(collection_name="documents")
for s in snapshots:
    print(f"  {s.name} — size: {s.size}, created: {s.creation_time}")

# Download snapshot (for off-site backup)
client.download_snapshot(
    collection_name="documents",
    snapshot_name=snapshot.name,
    path="./backups/documents_snapshot.tar",
)

# Restore from snapshot
client.recover_snapshot(
    collection_name="documents",
    location=f"file:///backups/documents_snapshot.tar",
)
```

### Full-System Snapshots

```python
# Snapshot ALL collections at once
snapshot = client.create_full_snapshot()
print(f"Full snapshot: {snapshot.name}")

# List full snapshots
snapshots = client.list_full_snapshots()
```

### Automated Backup Schedule

```python
"""
Automated backup script — run via cron or scheduled task.
"""
import os
import time
from datetime import datetime, timezone
from pathlib import Path
from qdrant_client import QdrantClient

client = QdrantClient(host="localhost", port=6333)
BACKUP_DIR = Path("./backups")
RETENTION_DAYS = 7

def backup_collection(collection_name: str):
    """Create and download a snapshot."""
    timestamp = datetime.now(timezone.utc).strftime("%Y%m%d_%H%M%S")
    backup_path = BACKUP_DIR / f"{collection_name}_{timestamp}.snapshot"

    # Create snapshot
    snapshot = client.create_snapshot(collection_name=collection_name)

    # Download
    client.download_snapshot(
        collection_name=collection_name,
        snapshot_name=snapshot.name,
        path=str(backup_path),
    )

    print(f"Backup created: {backup_path} ({backup_path.stat().st_size / 1024 / 1024:.1f} MB)")

    # Upload to S3 (production)
    # import boto3
    # s3 = boto3.client('s3')
    # s3.upload_file(str(backup_path), 'my-backups-bucket', f'qdrant/{backup_path.name}')

    return backup_path

def cleanup_old_backups():
    """Remove backups older than retention period."""
    now = time.time()
    for backup_file in BACKUP_DIR.glob("*.snapshot"):
        age_days = (now - backup_file.stat().st_mtime) / 86400
        if age_days > RETENTION_DAYS:
            backup_file.unlink()
            print(f"Deleted old backup: {backup_file.name}")

def run_backup():
    BACKUP_DIR.mkdir(exist_ok=True)
    collections = client.get_collections().collections

    for collection in collections:
        try:
            backup_collection(collection.name)
        except Exception as e:
            print(f"FAILED to backup {collection.name}: {e}")
            # In production: alert, retry, etc.

    cleanup_old_backups()

if __name__ == "__main__":
    run_backup()
```

### Backup Strategy for Production

```
┌─────────────────────────────────────────────────┐
│  Backup Schedule                                 │
│                                                  │
│  Continuous:  WAL provides point-in-time recovery │
│  Hourly:     Incremental snapshots (if supported) │
│  Daily:      Full snapshot → S3/GCS              │
│  Weekly:     Full snapshot → cross-region S3     │
│  Monthly:    Archive to Glacier/Coldline         │
│                                                  │
│  Retention:                                      │
│    Hourly:  48 hours                             │
│    Daily:   30 days                              │
│    Weekly:  12 weeks                             │
│    Monthly: 12 months                            │
└─────────────────────────────────────────────────┘
```

### Recovery Time vs Recovery Point

```
RPO (Recovery Point Objective):
  "How much data can we afford to lose?"
  → WAL: 0 data loss (replayed on restart)
  → Snapshots: data since last snapshot is lost
  → If daily snapshots: up to 24 hours of data loss

RTO (Recovery Time Objective):
  "How long can we be down?"
  → WAL replay: seconds to minutes
  → Snapshot restore: minutes to hours (depends on size)
  → Full re-ingestion: hours to days

Example targets:
  RPO: 1 hour (hourly snapshots + WAL)
  RTO: 15 minutes (pre-staged snapshot + fast restore)
```

---

## Data Corruption Detection

### Checksums

```
Every WAL entry and segment has a checksum:

Write:
  data = serialize(vector + payload)
  checksum = crc32(data)
  store(data + checksum)

Read:
  data, stored_checksum = load()
  computed_checksum = crc32(data)
  if computed_checksum != stored_checksum:
      CORRUPTION DETECTED → use replica or restore from backup
```

### Periodic Integrity Checks

```python
def verify_collection_integrity(client, collection_name: str):
    """Verify collection data integrity."""
    info = client.get_collection(collection_name)

    checks = {
        "status": str(info.status),
        "points_count": info.points_count,
        "vectors_count": info.vectors_count,
        "segments_count": info.segments_count,
    }

    # Points and vectors should match (unless using named vectors)
    if info.points_count != info.vectors_count:
        checks["warning"] = "Point count != vector count (possible corruption or named vectors)"

    # Check segment health
    # Qdrant exposes segment info via the API
    # Healthy segments should be in "green" or "yellow" status

    # Sample verification: read random points and check dimensions
    import random
    sample_ids = random.sample(range(1, info.points_count + 1), min(100, info.points_count))

    try:
        points = client.retrieve(
            collection_name, ids=sample_ids, with_vectors=True
        )
        expected_dim = info.config.params.vectors.size
        for point in points:
            if len(point.vector) != expected_dim:
                checks["corruption"] = f"Point {point.id} has wrong dimensions: {len(point.vector)} != {expected_dim}"
                break
        else:
            checks["vector_integrity"] = "ok"
    except Exception as e:
        checks["error"] = str(e)

    return checks
```

---

## Disaster Recovery Scenarios

### Scenario 1: Single Node Failure

```
Problem: One of 3 nodes in a cluster crashes
Impact: Some shards lose a replica
RTO: Seconds (automatic failover)

Recovery:
  1. Leader election for affected shards (automatic, ~5s)
  2. Remaining replicas serve traffic (no downtime)
  3. Replace failed node
  4. New node syncs data from leaders (background)
  5. Full redundancy restored

No manual intervention required.
```

### Scenario 2: Data Center Failure

```
Problem: Entire availability zone goes down
Impact: All nodes in that AZ are unavailable
RTO: Depends on multi-AZ setup

If multi-AZ deployed:
  1. Nodes in other AZs take over (automatic)
  2. Reduced capacity until AZ recovers
  3. When AZ recovers, nodes rejoin and sync

If single-AZ:
  1. Complete outage
  2. Wait for AZ recovery, or...
  3. Restore from off-site backups in another AZ
  4. RTO: hours (depending on data size)
```

### Scenario 3: Data Corruption (Silent)

```
Problem: Bad deployment writes incorrect embeddings for 2 days
Impact: Search quality degraded, but no errors
Detection: Monitoring recall@K metrics drops

Recovery:
  Option A: Restore snapshot from before corruption
    1. Identify when corruption started
    2. Restore snapshot from before that time
    3. Re-ingest data from the corruption period with correct embeddings
    4. RTO: hours, RPO: data since snapshot

  Option B: Selective re-embedding
    1. Query for points with embedding_model="bad_version"
    2. Re-embed those documents with correct model
    3. Upsert corrected vectors
    4. RTO: hours (depends on volume), RPO: 0 (fix in place)
```

### Scenario 4: Accidental Deletion

```
Problem: Someone runs client.delete_collection("production_docs")
Impact: All data gone

Prevention:
  1. RBAC: Only admins can delete collections
  2. Deletion protection: Require confirmation flag
  3. Soft delete: Mark as deleted, actually delete after 24h

Recovery:
  1. Restore from latest snapshot
  2. Re-ingest any data since the snapshot
```

---

## Versioning and Audit Trail

### Operation Versioning

```python
from datetime import datetime, timezone

def versioned_upsert(client, collection_name: str, points: list, operator: str):
    """Upsert with audit metadata."""
    for point in points:
        point.payload["_last_modified"] = datetime.now(timezone.utc).isoformat()
        point.payload["_modified_by"] = operator
        point.payload["_version"] = point.payload.get("_version", 0) + 1

    client.upsert(collection_name=collection_name, points=points)

    # Log to audit trail (separate system)
    audit_log.append({
        "operation": "upsert",
        "collection": collection_name,
        "point_ids": [p.id for p in points],
        "operator": operator,
        "timestamp": datetime.now(timezone.utc).isoformat(),
    })
```

### Embedding Version Tracking

```python
EMBEDDING_REGISTRY = {
    "v1": {"model": "all-MiniLM-L6-v2", "dim": 384, "date": "2024-01-01"},
    "v2": {"model": "BAAI/bge-small-en-v1.5", "dim": 384, "date": "2024-06-01"},
    "v3": {"model": "text-embedding-3-small", "dim": 1536, "date": "2024-09-01"},
}

CURRENT_VERSION = "v2"

# Every point stores its embedding version
payload = {
    "text": "...",
    "embedding_version": CURRENT_VERSION,
    "embedding_model": EMBEDDING_REGISTRY[CURRENT_VERSION]["model"],
}

# Migration: find and re-embed old versions
def find_outdated_vectors(client, collection_name: str, current_version: str):
    """Find vectors that need re-embedding."""
    from qdrant_client.models import Filter, FieldCondition, MatchExcept

    count = client.count(
        collection_name=collection_name,
        count_filter=Filter(
            must_not=[
                FieldCondition(
                    key="embedding_version",
                    match=MatchExcept(except_=[current_version]),
                ),
            ]
        ),
    )
    return count.count
```

---

## Production Data Integrity Checklist

```
□ WAL enabled and fsync'd (Qdrant default — don't disable)
□ Replication factor >= 2 for all production collections
□ Automated daily snapshots
□ Snapshots stored off-site (S3/GCS, different region)
□ Snapshot restore tested regularly (at least quarterly)
□ Checksums verified on backup files
□ Embedding version tracked in every payload
□ Audit trail for all write operations
□ Monitoring for:
  □ Replication lag
  □ Segment health
  □ Point count anomalies
  □ Search quality (recall@K) regressions
□ RBAC restricting delete operations
□ Runbook for each disaster recovery scenario
□ RPO and RTO defined and validated
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| WAL | Every write logged before applied; crash recovery via replay |
| Snapshots | Full collection backup; automate daily + upload to S3 |
| RPO | How much data loss is acceptable (WAL=0, snapshots=since last) |
| RTO | How long downtime is acceptable (failover=seconds, restore=minutes-hours) |
| Checksums | Detect corruption in WAL entries and segments |
| Versioning | Track embedding model version in every payload |
| Audit trail | Log who changed what and when |
| Disaster recovery | Plan for node failure, AZ failure, corruption, accidental deletion |

---

## Resources

- [Qdrant Snapshots](https://qdrant.tech/documentation/concepts/snapshots/) — backup/restore docs
- [PostgreSQL WAL Internals](https://www.postgresql.org/docs/current/wal-intro.html) — WAL concepts (applies broadly)
- [AWS Disaster Recovery Strategies](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html) — DR patterns
- [Designing Data-Intensive Applications, Ch 3](https://dataintensive.net/) — Storage and retrieval internals
