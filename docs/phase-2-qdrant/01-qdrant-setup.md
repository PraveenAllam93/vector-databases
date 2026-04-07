# 1. Qdrant Setup — From Zero to Running

---

## ELI5

Qdrant is like a supercharged filing cabinet that understands meaning. You put documents in, and instead of searching by exact labels, you ask "find me things similar to THIS" — and it does it in milliseconds, even with millions of documents.

We'll run it in Docker — a container that packages Qdrant with everything it needs, so it works the same on your Mac, Linux, or in the cloud.

---

## Why Qdrant?

| Feature | Why It Matters |
|---------|---------------|
| Written in Rust | Fast, memory-safe, no garbage collection pauses |
| HNSW + quantization | Best-in-class search speed |
| Rich filtering | Filter by metadata DURING search (not after) |
| Payload indexing | Indexes metadata fields for fast filtering |
| gRPC + REST | Both high-performance and easy-to-use APIs |
| Snapshots | Built-in backup/restore |
| Distributed mode | Sharding + replication for production scale |
| Open source | Apache 2.0, no vendor lock-in |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                   Qdrant Node                    │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ REST API  │  │ gRPC API │  │  Web UI      │  │
│  │ :6333     │  │ :6334    │  │  :6333/dashboard│
│  └─────┬─────┘  └─────┬────┘  └──────────────┘  │
│        │               │                         │
│  ┌─────▼───────────────▼─────┐                   │
│  │     Collection Manager     │                   │
│  │  (create, delete, update)  │                   │
│  └─────────────┬─────────────┘                   │
│                │                                  │
│  ┌─────────────▼─────────────┐                   │
│  │      Search Engine         │                   │
│  │  ┌─────┐  ┌────────────┐  │                   │
│  │  │HNSW │  │  Payload   │  │                   │
│  │  │Index│  │  Filters   │  │                   │
│  │  └─────┘  └────────────┘  │                   │
│  └─────────────┬─────────────┘                   │
│                │                                  │
│  ┌─────────────▼─────────────┐                   │
│  │     Storage Engine         │                   │
│  │  ┌─────┐  ┌────────────┐  │                   │
│  │  │ WAL │  │  Segments   │  │                   │
│  │  └─────┘  └────────────┘  │                   │
│  └───────────────────────────┘                   │
│                                                  │
│  📁 /qdrant/storage (persistent volume)          │
└─────────────────────────────────────────────────┘
```

**Ports:**
- `6333` — REST API + Web Dashboard
- `6334` — gRPC API (faster, used by client libraries)

---

## Setup with Docker

### Option 1: Docker Run (Quick Start)

```bash
# Pull and run Qdrant
docker run -d \
  --name qdrant \
  -p 6333:6333 \
  -p 6334:6334 \
  -v $(pwd)/qdrant_storage:/qdrant/storage:z \
  qdrant/qdrant
```

Verify it's running:

```bash
# Health check
curl http://localhost:6333/healthz
# Should return: {"title":"qdrant - vectorass engine","version":"..."}

# Or open the dashboard
open http://localhost:6333/dashboard
```

### Option 2: Docker Compose (Recommended)

Create `docker-compose.yml` in project root:

```yaml
services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    ports:
      - "6333:6333"  # REST API + Dashboard
      - "6334:6334"  # gRPC
    volumes:
      - ./qdrant_storage:/qdrant/storage:z
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
    restart: unless-stopped
```

```bash
# Start
docker compose up -d

# Check logs
docker compose logs -f qdrant

# Stop
docker compose down

# Stop and delete data
docker compose down -v
```

### Option 3: Docker Compose with Custom Config

For production-like settings, add a config file:

```yaml
# config/qdrant_config.yaml
storage:
  storage_path: /qdrant/storage
  snapshots_path: /qdrant/snapshots

  # Performance tuning
  optimizers:
    default_segment_number: 2
    max_segment_size_kb: 200000
    memmap_threshold_kb: 50000
    indexing_threshold_kb: 20000
    flush_interval_sec: 5

  # WAL config
  wal:
    wal_capacity_mb: 32
    wal_segments_ahead: 0

service:
  max_request_size_mb: 32
  grpc_port: 6334
  enable_tls: false

# Telemetry (disable for privacy)
telemetry_disabled: true
```

```yaml
# docker-compose.yml (with config)
services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - ./qdrant_storage:/qdrant/storage:z
      - ./qdrant_snapshots:/qdrant/snapshots:z
      - ./config/qdrant_config.yaml:/qdrant/config/production.yaml
    restart: unless-stopped
```

---

## Python Client Setup

```bash
uv add qdrant-client
```

### Connection

```python
from qdrant_client import QdrantClient

# Local Docker
client = QdrantClient(host="localhost", port=6333)

