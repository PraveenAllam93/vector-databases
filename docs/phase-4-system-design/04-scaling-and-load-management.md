# 4. Scaling & Load Management — Handling Growth

---

## ELI5

Your pizza shop is getting popular. You can either buy bigger ovens (scale up) or open more locations (scale out). But just adding capacity isn't enough — you also need to manage the line (queues), turn away huge orders that would slow everyone down (rate limiting), and stop serving if the kitchen is on fire (circuit breakers).

---

## Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)

Add more resources to a single machine.

```
Before:  4 CPU, 16GB RAM → handles 500 QPS
After:   32 CPU, 128GB RAM → handles 3000 QPS

Vector DB impact:
  More RAM → more vectors in memory → faster search
  More CPU → more concurrent queries → higher throughput
  Faster SSD → faster mmap reads → lower p99 latency
```

| Pros | Cons |
|------|------|
| Simple (no distributed complexity) | Hardware ceiling (max ~128 cores, 2TB RAM) |
| No network latency between nodes | Single point of failure |
| All data local (fast access) | Expensive at top end |
| No sharding overhead | Downtime for upgrades |

**When to use**: Under ~50M vectors, single-digit millisecond latency requirement, team doesn't have distributed systems expertise.

### Horizontal Scaling (Scale Out)

Add more machines.

```
Before:  1 node → 500 QPS, 20M vectors
After:   4 nodes → 2000 QPS, 80M vectors

Vector DB impact:
  More nodes → more shards → higher throughput
  More replicas → fault tolerance + read scaling
  But: network latency, coordination overhead, operational complexity
```

| Pros | Cons |
|------|------|
| Unlimited scaling | Distributed systems complexity |
| Fault tolerant | Network latency between nodes |
| Cost-effective at scale | Operational overhead (monitoring, debugging) |
| Rolling upgrades (no downtime) | Scatter-gather overhead for searches |

**When to use**: Over ~50M vectors, need fault tolerance, need >1000 QPS, multi-region.

### Scaling Decision Matrix

```
Vectors     QPS        Availability    → Recommendation
< 10M       < 500      Best effort     → Single node, vertical
10M-50M     < 1000     99.9%           → Single node with replica
50M-500M    < 5000     99.95%          → 3-5 nodes, sharded + replicated
500M+       > 5000     99.99%          → 10+ nodes, multi-region
```

---

## Capacity Planning

### Memory Estimation

```python
def estimate_memory(
    num_vectors: int,
    dimensions: int,
    bytes_per_dim: int = 4,  # float32
    hnsw_overhead: float = 1.5,  # graph structure
    payload_avg_bytes: int = 500,
    quantization: str = "none",  # "none", "scalar", "binary"
) -> dict:
    """Estimate memory requirements."""
    # Vector storage
    vector_bytes = num_vectors * dimensions * bytes_per_dim

    # Quantization reduces vector memory
    if quantization == "scalar":
        vector_bytes = num_vectors * dimensions * 1  # int8 = 1 byte
    elif quantization == "binary":
        vector_bytes = num_vectors * dimensions // 8  # 1 bit per dim

    # HNSW graph overhead
    index_bytes = vector_bytes * (hnsw_overhead - 1)  # additional overhead

    # Payload storage
    payload_bytes = num_vectors * payload_avg_bytes

    # WAL buffer
    wal_bytes = 32 * 1024 * 1024  # 32MB typical

    total = vector_bytes + index_bytes + payload_bytes + wal_bytes

    return {
        "vectors_gb": vector_bytes / (1024**3),
        "index_gb": index_bytes / (1024**3),
        "payload_gb": payload_bytes / (1024**3),
        "total_gb": total / (1024**3),
        "recommended_ram_gb": total / (1024**3) * 1.3,  # 30% headroom
    }

# Examples
print("1M vectors, 768d, no quantization:")
print(estimate_memory(1_000_000, 768))
# → ~6.5 GB total, ~8.5 GB recommended RAM

print("\n10M vectors, 768d, scalar quantization:")
print(estimate_memory(10_000_000, 768, quantization="scalar"))
# → ~12 GB total, ~16 GB recommended RAM

print("\n100M vectors, 768d, scalar quantization:")
print(estimate_memory(100_000_000, 768, quantization="scalar"))
# → ~120 GB total → needs sharding across multiple nodes
```

### QPS Estimation

```
Single Qdrant node (typical):
  - Simple search (no filter): 1000-3000 QPS
  - Search with payload filter: 500-2000 QPS
  - Search with re-ranking: 200-500 QPS (bottleneck: cross-encoder)

Scaling:
  - Add read replicas: QPS scales linearly (3 replicas ≈ 3x QPS)
  - Add shards: marginal QPS improvement per shard (scatter-gather overhead)
  - Add caching: 10-100x QPS for repeated queries
```

### Disk Estimation

