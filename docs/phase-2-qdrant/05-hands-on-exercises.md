# 5. Hands-On Exercises — Qdrant in Practice

---

## Prerequisites

```bash
# Start Qdrant
docker compose up -d

# Install dependencies
uv add qdrant-client sentence-transformers fastembed
```

Verify: `curl http://localhost:6333/healthz` and open `http://localhost:6333/dashboard`

---

## Exercise 1: Collection CRUD & Basic Search

**Goal**: Create a collection, insert points, search, update, delete.

```python
"""
Exercise 1: Qdrant basics — CRUD and search.
Run: uv run python exercises/06_qdrant_basics.py
"""
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct,
    Filter, FieldCondition, MatchValue, Range,
    PayloadSchemaType,
)
from sentence_transformers import SentenceTransformer

# Setup
client = QdrantClient(host="localhost", port=6333)
model = SentenceTransformer("all-MiniLM-L6-v2")  # 384 dims

COLLECTION = "exercise_articles"

# Clean up from previous runs
if client.collection_exists(COLLECTION):
    client.delete_collection(COLLECTION)

# --- CREATE COLLECTION ---
client.create_collection(
    collection_name=COLLECTION,
    vectors_config=VectorParams(size=384, distance=Distance.COSINE),
)
print(f"Created collection: {COLLECTION}")

# --- CREATE PAYLOAD INDEXES (do this before bulk insert) ---
client.create_payload_index(COLLECTION, "category", PayloadSchemaType.KEYWORD)
client.create_payload_index(COLLECTION, "year", PayloadSchemaType.INTEGER)
print("Created payload indexes")

# --- INSERT POINTS ---
articles = [
    {"id": 1, "text": "Introduction to machine learning and neural networks", "category": "ai", "year": 2024, "author": "Alice"},
    {"id": 2, "text": "Docker containers and Kubernetes orchestration", "category": "devops", "year": 2024, "author": "Bob"},
    {"id": 3, "text": "Building REST APIs with FastAPI and Python", "category": "backend", "year": 2023, "author": "Alice"},
    {"id": 4, "text": "Deep learning with transformers and attention mechanisms", "category": "ai", "year": 2024, "author": "Charlie"},
    {"id": 5, "text": "PostgreSQL performance tuning and query optimization", "category": "database", "year": 2023, "author": "Bob"},
    {"id": 6, "text": "Vector databases for similarity search at scale", "category": "database", "year": 2024, "author": "Alice"},
    {"id": 7, "text": "CI/CD pipelines with GitHub Actions", "category": "devops", "year": 2023, "author": "Charlie"},
    {"id": 8, "text": "Natural language processing with BERT and GPT", "category": "ai", "year": 2024, "author": "Alice"},
    {"id": 9, "text": "Redis caching strategies for high-traffic applications", "category": "database", "year": 2023, "author": "Bob"},
    {"id": 10, "text": "Microservices architecture patterns and best practices", "category": "backend", "year": 2024, "author": "Charlie"},
]

texts = [a["text"] for a in articles]
embeddings = model.encode(texts)

points = [
    PointStruct(
        id=a["id"],
        vector=emb.tolist(),
        payload={"text": a["text"], "category": a["category"], "year": a["year"], "author": a["author"]},
    )
    for a, emb in zip(articles, embeddings)
]

client.upsert(collection_name=COLLECTION, points=points)
print(f"Inserted {len(points)} points")

# --- BASIC SEARCH ---
query = "How do neural networks work?"
query_vector = model.encode(query).tolist()

print(f"\n--- Search: '{query}' ---")
results = client.query_points(
    collection_name=COLLECTION,
    query=query_vector,
    limit=3,
)
for p in results.points:
    print(f"  [{p.score:.4f}] {p.payload['text']}")

# --- FILTERED SEARCH ---
print(f"\n--- Search: '{query}' (category=ai only) ---")
results = client.query_points(
    collection_name=COLLECTION,
    query=query_vector,
    query_filter=Filter(
        must=[FieldCondition(key="category", match=MatchValue(value="ai"))],
    ),
    limit=3,
)
for p in results.points:
    print(f"  [{p.score:.4f}] [{p.payload['category']}] {p.payload['text']}")

# --- RANGE FILTER ---
print(f"\n--- Search: '{query}' (year >= 2024) ---")
results = client.query_points(
    collection_name=COLLECTION,
    query=query_vector,
    query_filter=Filter(
        must=[FieldCondition(key="year", range=Range(gte=2024))],
    ),
    limit=3,
)
for p in results.points:
    print(f"  [{p.score:.4f}] [{p.payload['year']}] {p.payload['text']}")

# --- UPDATE PAYLOAD ---
client.set_payload(
    collection_name=COLLECTION,
    payload={"views": 5000, "featured": True},
    points=[1],
)
updated = client.retrieve(COLLECTION, ids=[1])
print(f"\n--- Updated point 1 ---")
print(f"  Payload: {updated[0].payload}")

# --- DELETE ---
client.delete(collection_name=COLLECTION, points_selector=[10])
count = client.count(collection_name=COLLECTION)
print(f"\n--- After deleting point 10 ---")
print(f"  Remaining points: {count.count}")

# --- COLLECTION INFO ---
info = client.get_collection(COLLECTION)
print(f"\n--- Collection Info ---")
print(f"  Points: {info.points_count}")
print(f"  Segments: {info.segments_count}")
print(f"  Status: {info.status}")
```

