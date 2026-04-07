# 7. System Design Interview Patterns — Putting It All Together

---

## How to Approach a Vector DB System Design Interview

### Framework

```
1. CLARIFY requirements (2-3 minutes)
   - Scale: How many vectors? QPS? Latency targets?
   - Consistency: Can we serve stale results? How stale?
   - Availability: What's the target? 99.9%? 99.99%?
   - Features: Filtering? Multi-tenancy? Real-time ingestion?

2. HIGH-LEVEL DESIGN (5-7 minutes)
   - Draw the architecture (components and data flow)
   - Explain why each component exists

3. DEEP DIVE into 2-3 areas (15-20 minutes)
   - Interviewer will pick what interests them
   - Show depth in distributed systems, scaling, or data integrity

4. TRADEOFFS and operational concerns (5 minutes)
   - What could go wrong?
   - How would you monitor this?
   - What would you do differently at 10x scale?
```

---

## Sample Question 1: "Design a Semantic Search System"

### Requirements

```
- 50M documents, growing 1M/month
- 768-dimension embeddings
- < 50ms p99 search latency
- 1000 QPS search, 100 QPS ingestion
- Metadata filtering (category, date, tenant)
- 99.95% availability
```

### High-Level Architecture

```
┌─────────┐     ┌─────────────┐     ┌──────────────┐
│  Client  │────▶│  API Gateway │────▶│  Search API   │
└─────────┘     │  (rate limit, │     │  (FastAPI)    │
                │   auth)       │     └──────┬───────┘
                └──────────────┘            │
                                   ┌────────┼────────┐
                                   ▼        ▼        ▼
                              ┌────────┐┌────────┐┌──────┐
                              │Qdrant  ││Qdrant  ││Redis │
                              │Cluster ││Cluster ││Cache │
                              │(search)││(search)││      │
                              └────────┘└────────┘└──────┘

┌─────────┐     ┌──────────┐     ┌──────────┐     ┌────────┐
│  Source  │────▶│  Kafka   │────▶│ Ingestion │────▶│ Qdrant │
│  (files, │     │  Queue   │     │ Workers   │     │        │
│   API)   │     └──────────┘     └──────────┘     └────────┘
```

### Component Justification

```
API Gateway:     Rate limiting, authentication, routing
Search API:      Embed query → search Qdrant → re-rank → return
Redis Cache:     Cache frequent queries (10x QPS improvement)
Kafka Queue:     Decouple ingestion from storage, provide back-pressure
Ingestion Workers: Chunk → embed → upsert (scalable independently)
Qdrant Cluster:  3 nodes, 6 shards, replication factor 2
```

### Capacity Math

```
Storage:
  50M vectors × 768 dims × 4 bytes = ~153 GB vectors
  HNSW overhead (~50%): ~77 GB
  Payloads (~500 bytes each): ~25 GB
  Total: ~255 GB

  With scalar quantization: ~77 GB vectors + 77 GB graph + 25 GB payloads = ~179 GB
  Per node (3 nodes, replication factor 2): ~120 GB each

RAM per node: 128 GB (120 GB data + headroom)
Disk per node: 500 GB (data + snapshots + compaction headroom)

QPS:
  1000 QPS / 3 nodes = ~333 QPS per node (well within single-node capacity)
  With cache hit rate of 30%: effective 700 QPS to Qdrant → easy
```

### Deep Dive: Scaling Plan

```
Year 1 (50M vectors):
  3 Qdrant nodes, 6 shards, replication=2
  Scalar quantization
  128GB RAM per node

Year 2 (62M vectors):
  Same cluster, within capacity

Year 3 (74M vectors):
  Add 4th node, rebalance shards
  Or: increase to binary quantization if model supports it

Year 5 (100M+ vectors):
  5-6 nodes, 12 shards
  Consider splitting query/ingestion traffic
```

---

## Sample Question 2: "Design a Multi-Tenant RAG Platform"

### Requirements

```
- 1000 tenants, each with 10K-10M documents
- Total: ~500M vectors
- Tenant isolation (MUST NOT leak data)
- Per-tenant QPS varies wildly (1-500 QPS)
- 99.99% availability
- Each tenant can use different embedding models
```

### Architecture Decisions

```
Tenancy model: Shared cluster with custom sharding
  Why not separate clusters? Too expensive at 1000 tenants
  Why not single collection? No isolation, noisy neighbors

Approach: Qdrant custom shard keys
  Each tenant gets their own shard key
  Shard keys are isolated → no cross-tenant access
  Large tenants get multiple shards
  Small tenants share physical nodes
```

