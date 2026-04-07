# 4. Vector Databases — Putting It All Together

---

## ELI5

A regular database is like a filing cabinet — you find things by their exact label ("Customer #12345").

A vector database is like a "find similar things" engine. You show it a photo and say "find me photos that look like this." It doesn't match exact labels — it matches **meaning**.

Under the hood, it stores embeddings (number lists) and uses the indexing algorithms we learned (HNSW, IVF, etc.) to search them fast.

---

## Why Not Just Use FAISS?

FAISS is an **index library**, not a database. Here's what a vector database adds:

| Feature | FAISS (Library) | Vector DB (Qdrant, Milvus, etc.) |
|---------|----------------|----------------------------------|
| Store vectors | In-memory only | Persistent on disk |
| Metadata filtering | No | Yes (`WHERE category = "tech"`) |
| CRUD operations | Add only (no update/delete) | Full CRUD |
| Replication | No | Leader-follower, multi-region |
| Sharding | Manual | Built-in |
| API access | Python only | REST/gRPC |
| Authentication | No | API keys, JWT |
| Backups | Manual | Automated snapshots |
| Concurrent access | Not thread-safe (mostly) | Multi-client safe |

**Think of it as**: FAISS = engine, Vector DB = the whole car.

---

## How a Vector Database Works Internally

```
Client Request
     │
     ▼
┌──────────┐     ┌─────────────────┐
│  API      │────▶│  Query Parser   │
│  Layer    │     │  (REST/gRPC)    │
└──────────┘     └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │  Filter Engine   │  ← metadata filtering
                 │  (pre/post)     │
                 └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │  ANN Index       │  ← HNSW / IVF / PQ
                 │  (in-memory)    │
                 └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │  Storage Engine  │  ← WAL, segments, compaction
                 │  (disk)         │
                 └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │  Results         │  ← ranked by similarity
                 └─────────────────┘
```

### Key Internal Components

**Write-Ahead Log (WAL)**: Every write is logged before being applied. If the server crashes, it replays the log to recover.

**Segments**: Vectors are stored in immutable segments. New data goes to a mutable segment, which is periodically "sealed" and merged (compaction).

**Index**: The ANN structure (HNSW graph, IVF clusters) is built on segments and lives in memory for fast access.

---

## The Major Vector Databases

### Chroma (Local Development)

```
Best for: Prototyping, learning, small projects
Deployment: In-process (Python), or standalone server
Max scale: ~100K vectors comfortably
License: Apache 2.0
```

```python
import chromadb

client = chromadb.Client()  # in-memory
# or
client = chromadb.PersistentClient(path="./chroma_data")  # persistent

collection = client.create_collection(
    name="documents",
    metadata={"hnsw:space": "cosine"}
)

# Add vectors with metadata
collection.add(
    ids=["doc1", "doc2", "doc3"],
    embeddings=[[0.1, 0.2, 0.3], [0.4, 0.5, 0.6], [0.7, 0.8, 0.9]],
    metadatas=[
        {"source": "wiki", "topic": "science"},
        {"source": "blog", "topic": "tech"},
        {"source": "wiki", "topic": "tech"},
    ],
    documents=["The earth orbits the sun", "Python is great", "AI is evolving"]
)

# Query
results = collection.query(
    query_embeddings=[[0.1, 0.2, 0.35]],
    n_results=2,
    where={"topic": "tech"}  # metadata filter
)
```

---

### Qdrant (Production — Recommended)

```
Best for: Mid-scale production, rich filtering
Deployment: Docker, Cloud, Kubernetes
Max scale: ~100M vectors per node
License: Apache 2.0
Standout: Advanced filtering, payload indexing, quantization
```

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

# Connect
client = QdrantClient(host="localhost", port=6333)

# Create collection
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
)

# Insert vectors (called "points" in Qdrant)
client.upsert(
    collection_name="documents",
    points=[
        PointStruct(id=1, vector=[0.1, 0.2, ...], payload={"source": "wiki", "topic": "science"}),
        PointStruct(id=2, vector=[0.4, 0.5, ...], payload={"source": "blog", "topic": "tech"}),
    ]
)

# Search with metadata filtering
results = client.query_points(
    collection_name="documents",
    query=[0.1, 0.2, ...],  # query vector
    query_filter={
        "must": [{"key": "topic", "match": {"value": "tech"}}]
    },
    limit=10
)
```

**Docker setup:**

```bash
docker run -p 6333:6333 -p 6334:6334 \
  -v $(pwd)/qdrant_data:/qdrant/storage \
  qdrant/qdrant
```

---

### Milvus (Large Scale)

```
Best for: Large-scale distributed deployments
Deployment: Kubernetes (cloud-native)
Max scale: Billions of vectors
License: Apache 2.0
Standout: Distributed architecture, GPU support, multiple index types
```

```python
from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType

connections.connect(host="localhost", port="19530")

fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=768),
    FieldSchema(name="topic", dtype=DataType.VARCHAR, max_length=64),
]
schema = CollectionSchema(fields)
collection = Collection("documents", schema)

# Insert
collection.insert([
    [[0.1, 0.2, ...], [0.4, 0.5, ...]],  # embeddings
    ["science", "tech"],                     # topics
])

# Create index
collection.create_index("embedding", {
    "index_type": "HNSW",
    "metric_type": "COSINE",
    "params": {"M": 32, "efConstruction": 200}
})