---

## Exercise 2: Named Vectors — Multi-Aspect Search

**Goal**: Store title and content as separate vectors, search by either.

```python
"""
Exercise 2: Named vectors — search by title or content separately.
Run: uv run python exercises/07_named_vectors.py
"""
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct
from sentence_transformers import SentenceTransformer

client = QdrantClient(host="localhost", port=6333)
model = SentenceTransformer("all-MiniLM-L6-v2")

COLLECTION = "exercise_named_vectors"

if client.collection_exists(COLLECTION):
    client.delete_collection(COLLECTION)

# Collection with TWO vector types
client.create_collection(
    collection_name=COLLECTION,
    vectors_config={
        "title": VectorParams(size=384, distance=Distance.COSINE),
        "content": VectorParams(size=384, distance=Distance.COSINE),
    },
)

# Articles with different titles and content
articles = [
    {
        "id": 1,
        "title": "Getting Started with Docker",
        "content": "Docker is a platform for containerization. It packages applications with their dependencies into standardized units called containers. Containers are lightweight and share the host OS kernel.",
    },
    {
        "id": 2,
        "title": "Kubernetes in Production",
        "content": "Kubernetes (K8s) orchestrates container deployments. It handles scaling, load balancing, and self-healing. Production clusters typically use multi-node setups with separate control and data planes.",
    },
    {
        "id": 3,
        "title": "Python Web Frameworks",
        "content": "FastAPI is a modern Python web framework. It supports async, automatic OpenAPI docs, and type validation with Pydantic. Django is a batteries-included alternative for larger applications.",
    },
]

# Embed titles and content separately
points = []
for a in articles:
    title_vec = model.encode(a["title"]).tolist()
    content_vec = model.encode(a["content"]).tolist()
    points.append(PointStruct(
        id=a["id"],
        vector={"title": title_vec, "content": content_vec},
        payload={"title": a["title"], "content": a["content"]},
    ))

client.upsert(collection_name=COLLECTION, points=points)
print(f"Inserted {len(points)} points with named vectors\n")

# --- Search by TITLE similarity ---
query = "container management tools"
query_vec = model.encode(query).tolist()

print(f"--- Search by TITLE: '{query}' ---")
results = client.query_points(
    collection_name=COLLECTION,
    query=query_vec,
    using="title",  # search title vectors
    limit=3,
)
for p in results.points:
    print(f"  [{p.score:.4f}] {p.payload['title']}")

# --- Search by CONTENT similarity ---
print(f"\n--- Search by CONTENT: '{query}' ---")
results = client.query_points(
    collection_name=COLLECTION,
    query=query_vec,
    using="content",  # search content vectors
    limit=3,
)
for p in results.points:
    print(f"  [{p.score:.4f}] {p.payload['title']}")

# Notice: rankings may differ because titles and content
# capture different aspects of the same document
```

---

## Exercise 3: Hybrid Search (Dense + Sparse)

**Goal**: Combine semantic and keyword search using Qdrant's prefetch + fusion.