```
┌──────────────────────────────────────────────────────┐
│  Qdrant Cluster (10 nodes)                            │
│                                                       │
│  Collection: "documents"                              │
│  Sharding: CUSTOM (by tenant_id)                      │
│                                                       │
│  Tenant "acme" → Shard Key "acme" → Nodes [1, 4, 7] │
│  Tenant "globex" → Shard Key "globex" → Nodes [2, 5, 8]│
│  Tenant "initech" → Shard Key "initech" → Nodes [3, 6, 9]│
│  ...                                                  │
└──────────────────────────────────────────────────────┘
```

### Tenant Isolation Enforcement

```python
class TenantSearchService:
    def search(self, tenant_id: str, query: str, top_k: int = 10):
        query_vec = self.get_tenant_embedder(tenant_id).embed(query)

        # CRITICAL: Always include tenant shard key
        results = self.client.query_points(
            collection_name="documents",
            query=query_vec,
            shard_key_selector=tenant_id,   # Only search this tenant's shard
            limit=top_k,
        )
        return results

    def get_tenant_embedder(self, tenant_id: str):
        """Each tenant can use a different embedding model."""
        config = self.tenant_configs[tenant_id]
        return self.embedders[config["embedding_model"]]
```

### Noisy Neighbor Prevention

```
Problem: Tenant "acme" sends 500 QPS → starves other tenants

Solution 1: Per-tenant rate limiting
  Each tenant has a QPS quota based on their plan
  Free: 10 QPS, Pro: 100 QPS, Enterprise: 1000 QPS

Solution 2: Priority queues
  High-priority tenants get dedicated search threads
  Low-priority queries wait during congestion

Solution 3: Separate pools
  Enterprise tenants get dedicated Qdrant nodes
  Shared tenants use a shared pool with rate limits
```

---

## Sample Question 3: "Design a Real-Time Vector Search with Sub-10ms Latency"

### Requirements

```
- 10M vectors, 384 dimensions
- < 10ms p99 latency
- 5000 QPS
- Results must include vectors added in last 1 second
```

### Key Design Decisions

```
1. Single-region, multi-AZ (no cross-region latency)
2. All data in RAM (no mmap for this latency target)
3. Binary quantization for fastest search
4. No re-ranking (adds 50-200ms, breaks latency budget)
5. gRPC for client-server communication (faster than REST)
6. Connection pooling (avoid TCP handshake per request)
```

```
Latency budget:
  Network (client → Qdrant): ~1ms
  Deserialization:            ~0.5ms
  HNSW search (BQ, 10M):     ~2-4ms
  Payload fetch:              ~1ms
  Serialization:              ~0.5ms
  Network (Qdrant → client):  ~1ms
  ─────────────────────────────────
  Total:                      ~6-8ms ✓ (under 10ms)
```

### Achieving 1-Second Data Freshness

```
Problem: HNSW index build takes seconds. Can't wait for sealed segments.

Solution: Qdrant's mutable segment
  New vectors go to mutable segment immediately
  Mutable segment uses flat scan (no HNSW needed)
  For 10M total vectors with <1000 in mutable → flat scan adds ~0.1ms

  New vector is searchable within:
    WAL write: ~1ms
    Mutable segment append: ~1ms
    Total: ~2ms from write to searchable ✓

  The mutable segment is sealed periodically and HNSW is built in background.
  This doesn't affect latency because sealed segments already have HNSW.
```

---

## Common Interview Questions & Model Answers

### "How would you handle a 10x traffic spike?"

```
Immediate (minutes):
  - Redis cache absorbs repeated queries
  - Auto-scaling adds API pods (HPA)
  - Rate limiting protects Qdrant from overload

Short-term (hours):
  - Add read replicas to Qdrant (if pre-provisioned)
  - Increase cache TTL to reduce Qdrant load
  - Enable more aggressive quantization

Long-term (days):
  - Add more Qdrant nodes and rebalance shards
  - Optimize hot queries (pre-compute, dedicated cache)
  - Consider CDN for static/common queries
```

### "What happens when the embedding model is updated?"

```
1. Deploy new model alongside old model
2. Create new collection (or use named vectors)
3. Dual-write: new ingestions go to both old and new
4. Background job: re-embed all existing documents
5. Run evaluation: compare recall@K on test queries
6. If quality improves: switch query traffic to new collection
7. Decommission old collection

Timeline: 1-7 days depending on corpus size
Key: Never switch without validating quality first
```

### "How do you debug poor search quality?"

```
Step 1: Check the query
  - Is the query embedding correct? (embed and inspect)
  - Is the query too vague? (try more specific)

Step 2: Check retrieval
  - Are relevant documents in the collection? (search by ID)
  - What's the recall@K? (compare against ground truth)
  - Are filters too restrictive? (try without filters)

Step 3: Check the index
  - Is efSearch too low? (increase and compare)
  - Is the collection optimized? (check segment status)
  - Are vectors normalized? (check magnitudes)

Step 4: Check the data
  - Are embeddings from the correct model version?
  - Are chunks too big/too small?
  - Is there data staleness? (check replication lag)

Step 5: Check re-ranking
  - Is the cross-encoder model appropriate for this domain?
  - Are enough candidates being passed to re-ranker?
```

