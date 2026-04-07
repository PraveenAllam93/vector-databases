# 3. Race Conditions & Concurrency — The Hard Problems

---

## ELI5

Imagine two people editing the same Google Doc at the same time. If both change the same sentence simultaneously, whose version wins? That's a race condition.

In vector databases, this happens constantly: multiple services writing vectors, indexes being rebuilt while searches run, queries hitting stale data. Getting concurrency wrong means lost data, wrong search results, or crashes.

---

## Race Condition Categories in Vector DBs

```
┌──────────────────────────────────────────────┐
│  1. WRITE-WRITE CONFLICTS                     │
│     Two processes update the same vector       │
│                                               │
│  2. READ-WRITE CONFLICTS                      │
│     Search runs while data is being updated    │
│                                               │
│  3. INDEX-DATA SKEW                           │
│     Index doesn't match stored vectors         │
│                                               │
│  4. INGESTION RACES                           │
│     Multiple workers ingest overlapping data   │
│                                               │
│  5. DELETE-DURING-SEARCH                      │
│     Vector deleted while search is in progress │
│                                               │
│  6. SCHEMA CHANGE RACES                       │
│     Collection config changes during operations│
└──────────────────────────────────────────────┘
```

---

## 1. Write-Write Conflicts

### The Problem

```
Service A:  Read vector V1 → modify → write V1 back
Service B:  Read vector V1 → modify → write V1 back

Timeline:
  T1: Service A reads V1 = [0.1, 0.2, 0.3]
  T2: Service B reads V1 = [0.1, 0.2, 0.3]  (same version!)
  T3: Service A writes V1 = [0.4, 0.5, 0.6]  ← A's update
  T4: Service B writes V1 = [0.7, 0.8, 0.9]  ← B's update OVERWRITES A's!

Result: Service A's write is silently lost. This is called a "lost update."
```

### Solution 1: Optimistic Concurrency Control (OCC)

Use version numbers to detect conflicts:

```python
# Read with version
point = client.retrieve("docs", ids=[1])[0]
current_version = point.payload.get("version", 0)

# Update with version check
new_version = current_version + 1
new_payload = {**point.payload, "version": new_version}

# Conditional write: only succeeds if version hasn't changed
# Qdrant doesn't have native CAS, so implement at application level:
client.set_payload(
    collection_name="docs",
    payload={"version": new_version, "data": "new_data"},
    points=[1],
    # Check version in application logic
)
```

**Application-level OCC pattern:**

```python
import time

def safe_update(client, collection, point_id, update_fn, max_retries=5):
    """Update a point with optimistic concurrency control."""
    for attempt in range(max_retries):
        # Read current state
        points = client.retrieve(collection, ids=[point_id], with_payload=True)
        if not points:
            raise ValueError(f"Point {point_id} not found")

        current = points[0]
        current_version = current.payload.get("_version", 0)

        # Apply update
        new_payload = update_fn(current.payload)
        new_payload["_version"] = current_version + 1

        # Try to write
        # In a real system, use a distributed lock or CAS operation
        # For Qdrant, we check version after write
        client.set_payload(collection, payload=new_payload, points=[point_id])

        # Verify our write won (read-after-write check)
        updated = client.retrieve(collection, ids=[point_id], with_payload=True)
        if updated[0].payload.get("_version") == current_version + 1:
            return updated[0]  # Success!

        # Another writer beat us — retry
        time.sleep(0.01 * (2 ** attempt))  # exponential backoff

    raise Exception(f"Failed to update after {max_retries} retries (contention)")
```

### Solution 2: Write Queues (Serialization)

Route all writes through a single queue:

```
Service A ─┐
           ├──▶ [Kafka Queue] ──▶ [Single Writer] ──▶ Qdrant
Service B ─┘       (ordered)         (sequential)

No races: all writes are serialized through the queue.
The single writer processes them in order.
```

```python
# Producer (any service)
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode(),
)

def enqueue_write(vector_id, vector, payload):
    producer.send('vector-writes', {
        'operation': 'upsert',
        'id': vector_id,
        'vector': vector,
        'payload': payload,
        'timestamp': time.time(),
    })
    producer.flush()

# Consumer (single writer)
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'vector-writes',
    bootstrap_servers=['localhost:9092'],
    group_id='vector-writer',
    value_deserializer=lambda v: json.loads(v),
)

for message in consumer:
    op = message.value
    if op['operation'] == 'upsert':
        client.upsert(
            collection_name="docs",
            points=[PointStruct(id=op['id'], vector=op['vector'], payload=op['payload'])],
        )
    elif op['operation'] == 'delete':
        client.delete(collection_name="docs", points_selector=[op['id']])
```

### Solution 3: Idempotent Writes

Design writes so that applying them multiple times has the same effect as once:

```python
# BAD: Not idempotent — incrementing a counter
payload["view_count"] += 1  # If retried, increments twice!

# GOOD: Idempotent — setting an absolute value
payload["view_count"] = 150  # Same result no matter how many times applied

# GOOD: Idempotent — upsert with content-based ID
content_hash = hashlib.sha256(text.encode()).hexdigest()[:16]
client.upsert(
    collection_name="docs",
    points=[PointStruct(id=content_hash, vector=embedding, payload=payload)],
)
# Same content → same ID → same point → no duplicates
```

---

## 2. Read-Write Conflicts (Read-After-Write Inconsistency)

### The Problem

```
Timeline:
  T1: Writer updates vector V1 (new embedding)
  T2: Reader searches and gets V1 in results (OLD embedding used for similarity!)
  T3: Reader sees V1's payload is updated but the search SCORE was based on old data

  The search result includes V1, but the relevance score is wrong
  because the index hasn't been updated yet.
```

### Deeper: HNSW Index Lag

```
Vector storage: [V1_new, V2, V3, ...]     ← V1 updated immediately
HNSW index:     graph built on [V1_old, V2, V3, ...]  ← still references old V1

Search traverses the HNSW graph using V1_old's position
But returns V1_new's data when fetching the result

Result: V1 might appear in results when it shouldn't (or vice versa)
```

### Solutions