```
Base storage = vectors + payloads + WAL + snapshots

Vectors: num_vectors × dimensions × 4 bytes
Payloads: num_vectors × avg_payload_size
WAL: ~100MB-1GB depending on write rate
Snapshots: 1x full collection size per snapshot
HNSW index: ~0.5x vector size (if using mmap)

Rule: Provision 3x the estimated base storage
  1x for data
  1x for snapshots/backups
  1x for compaction headroom (temporary space during optimization)
```

---

## Load Balancing

### For Read Queries

```
                    ┌──────────────┐
Client requests ──▶ │ Load Balancer │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌────────┐  ┌────────┐  ┌────────┐
         │ Node 1 │  │ Node 2 │  │ Node 3 │
         │(replica)│  │(replica)│  │(replica)│
         └────────┘  └────────┘  └────────┘
```

**Strategies:**

```
Round-robin: Node1 → Node2 → Node3 → Node1 → ...
  Simple, even distribution, but ignores node load

Least-connections: Pick the node with fewest active queries
  Better under uneven load, requires tracking connections

Weighted: Node1 (powerful) gets 50%, Node2 gets 30%, Node3 gets 20%
  When nodes have different hardware specs

Latency-based: Pick the node that responded fastest recently
  Best for geo-distributed setups

Consistent hashing: Same query always goes to same node
  Useful for per-node query caching
```

### For Write Operations

```
All writes → Shard Leader

Router determines shard:
  shard_id = hash(vector_id) % num_shards
  route to shard_id's leader
```

---

## Back-Pressure

### What Is It?

When downstream can't keep up with upstream, back-pressure signals "slow down!"

```
Without back-pressure:
  Ingestion: 10,000 vectors/sec → Queue fills → OOM → crash

With back-pressure:
  Ingestion: 10,000 vectors/sec → Queue at 80% → slow producers to 5,000/sec → stable
```

### Implementation Patterns

**1. Bounded queue with blocking:**
```python
import asyncio

class BoundedIngestionQueue:
    def __init__(self, max_size: int = 10000):
        self.queue = asyncio.Queue(maxsize=max_size)

    async def enqueue(self, item):
        # Blocks if queue is full → back-pressure on producer
        await self.queue.put(item)

    async def process(self):
        while True:
            item = await self.queue.get()
            await ingest_to_qdrant(item)
            self.queue.task_done()
```

**2. Rate limiting at the API level:**
```python
from fastapi import FastAPI, Request, HTTPException
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()

@app.post("/ingest")
@limiter.limit("100/minute")  # max 100 ingestion requests per minute per IP
async def ingest(request: Request):
    ...

@app.post("/ask")
@limiter.limit("60/minute")   # max 60 queries per minute per IP
async def ask(request: Request):
    ...
```

**3. Adaptive batch sizing:**
```python
class AdaptiveIngester:
    def __init__(self):
        self.batch_size = 100
        self.min_batch = 10
        self.max_batch = 1000

    async def ingest_batch(self, items):
        start = time.time()
        # Ingest current batch
        client.upsert("docs", points=items[:self.batch_size])
        elapsed = time.time() - start

        # Adapt batch size based on latency
        if elapsed > 5.0:  # too slow
            self.batch_size = max(self.min_batch, self.batch_size // 2)
        elif elapsed < 1.0:  # room to grow
            self.batch_size = min(self.max_batch, self.batch_size * 2)
```

---

## Circuit Breaker

### The Problem

```
Qdrant is overloaded:
  API sends query → timeout (5s)
  API retries → timeout (5s)
  API retries → timeout (5s)
  ...
  Meanwhile: 1000 other requests pile up, each retrying
  Qdrant gets MORE overwhelmed → cascading failure
```

### The Solution

```
Circuit states:
  CLOSED (normal):    Requests flow through normally
  OPEN (tripped):     ALL requests fail immediately (no load on Qdrant)
  HALF-OPEN (testing): Allow ONE request through to test if Qdrant recovered

State transitions:
  CLOSED → OPEN:     After N consecutive failures
  OPEN → HALF-OPEN:  After cooldown period (e.g., 30 seconds)
  HALF-OPEN → CLOSED: If test request succeeds
  HALF-OPEN → OPEN:   If test request fails
```

```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 30,
        half_open_max_calls: int = 1,
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls

        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0
        self.half_open_calls = 0

    def can_execute(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True

        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
                return True
            return False

        if self.state == CircuitState.HALF_OPEN:
            return self.half_open_calls < self.half_open_max_calls

        return False

    def record_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED
        self.failure_count = 0

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
        elif self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

    def execute(self, func, *args, **kwargs):
        if not self.can_execute():
            raise Exception(f"Circuit OPEN — Qdrant unavailable, retry after {self.recovery_timeout}s")

        try:
            result = func(*args, **kwargs)
            self.record_success()
            return result
        except Exception as e:
            self.record_failure()
            raise


# Usage
qdrant_breaker = CircuitBreaker(failure_threshold=5, recovery_timeout=30)

def safe_search(query_vector):
    return qdrant_breaker.execute(
        client.query_points,
        collection_name="docs",
        query=query_vector,
        limit=10,
    )
```