# Search
collection.search(
    data=[[0.1, 0.2, ...]],
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"ef": 100}},
    limit=10,
    expr='topic == "tech"'  # metadata filter
)
```

---

### FAISS (Research / Custom)

```
Best for: Research, custom pipelines, maximum control
Deployment: Library (Python/C++)
Max scale: Billions (with IVFPQ)
License: MIT
Standout: Most index types, GPU support, Facebook-backed
```

Not a database — a library. Use when you need full control and will handle persistence, filtering, and APIs yourself.

---

## Selection Guide

```
What's your scale?
  │
  ├─ Learning / Prototype (< 100K vectors)
  │   → Chroma
  │   → Zero config, Python-native
  │
  ├─ Production (100K - 100M vectors)
  │   → Qdrant
  │   → Rich filtering, easy Docker setup, great docs
  │
  ├─ Large scale (100M+ vectors)
  │   → Milvus
  │   → Distributed, GPU, Kubernetes-native
  │
  └─ Research / Custom pipeline
      → FAISS
      → Maximum control, build your own infra
```

---

## Metadata Filtering: Pre-filter vs Post-filter

This is a critical concept for production systems.

### Post-filter (Naive)

```
1. Find top 1000 similar vectors
2. Filter by metadata (topic = "tech")
3. Return top 10 from filtered results

Problem: If only 5 of the 1000 match the filter, you return only 5
         (asked for 10, got 5)
```

### Pre-filter (Better)

```
1. Build a bitmap of which vectors match the filter
2. Only search those vectors for similarity
3. Return top 10

Problem: If the filter is very selective (matches 0.1% of vectors),
         the HNSW graph becomes sparse and quality degrades
```

### Hybrid (Best — what Qdrant does)

```
1. Start HNSW search
2. At each step, check if the candidate passes the filter
3. If yes, keep it. If no, continue to next candidate
4. Stop when we have K passing results

Advantage: Uses the HNSW graph fully while respecting filters
```

---

## Data Operations

### Upsert (Insert or Update)

```python
# Qdrant upsert — if ID exists, update it; if not, insert it
client.upsert(
    collection_name="docs",
    points=[PointStruct(id=1, vector=[...], payload={...})]
)
```

**Why upsert matters**: Enables idempotent writes. If your ingestion pipeline retries (network error, crash), the same vector won't be duplicated.

### Delete

```python
# Qdrant delete by ID
client.delete(collection_name="docs", points_selector=[1, 2, 3])

# Delete by filter
client.delete(
    collection_name="docs",
    points_selector=FilterSelector(
        filter=Filter(must=[FieldCondition(key="source", match=MatchValue(value="old_source"))])
    )
)
```

### Batch Operations

Always use batch operations for bulk ingestion:

```python
# Bad — one at a time (slow, many round trips)
for doc in documents:
    client.upsert(collection_name="docs", points=[doc])

# Good — batch (fast, fewer round trips)
BATCH_SIZE = 100
for i in range(0, len(documents), BATCH_SIZE):
    batch = documents[i:i + BATCH_SIZE]
    client.upsert(collection_name="docs", points=batch)
```

---

## Collection Management

### Namespaces / Multi-tenancy

```python
# Option 1: Separate collections per tenant
client.create_collection("tenant_A_docs", ...)
client.create_collection("tenant_B_docs", ...)

# Option 2: Single collection with tenant filter (more efficient)
client.upsert(
    collection_name="docs",
    points=[
        PointStruct(id=1, vector=[...], payload={"tenant_id": "A", ...}),
        PointStruct(id=2, vector=[...], payload={"tenant_id": "B", ...}),
    ]
)

# Query with tenant isolation
results = client.query_points(
    collection_name="docs",
    query=[...],
    query_filter={"must": [{"key": "tenant_id", "match": {"value": "A"}}]},
    limit=10
)
```

### Snapshots & Backups

```python
# Qdrant snapshot
client.create_snapshot(collection_name="docs")

# List snapshots
snapshots = client.list_snapshots(collection_name="docs")
```

---

## Performance Numbers (Ballpark)

For 1M vectors, 768 dimensions, single node:

| Operation | Qdrant | Milvus | Chroma |
|-----------|--------|--------|--------|
| Insert (batch 1000) | ~50ms | ~100ms | ~200ms |
| Query (top 10) | ~2-5ms | ~5-10ms | ~10-50ms |
| Query with filter | ~3-8ms | ~8-15ms | ~20-100ms |
| Memory per 1M vectors | ~3-4 GB | ~4-6 GB | ~3-5 GB |

These vary wildly based on hardware, index config, and filter complexity. Always benchmark on YOUR data.

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Vector DB vs Library | DB adds persistence, filtering, CRUD, replication, APIs |
| Chroma | Best for local dev and learning |
| Qdrant | Best for mid-scale production |
| Milvus | Best for large-scale distributed |
| FAISS | Library, not DB — maximum control |
| Metadata filtering | Pre-filter, post-filter, or hybrid — affects result quality |
| Upsert | Idempotent writes — critical for reliability |
| Batch operations | Always batch inserts (100-1000 per batch) |
| Multi-tenancy | Use payload filters or separate collections |

---

## Resources

- [Qdrant Documentation](https://qdrant.tech/documentation/) — excellent getting-started guide
- [Milvus Documentation](https://milvus.io/docs) — distributed vector DB docs
- [Chroma Documentation](https://docs.trychroma.com/) — simplest getting started
- [Pinecone: What is a Vector Database?](https://www.pinecone.io/learn/vector-database/) — conceptual overview
- [Qdrant vs Milvus vs Weaviate (Benchmark)](https://qdrant.tech/benchmarks/) — performance comparisons
- [Vector Database Comparison (SuperLinked)](https://superlinked.com/vector-db-comparison/) — feature-by-feature comparison