### "How do you prevent data leaks in multi-tenant?"

```
Defense in depth:
  1. Shard-level isolation: Custom shard keys per tenant
  2. Query-level: EVERY query MUST include tenant_id filter
  3. API-level: Extract tenant_id from JWT, inject into query (never trust client)
  4. Audit: Log all queries with tenant context
  5. Testing: Automated tests that try cross-tenant access (must fail)

# API middleware example
@app.middleware("http")
async def inject_tenant(request, call_next):
    tenant_id = extract_tenant_from_jwt(request.headers["Authorization"])
    request.state.tenant_id = tenant_id
    response = await call_next(request)
    return response

# Every search endpoint
@app.post("/search")
async def search(query: SearchQuery, request: Request):
    tenant_id = request.state.tenant_id  # from middleware, not from client
    results = retriever.search(
        query=query.text,
        shard_key=tenant_id,
        filters={"tenant_id": tenant_id},  # belt AND suspenders
    )
```

### "Walk me through a write path, end to end."

```
1. Client sends POST /ingest with document text
2. API validates request, extracts tenant_id from JWT
3. API publishes to Kafka topic "vector-ingestion" with tenant_id
4. Ingestion worker consumes message:
   a. Parse and clean text
   b. Chunk (recursive split, 400 tokens, 80 overlap)
   c. Generate deterministic chunk IDs (content hash)
   d. Embed chunks (batch, with retry)
   e. Attach metadata (doc_id, chunk_index, embedding_version, timestamp)
5. Worker upserts to Qdrant:
   a. Qdrant receives on leader for the shard
   b. Leader writes to WAL (fsync)
   c. Leader replicates to follower(s)
   d. Majority ACK → write confirmed
   e. Data in mutable segment (immediately searchable)
6. Background: mutable segment sealed → HNSW index built
7. Worker ACKs Kafka message (at-least-once delivery)

Failure at any step:
  Step 3 fails: API retries to Kafka (Kafka is durable)
  Step 4 fails: Worker retries (idempotent, content-hash IDs)
  Step 5 fails: Worker retries upsert (Qdrant upsert is idempotent)
  Qdrant crash: WAL replayed on restart
```

---

## Architecture Decision Records

Use this template to document why you made each decision:

```markdown
## ADR-001: Qdrant over Milvus for Vector Storage

**Status:** Accepted
**Date:** 2024-06-01

**Context:** We need a vector database for 50M vectors with rich filtering.

**Decision:** Use Qdrant.

**Reasons:**
- Inline filtering during HNSW search (Milvus does post-filter)
- Simpler operations (single binary, no Zookeeper/etcd dependencies)
- Better documented Python client
- Custom sharding for multi-tenancy

**Tradeoffs:**
- Milvus has better GPU support (not needed for our scale)
- Milvus has more index types (HNSW is sufficient for us)

**Consequences:**
- Team needs to learn Qdrant API
- Deployment via Docker/K8s (Qdrant has official Helm chart)
```

---

## Summary Cheat Sheet

```
SCALING:
  < 10M vectors    → single node, vertical scaling
  10M-100M vectors → 3-5 nodes, sharded + replicated
  100M+ vectors    → 10+ nodes, multi-region

CONSISTENCY:
  Reads:  eventual consistency is fine (stale search results acceptable)
  Writes: semi-synchronous (majority quorum)

AVAILABILITY:
  99.9%:  3 nodes, single region, multi-AZ
  99.99%: 3+ nodes, multi-region active-passive
  99.999%: multi-region active-active

LATENCY:
  < 10ms:  all in RAM, binary quantization, no re-ranking
  < 50ms:  scalar quantization + rescore, optional re-ranking
  < 200ms: re-ranking with cross-encoder, full RAG pipeline

DATA SAFETY:
  WAL for crash recovery
  Snapshots daily to S3
  Replication factor >= 2
  Embedding versioning in payloads

MULTI-TENANCY:
  Custom shard keys per tenant
  Mandatory tenant_id filter on every query
  Extract tenant from JWT (never trust client input)
  Per-tenant rate limiting
```

---

## Resources

- [System Design Interview (Alex Xu)](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF) — the classic book
- [ByteByteGo](https://bytebytego.com/) — visual system design
- [Designing Data-Intensive Applications](https://dataintensive.net/) — the bible for distributed systems
- [Qdrant Architecture](https://qdrant.tech/documentation/overview/) — understand the internals
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/) — free, comprehensive reliability guide