```python
"""
Exercise 3: Hybrid search with dense + sparse vectors.
Run: uv run python exercises/08_hybrid_search.py
"""
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, SparseVectorParams,
    PointStruct, SparseVector,
    Prefetch, FusionQuery, Fusion,
)
from fastembed import TextEmbedding, SparseTextEmbedding

client = QdrantClient(host="localhost", port=6333)

# Initialize embedding models
dense_model = TextEmbedding("BAAI/bge-small-en-v1.5")  # 384 dims
sparse_model = SparseTextEmbedding("prithivida/Splade_PP_en_v1")

COLLECTION = "exercise_hybrid"

if client.collection_exists(COLLECTION):
    client.delete_collection(COLLECTION)

# Collection with dense + sparse vectors
client.create_collection(
    collection_name=COLLECTION,
    vectors_config={
        "dense": VectorParams(size=384, distance=Distance.COSINE),
    },
    sparse_vectors_config={
        "sparse": SparseVectorParams(),
    },
)

# Documents — note some have specific technical terms
documents = [
    {"id": 1, "text": "Python error code E-4021 occurs when the authentication token expires during a session"},
    {"id": 2, "text": "How to handle authentication and authorization in web applications"},
    {"id": 3, "text": "Understanding JWT tokens and session management for secure APIs"},
    {"id": 4, "text": "Common Python exceptions and error handling best practices"},
    {"id": 5, "text": "OAuth2 flow implementation with refresh token rotation"},
    {"id": 6, "text": "Debugging runtime errors in production Python applications"},
    {"id": 7, "text": "CORS configuration and API security headers explained"},
    {"id": 8, "text": "Error code E-4021 fix: regenerate the API key in settings panel"},
]

# Generate embeddings
texts = [d["text"] for d in documents]
dense_embeddings = list(dense_model.embed(texts))
sparse_embeddings = list(sparse_model.embed(texts))

# Insert with both vector types
points = []
for doc, dense_emb, sparse_emb in zip(documents, dense_embeddings, sparse_embeddings):
    points.append(PointStruct(
        id=doc["id"],
        vector={
            "dense": dense_emb.tolist(),
            "sparse": SparseVector(
                indices=sparse_emb.indices.tolist(),
                values=sparse_emb.values.tolist(),
            ),
        },
        payload={"text": doc["text"]},
    ))

client.upsert(collection_name=COLLECTION, points=points)
print(f"Inserted {len(points)} hybrid points\n")

# --- Test queries ---
queries = [
    "error code E-4021",           # Keyword-heavy (sparse should help)
    "how to secure my web app",     # Semantic (dense should help)
    "E-4021 authentication fix",    # Both (hybrid shines)
]

for query in queries:
    print(f"=== Query: '{query}' ===\n")

    # Generate query embeddings
    q_dense = list(dense_model.embed([query]))[0]
    q_sparse = list(sparse_model.embed([query]))[0]

    # Dense only
    print("  Dense (semantic) only:")
    results = client.query_points(
        collection_name=COLLECTION,
        query=q_dense.tolist(),
        using="dense",
        limit=3,
    )
    for p in results.points:
        print(f"    [{p.score:.4f}] {p.payload['text'][:80]}")

    # Sparse only
    print("\n  Sparse (keyword) only:")
    results = client.query_points(
        collection_name=COLLECTION,
        query=SparseVector(
            indices=q_sparse.indices.tolist(),
            values=q_sparse.values.tolist(),
        ),
        using="sparse",
        limit=3,
    )
    for p in results.points:
        print(f"    [{p.score:.4f}] {p.payload['text'][:80]}")

    # Hybrid (RRF fusion)
    print("\n  Hybrid (RRF fusion):")
    results = client.query_points(
        collection_name=COLLECTION,
        prefetch=[
            Prefetch(query=q_dense.tolist(), using="dense", limit=5),
            Prefetch(
                query=SparseVector(
                    indices=q_sparse.indices.tolist(),
                    values=q_sparse.values.tolist(),
                ),
                using="sparse",
                limit=5,
            ),
        ],
        query=FusionQuery(fusion=Fusion.RRF),
        limit=3,
    )
    for p in results.points:
        print(f"    [{p.score:.4f}] {p.payload['text'][:80]}")

    print()
```