**1. Segment-based isolation (Qdrant's approach):**

```
Mutable segment:  New writes go here. Searched with flat scan (slow but current).
Sealed segments:  Immutable. HNSW index is built and accurate.

Write flow:
  1. New vector → mutable segment (instantly searchable via flat scan)
  2. Mutable segment reaches threshold → sealed
  3. HNSW index built on sealed segment (background)
  4. Once indexed, switch traffic to sealed segment

Result: New data is always searchable (via flat scan on mutable segment)
        while sealed segments have optimized HNSW indexes.
```

**2. Write-then-read pattern:**

```python
# After writing, wait for segment optimization before querying
client.upsert(collection_name="docs", points=[...])

# Option A: Wait for optimization (blocking)
client.update_collection(
    collection_name="docs",
    optimizer_config=OptimizersConfigDiff(
        flush_interval_sec=1,  # flush faster
    ),
)

# Option B: Rely on eventual consistency (non-blocking, acceptable for most cases)
# The mutable segment is flat-scanned, so new data IS searchable — just slightly slower
```

**3. Two-phase reads (guaranteed consistency):**

```python
def consistent_search(query_vector, collection, point_id_just_written=None):
    """Search that guarantees recently written data is included."""
    results = client.query_points(
        collection_name=collection,
        query=query_vector,
        limit=10,
    )

    result_ids = {p.id for p in results.points}

    # If we just wrote a point, check if it should be in results
    if point_id_just_written and point_id_just_written not in result_ids:
        # Explicitly fetch and check similarity
        point = client.retrieve(collection, ids=[point_id_just_written], with_vectors=True)
        if point:
            # Compute similarity manually and inject if relevant
            import numpy as np
            sim = np.dot(query_vector, point[0].vector) / (
                np.linalg.norm(query_vector) * np.linalg.norm(point[0].vector)
            )
            if sim > results.points[-1].score:  # better than worst result
                # Include it in results
                pass  # application logic to merge

    return results
```

---

## 3. Index-Data Skew

### The Problem

```
When the HNSW index was built:
  Vector V1 = [0.1, 0.2, 0.3]  ← V1's position in the graph

Current stored vector:
  Vector V1 = [0.9, 0.8, 0.7]  ← completely different!

The graph still has edges based on V1's OLD position.
Search might:
  - Not find V1 when it should (graph routes away from it)
  - Find V1 when it shouldn't (graph thinks it's in old neighborhood)
```

### Solutions

**1. Immutable segments (Qdrant):**
```
Vectors and their index are in the same sealed segment.
An update creates a NEW entry in the mutable segment.
The old entry is marked as deleted in the sealed segment.
During compaction, the old segment is rebuilt without deleted entries.

No skew possible: segment data and index are always consistent.
```

**2. Versioned embeddings:**
```python
# Store embedding model version with each vector
payload = {
    "text": "...",
    "embedding_model": "v2",
    "embedding_timestamp": "2024-06-01T10:00:00Z",
}

# During re-embedding migration:
# 1. Old vectors: embedding_model="v1"
# 2. New vectors: embedding_model="v2"
# 3. Search with v2 query embedding, filter to v2 vectors
# 4. After migration complete, delete v1 vectors
```

---

## 4. Ingestion Races

### The Problem

Multiple workers ingesting the same documents:

```
Worker A: Processing "docker_guide.md" → chunks 1-5
Worker B: Processing "docker_guide.md" → chunks 1-5 (duplicate!)

Result: 10 chunks instead of 5, duplicate search results
```

### Solutions

**1. Content-based deduplication:**
```python
# Same content → same hash → same point ID → upsert overwrites
chunk_id = hashlib.sha256(chunk_text.encode()).hexdigest()[:16]
# If both workers upsert the same chunk_id, the second is a no-op
```

**2. Distributed locking:**
```python
import redis

lock_client = redis.Redis()

def ingest_with_lock(doc_id: str, file_path: str):
    """Only one worker can ingest a document at a time."""
    lock = lock_client.lock(f"ingest:{doc_id}", timeout=300)

    if lock.acquire(blocking=True, blocking_timeout=10):
        try:
            # Check if already ingested
            count = qdrant_client.count(
                collection_name="docs",
                count_filter=Filter(
                    must=[FieldCondition(key="doc_id", match=MatchValue(value=doc_id))]
                ),
            )
            if count.count > 0:
                print(f"Already ingested: {doc_id}")
                return

            # Ingest
            pipeline.ingest_file(file_path, doc_id=doc_id)
        finally:
            lock.release()
    else:
        print(f"Another worker is ingesting {doc_id}, skipping")
```

**3. Claim-based partitioning:**
```python
# Each worker claims a subset of documents
# Worker 0 processes doc_ids where hash(doc_id) % num_workers == 0
# Worker 1 processes doc_ids where hash(doc_id) % num_workers == 1
# No overlap, no locks needed

def should_process(doc_id: str, worker_id: int, num_workers: int) -> bool:
    return hash(doc_id) % num_workers == worker_id
```

---

## 5. Delete-During-Search

### The Problem

```
T1: Search starts, traverses HNSW graph
T2: Vector V7 is on the search path
T3: Another thread deletes V7
T4: Search tries to read V7 → CRASH or wrong results
```

### Solutions

**1. Soft deletes (Qdrant's approach):**
```
Delete doesn't remove the vector immediately.
It marks the vector as "deleted" in a bitfield.
Search skips deleted vectors.
Background compaction actually removes them later.

Timeline:
  T1: Search starts, V7 is candidate
  T2: V7 marked as deleted (bitfield update, atomic)
  T3: Search checks V7 → sees deleted flag → skips it
  T4: Background: compaction removes V7 from segment

No crash, no inconsistency.
```

**2. MVCC (Multi-Version Concurrency Control):**
```
Each operation sees a consistent snapshot of the data.

Read (search) at snapshot_version=5:
  Sees all vectors committed at or before version 5
  Doesn't see vectors written at version 6+
  Doesn't see vectors deleted at version 6+

Write/Delete at version=6:
  Creates new version, doesn't affect version 5

Search and delete can run concurrently without interference.
```

---

## 6. Schema Change Races

### The Problem

```
T1: Collection has 768-dim vectors
T2: Admin starts migration to 1024-dim vectors
T3: Ingestion service is still writing 768-dim vectors
T4: Search queries expect 768-dim embeddings but some are 1024

Result: Dimension mismatch errors, wrong search results
```

### Solutions

**1. Blue-green collections:**
```python
# Phase 1: Create new collection
client.create_collection("docs_v2", vectors_config=VectorParams(size=1024, ...))

# Phase 2: Dual-write (write to both)
for point in new_data:
    client.upsert("docs_v1", points=[point_768d])
    client.upsert("docs_v2", points=[point_1024d])

# Phase 3: Backfill v2 with re-embedded v1 data
for batch in iter_all_points("docs_v1"):
    re_embedded = new_model.encode([p.payload["text"] for p in batch])
    client.upsert("docs_v2", points=[...])

# Phase 4: Switch reads to v2
ACTIVE_COLLECTION = "docs_v2"

# Phase 5: Stop writing to v1, delete it
client.delete_collection("docs_v1")
```

**2. Named vectors for gradual migration:**
```python
# Add a new named vector field without touching existing ones
# Old queries use "embedding_v1", new queries use "embedding_v2"
client.create_collection(
    collection_name="docs",
    vectors_config={
        "embedding_v1": VectorParams(size=768, distance=Distance.COSINE),
        "embedding_v2": VectorParams(size=1024, distance=Distance.COSINE),
    },
)
```

---

## Concurrency Patterns Summary

### Pattern: Write Queue + Read Replicas

```
┌─────────┐     ┌──────────┐     ┌──────────┐
│ Writers  │────▶│  Kafka   │────▶│  Single  │──── Qdrant Leader
│ (many)   │     │  Queue   │     │  Writer  │        │
└─────────┘     └──────────┘     └──────────┘        │
                                                 replication
┌─────────┐                                          │
│ Readers  │──── Qdrant Followers ◀──────────────────┘
│ (many)   │     (read replicas)
└─────────┘
```

**Properties:**
- Writes serialized through Kafka (no write-write conflicts)
- Reads from followers (horizontally scalable)
- Replication lag = time between write and searchable
- Exactly what your CLAUDE.md prescribes!

### Pattern: Optimistic Locking + Retry

```
Read current version
  ↓
Compute update
  ↓
Write with version check ──── Conflict? → Retry from start
  ↓
Success
```

### Pattern: Partition by Writer

```
Writer A owns: doc IDs hash(id) % 3 == 0
Writer B owns: doc IDs hash(id) % 3 == 1
Writer C owns: doc IDs hash(id) % 3 == 2

No two writers ever touch the same document → no conflicts by design
```

---

## Deadlocks

### Can Vector DBs Deadlock?

Traditional deadlock: Thread A holds Lock 1, waits for Lock 2. Thread B holds Lock 2, waits for Lock 1.

```
Vector DBs typically avoid deadlocks because:
  1. Writes are simple upserts (no complex transactions)
  2. No multi-collection transactions
  3. Internal operations use ordered locking (always acquire in same order)

BUT application-level deadlocks can happen:
  Service A: Lock("doc_1") → tries Lock("doc_2")
  Service B: Lock("doc_2") → tries Lock("doc_1")
  → DEADLOCK

Prevention: Always acquire locks in a consistent order (e.g., sorted by key)
```

```python
def update_multiple_docs(doc_ids: list[str]):
    """Acquire locks in sorted order to prevent deadlocks."""
    sorted_ids = sorted(doc_ids)
    locks = []

    try:
        for doc_id in sorted_ids:
            lock = redis_client.lock(f"doc:{doc_id}", timeout=30)
            if not lock.acquire(blocking_timeout=5):
                raise TimeoutError(f"Could not acquire lock for {doc_id}")
            locks.append(lock)

        # All locks acquired in order — safe to proceed
        for doc_id in doc_ids:
            process_document(doc_id)
    finally:
        for lock in reversed(locks):
            lock.release()
```

---

## Thundering Herd Problem

### The Problem

```
Cache expires for popular query:
  T1: 1000 concurrent users query "popular topic"
  T2: Cache miss! All 1000 queries hit Qdrant simultaneously
  T3: Qdrant is overwhelmed → high latency → potential crash
  T4: Each response tries to write to cache simultaneously
```

### Solutions

**1. Cache stampede lock:**
```python
def cached_search(query: str):
    # Try cache
    cached = redis.get(f"search:{query_hash}")
    if cached:
        return json.loads(cached)

    # Lock: only ONE process recomputes
    lock = redis.lock(f"search_lock:{query_hash}", timeout=10)
    if lock.acquire(blocking=True, blocking_timeout=2):
        try:
            # Double-check cache (another process may have filled it)
            cached = redis.get(f"search:{query_hash}")
            if cached:
                return json.loads(cached)

            # Compute and cache
            result = qdrant_search(query)
            redis.setex(f"search:{query_hash}", 3600, json.dumps(result))
            return result
        finally:
            lock.release()
    else:
        # Lock held by another process — wait and retry
        time.sleep(0.1)
        return cached_search(query)  # recursive retry
```

**2. Stale-while-revalidate:**
```python
def search_with_stale_while_revalidate(query: str):
    cached = redis.get(f"search:{query_hash}")
    ttl = redis.ttl(f"search:{query_hash}")

    if cached and ttl > 60:  # fresh
        return json.loads(cached)

    if cached and ttl <= 60:  # stale but usable
        # Return stale data immediately
        # Refresh in background
        background_refresh(query)
        return json.loads(cached)

    # No cache at all — must compute
    result = qdrant_search(query)
    redis.setex(f"search:{query_hash}", 3600, json.dumps(result))
    return result
```

---

## Summary

| Race Condition | Problem | Best Solution |
|---------------|---------|---------------|
| Write-Write | Lost updates | Write queue (Kafka) or idempotent upserts |
| Read-After-Write | Stale search results | Mutable segment flat scan + sticky sessions |
| Index-Data Skew | Index doesn't match vectors | Immutable segments (Qdrant's approach) |
| Ingestion Races | Duplicate chunks | Content-based IDs + distributed locks |
| Delete-During-Search | Crash or missing results | Soft deletes + MVCC |
| Schema Change | Dimension mismatch | Blue-green collections |
| Thundering Herd | Cache stampede | Lock + stale-while-revalidate |
| Deadlocks | Mutual lock waiting | Sorted lock acquisition order |

---

## Resources

- [Designing Data-Intensive Applications, Ch 7](https://dataintensive.net/) — Transactions and concurrency
- [Jepsen Analyses](https://jepsen.io/analyses) — real-world distributed system failure testing
- [Martin Fowler: Patterns of Distributed Systems](https://martinfowler.com/articles/patterns-of-distributed-systems/) — comprehensive patterns
- [Redis Distributed Locks (Redlock)](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/) — distributed locking with Redis