# With gRPC (faster for bulk operations)
client = QdrantClient(host="localhost", port=6334, prefer_grpc=True)

# In-memory (for testing, no Docker needed)
client = QdrantClient(":memory:")

# Qdrant Cloud
client = QdrantClient(
    url="https://your-cluster.qdrant.io",
    api_key="your-api-key",
)
```

### Verify Connection

```python
# Get cluster info
info = client.get_collections()
print(f"Collections: {info.collections}")

# Health check
print(client.health_check())
```

---

## Qdrant Terminology

| Qdrant Term | Equivalent In | Description |
|-------------|--------------|-------------|
| Collection | Table (SQL) / Index (Elasticsearch) | A named group of vectors with the same dimensions |
| Point | Row (SQL) / Document (Elasticsearch) | A single vector + its ID + payload |
| Vector | Column value | The embedding (list of floats) |
| Payload | Metadata / Columns | JSON data attached to a point (filterable) |
| Segment | Partition | Internal storage unit (managed automatically) |
| Snapshot | Backup | Full copy of a collection for backup/restore |
| Shard | Partition | Horizontal split of data across nodes |
| Replica | Replica | Copy of a shard for redundancy |

### Visual

```
Collection: "articles"
├── Point {id: 1, vector: [0.1, 0.2, ...], payload: {title: "AI intro", category: "tech"}}
├── Point {id: 2, vector: [0.4, 0.5, ...], payload: {title: "Cooking 101", category: "food"}}
├── Point {id: 3, vector: [0.7, 0.8, ...], payload: {title: "ML guide", category: "tech"}}
└── ...
```

---

## Storage Internals

### Write-Ahead Log (WAL)

Every write goes through the WAL before being applied:

```
Client Write → WAL (disk) → Acknowledge → Apply to Segment (async)
```

If Qdrant crashes after WAL write but before segment update, it replays the WAL on restart. **No data loss**.

### Segments

Data is stored in segments:

```
Collection "articles"
├── Segment 0 (sealed, optimized)
│   ├── HNSW index (in memory or mmap)
│   ├── Vectors (mmap from disk)
│   └── Payloads (on disk)
├── Segment 1 (sealed, optimized)
│   ├── HNSW index
│   ├── Vectors
│   └── Payloads
└── Segment 2 (mutable, receiving writes)
    ├── Plain index (small, no HNSW yet)
    ├── Vectors
    └── Payloads
```

**Optimization process:**
1. New writes go to the mutable segment
2. When the mutable segment reaches a threshold, it's sealed
3. Sealed segments get an HNSW index built
4. Small segments are merged (compaction)

### Memory-Mapped Files (mmap)

Qdrant can memory-map vector data from disk instead of loading everything into RAM:

```
RAM usage:
  - HNSW graph: always in RAM (fast navigation)
  - Vectors: can be mmap'd (OS manages caching)
  - Payloads: on disk, loaded on demand

Benefit: Store 10x more vectors than RAM allows
Cost: Slightly slower (~2-3x) for vectors not in OS page cache
```

Configure per collection:

```python
from qdrant_client.models import OptimizersConfigDiff

client.update_collection(
    collection_name="articles",
    optimizer_config=OptimizersConfigDiff(
        memmap_threshold=20000  # mmap segments larger than 20K vectors
    ),
)
```

---

## Dashboard

Qdrant includes a built-in web UI at `http://localhost:6333/dashboard`:

- View all collections and their stats
- Browse points and payloads
- Run test queries
- Monitor performance

This is incredibly useful during development — always have it open.

---

## Common Docker Operations

```bash
# View logs
docker compose logs -f qdrant

# Restart Qdrant
docker compose restart qdrant

# Check resource usage
docker stats qdrant

# Shell into container
docker compose exec qdrant /bin/bash

# Backup storage directory
tar -czf qdrant_backup_$(date +%Y%m%d).tar.gz qdrant_storage/

# Upgrade Qdrant version
docker compose pull
docker compose up -d
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Docker Compose | Preferred way to run Qdrant locally |
| Ports | 6333 (REST + UI), 6334 (gRPC) |
| Persistent storage | Mount `./qdrant_storage` volume |
| WAL | Ensures no data loss on crash |
| Segments | Immutable chunks, auto-optimized with HNSW |
| mmap | Trade RAM for disk — store more vectors than fit in memory |
| Dashboard | `localhost:6333/dashboard` — use it constantly |

---

## Resources

- [Qdrant Quick Start](https://qdrant.tech/documentation/quick-start/) — official getting started
- [Qdrant Docker Guide](https://qdrant.tech/documentation/guides/installation/#docker) — all Docker options
- [Qdrant Configuration](https://qdrant.tech/documentation/guides/configuration/) — every config option explained
- [Qdrant Python Client](https://python-client.qdrant.tech/) — client library docs