**What to observe**:
- "error code E-4021" → sparse finds exact match, dense returns general error docs
- "how to secure my web app" → dense understands semantics, sparse misses synonym matches
- Hybrid gets the best of both for all queries

---

## Exercise 4: Quantization Benchmark

**Goal**: Compare search accuracy and speed with different quantization settings.

```python
"""
Exercise 4: Quantization — measure memory and accuracy impact.
Run: uv run python exercises/09_quantization_bench.py
"""
import time
import numpy as np
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct,
    ScalarQuantization, ScalarQuantizationConfig, ScalarType,
    SearchParams, QuantizationSearchParams,
)
from sentence_transformers import SentenceTransformer

client = QdrantClient(host="localhost", port=6333)
model = SentenceTransformer("all-MiniLM-L6-v2")

N_VECTORS = 10000  # increase for more realistic benchmarks
DIM = 384
K = 10

# Generate synthetic embeddings (in production, use real data)
print(f"Generating {N_VECTORS} random vectors...")
np.random.seed(42)
vectors = np.random.random((N_VECTORS, DIM)).astype("float32")
# Normalize (important for cosine)
vectors = vectors / np.linalg.norm(vectors, axis=1, keepdims=True)

queries = vectors[:50]  # use first 50 as queries

# --- Collection WITHOUT quantization ---
COLL_NORMAL = "bench_normal"
if client.collection_exists(COLL_NORMAL):
    client.delete_collection(COLL_NORMAL)

client.create_collection(
    collection_name=COLL_NORMAL,
    vectors_config=VectorParams(size=DIM, distance=Distance.COSINE),
)

points = [PointStruct(id=i, vector=vectors[i].tolist()) for i in range(N_VECTORS)]
for i in range(0, len(points), 500):
    client.upsert(COLL_NORMAL, points[i:i+500])
print(f"Inserted {N_VECTORS} points (no quantization)")

# --- Collection WITH scalar quantization ---
COLL_SQ = "bench_sq"
if client.collection_exists(COLL_SQ):
    client.delete_collection(COLL_SQ)

client.create_collection(
    collection_name=COLL_SQ,
    vectors_config=VectorParams(size=DIM, distance=Distance.COSINE),
    quantization_config=ScalarQuantization(
        scalar=ScalarQuantizationConfig(type=ScalarType.INT8, always_ram=True),
    ),
)

for i in range(0, len(points), 500):
    client.upsert(COLL_SQ, points[i:i+500])
print(f"Inserted {N_VECTORS} points (scalar quantization)")

# Wait for optimization
import time as t
t.sleep(3)

# --- Benchmark ---
def benchmark_collection(name, search_params=None):
    latencies = []
    all_ids = []

    for q in queries:
        start = time.perf_counter()
        results = client.query_points(
            collection_name=name,
            query=q.tolist(),
            search_params=search_params,
            limit=K,
        )
        latencies.append(time.perf_counter() - start)
        all_ids.append([p.id for p in results.points])

    return latencies, all_ids

print("\nBenchmarking...\n")

# Ground truth (no quantization, high ef)
gt_latencies, gt_ids = benchmark_collection(
    COLL_NORMAL,
    SearchParams(hnsw_ef=256),
)

# Normal collection, default ef
normal_latencies, normal_ids = benchmark_collection(COLL_NORMAL)

# SQ without rescore
sq_latencies, sq_ids = benchmark_collection(
    COLL_SQ,
    SearchParams(quantization=QuantizationSearchParams(rescore=False)),
)

# SQ with rescore
sq_rescore_latencies, sq_rescore_ids = benchmark_collection(
    COLL_SQ,
    SearchParams(quantization=QuantizationSearchParams(rescore=True, oversampling=1.5)),
)

# Calculate recall
def calc_recall(predicted, ground_truth):
    recalls = []
    for pred, gt in zip(predicted, ground_truth):
        recalls.append(len(set(pred) & set(gt)) / K)
    return np.mean(recalls)

print(f"{'Config':<30} {'Avg Latency':>12} {'P99 Latency':>12} {'Recall@10':>10}")
print("-" * 70)

configs = [
    ("Normal (high ef)", normal_latencies, normal_ids),
    ("SQ (no rescore)", sq_latencies, sq_ids),
    ("SQ (rescore + oversample)", sq_rescore_latencies, sq_rescore_ids),
]

for name, lats, ids in configs:
    avg = np.mean(lats) * 1000
    p99 = np.percentile(lats, 99) * 1000
    recall = calc_recall(ids, gt_ids)
    print(f"{name:<30} {avg:>10.2f}ms {p99:>10.2f}ms {recall:>10.3f}")

# Collection sizes
normal_info = client.get_collection(COLL_NORMAL)
sq_info = client.get_collection(COLL_SQ)
print(f"\n--- Storage ---")
print(f"Normal:     {normal_info.points_count} points")
print(f"Quantized:  {sq_info.points_count} points")

# Cleanup
client.delete_collection(COLL_NORMAL)
client.delete_collection(COLL_SQ)
print("\nCleaned up benchmark collections")
```