---

## Health Checks & Readiness Probes

```python
from fastapi import FastAPI
from qdrant_client import QdrantClient

app = FastAPI()
client = QdrantClient(host="qdrant", port=6333)

@app.get("/health/live")
async def liveness():
    """Is the API process alive? (Kubernetes liveness probe)"""
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness():
    """Can the API serve requests? (Kubernetes readiness probe)"""
    checks = {}

    # Check Qdrant connectivity
    try:
        collections = client.get_collections()
        checks["qdrant"] = "ok"
    except Exception as e:
        checks["qdrant"] = f"error: {e}"

    # Check embedding model is loaded
    try:
        from services.embedder import embedder
        test = embedder.embed("test")
        checks["embedder"] = "ok"
    except Exception as e:
        checks["embedder"] = f"error: {e}"

    # Check LLM connectivity
    try:
        from openai import OpenAI
        llm = OpenAI()
        llm.models.list()
        checks["llm"] = "ok"
    except Exception as e:
        checks["llm"] = f"error: {e}"

    all_ok = all(v == "ok" for v in checks.values())
    status_code = 200 if all_ok else 503

    from fastapi.responses import JSONResponse
    return JSONResponse(
        content={"status": "ready" if all_ok else "not_ready", "checks": checks},
        status_code=status_code,
    )

@app.get("/health/metrics")
async def metrics():
    """Custom metrics for monitoring."""
    info = client.get_collection("documents")
    return {
        "collection_points": info.points_count,
        "collection_segments": info.segments_count,
        "collection_status": str(info.status),
    }
```

---

## Graceful Degradation

When parts of the system fail, degrade gracefully instead of failing completely:

```python
async def search_with_degradation(query: str, top_k: int = 5):
    """Search with multiple fallback levels."""

    # Level 1: Full pipeline (retrieve → re-rank → generate)
    try:
        chunks = retriever.search(query, top_k=top_k)
        if chunks:
            answer = generator.generate(query, chunks)
            return {"answer": answer, "sources": chunks, "level": "full"}
    except Exception:
        pass

    # Level 2: Retrieve only (skip re-ranking, skip LLM)
    try:
        query_vec = embedder.embed(query)
        results = client.query_points(
            collection_name="docs",
            query=query_vec,
            limit=top_k,
        )
        chunks = [{"text": p.payload["text"], "source": p.payload["source_doc"]} for p in results.points]
        return {"answer": "LLM unavailable. Here are relevant documents:", "sources": chunks, "level": "retrieval_only"}
    except Exception:
        pass

    # Level 3: Cache only
    try:
        cached = cache.get(query)
        if cached:
            return {**cached, "level": "cached", "warning": "Serving stale cached result"}
    except Exception:
        pass

    # Level 4: Complete failure
    return {
        "answer": "Service temporarily unavailable. Please try again.",
        "sources": [],
        "level": "unavailable",
    }
```

---

## Autoscaling

### Horizontal Pod Autoscaler (Kubernetes)

```yaml
# hpa.yaml — Scale API pods based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rag-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rag-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"  # Scale when >100 req/s per pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Wait 60s before scaling up
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60             # Add max 2 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120            # Remove max 1 pod per 2min
```

### Custom Metrics for Vector DB Scaling

```
Scale the QUERY service on:
  - QPS (queries per second)
  - p99 latency
  - Queue depth

Scale the INGESTION service on:
  - Ingestion queue depth
  - Processing lag

Scale Qdrant nodes on:
  - Memory utilization > 80%
  - CPU utilization > 70%
  - Replication lag > 1s
```

---

## Summary

| Concept | Key Point | When to Apply |
|---------|-----------|---------------|
| Vertical scaling | Bigger machine | < 50M vectors, simple setup |
| Horizontal scaling | More machines | > 50M vectors, need HA |
| Capacity planning | Estimate RAM, disk, QPS before deployment | Always, before going to production |
| Load balancing | Distribute reads across replicas | Multiple read replicas |
| Back-pressure | Slow producers when consumers are overwhelmed | High-throughput ingestion |
| Rate limiting | Cap requests per client | Public APIs |
| Circuit breaker | Fast-fail when downstream is dead | All external dependencies |
| Health checks | Liveness + readiness probes | Kubernetes deployments |
| Graceful degradation | Fallback levels when components fail | Production systems |
| Autoscaling | Scale pods/nodes based on metrics | Variable traffic patterns |

---

## Resources

- [Google SRE Book: Load Balancing](https://sre.google/sre-book/load-balancing-frontend/) — production load balancing
- [AWS Well-Architected: Scaling](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/scale-and-protect.html) — cloud scaling patterns
- [Circuit Breaker Pattern (Microsoft)](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) — comprehensive guide
- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) — autoscaling docs