---

## Exercise 5: Multi-Tenant Search

**Goal**: Implement tenant isolation using payload filters.

```python
"""
Exercise 5: Multi-tenant vector search.
Run: uv run python exercises/10_multi_tenant.py
"""
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct,
    Filter, FieldCondition, MatchValue,
    PayloadSchemaType,
)
from sentence_transformers import SentenceTransformer

client = QdrantClient(host="localhost", port=6333)
model = SentenceTransformer("all-MiniLM-L6-v2")

COLLECTION = "exercise_multitenant"

if client.collection_exists(COLLECTION):
    client.delete_collection(COLLECTION)

client.create_collection(
    collection_name=COLLECTION,
    vectors_config=VectorParams(size=384, distance=Distance.COSINE),
)

# CRITICAL: Index the tenant field for fast filtering
client.create_payload_index(COLLECTION, "tenant_id", PayloadSchemaType.KEYWORD)

# Simulate documents from different tenants
tenant_docs = {
    "acme_corp": [
        "Q3 revenue exceeded projections by 15%",
        "New product launch scheduled for September",
        "Employee satisfaction survey results are positive",
    ],
    "globex_inc": [
        "Server migration to AWS completed successfully",
        "Security audit findings require immediate attention",
        "New API rate limiting policy implemented",
    ],
    "initech": [
        "TPS report template has been updated",
        "Office relocation planned for next quarter",
        "New CRM system training starts Monday",
    ],
}

# Insert all documents with tenant isolation
point_id = 1
for tenant, docs in tenant_docs.items():
    embeddings = model.encode(docs)
    points = [
        PointStruct(
            id=point_id + i,
            vector=emb.tolist(),
            payload={"tenant_id": tenant, "text": doc},
        )
        for i, (doc, emb) in enumerate(zip(docs, embeddings))
    ]
    client.upsert(COLLECTION, points)
    point_id += len(docs)

print(f"Inserted docs for {len(tenant_docs)} tenants\n")

# --- Tenant-isolated search ---
query = "company performance and growth"
query_vec = model.encode(query).tolist()

# Search WITHOUT tenant filter (BAD — data leak!)
print(f"--- UNSAFE: No tenant filter ---")
results = client.query_points(
    collection_name=COLLECTION,
    query=query_vec,
    limit=5,
)
for p in results.points:
    print(f"  [{p.score:.4f}] [{p.payload['tenant_id']}] {p.payload['text']}")

# Search WITH tenant filter (SAFE — isolated)
for tenant in tenant_docs:
    print(f"\n--- SAFE: Tenant '{tenant}' only ---")
    results = client.query_points(
        collection_name=COLLECTION,
        query=query_vec,
        query_filter=Filter(
            must=[FieldCondition(key="tenant_id", match=MatchValue(value=tenant))],
        ),
        limit=3,
    )
    for p in results.points:
        print(f"  [{p.score:.4f}] {p.payload['text']}")

print("\n--- Key Takeaway ---")
print("Always include tenant_id filter in multi-tenant systems!")
print("Without it, users see other tenants' data → security breach.")
```

---

## Next Steps

After these exercises you should be comfortable with:
- Qdrant CRUD operations and filtering
- Named vectors for multi-aspect search
- Hybrid search with dense + sparse vectors
- Quantization impact on accuracy and performance
- Multi-tenant isolation patterns

Move on to **Phase 3** to build a complete RAG pipeline with FastAPI.
