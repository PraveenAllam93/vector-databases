# 2. Collections, Points & Payloads — Qdrant Data Model

---

## ELI5

Think of a **collection** as a folder. Inside the folder are **points** — each point is a sticky note with:
- A **number** written on it (the vector — what the content means)
- A **label** (the ID — how you find it again)
- Some **extra info** written on the back (the payload — metadata like author, date, category)

You can search by similarity (find notes that mean something similar) AND filter by the extra info (only notes from 2024, only notes in the "tech" category).

---

## Collections

A collection is a named container for vectors of the same type and dimension.

### Create a Collection

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams

client = QdrantClient(host="localhost", port=6333)

client.create_collection(
    collection_name="articles",
    vectors_config=VectorParams(
        size=768,              # embedding dimensions
        distance=Distance.COSINE,  # similarity metric
    ),
)
```

### Distance Options

```python
Distance.COSINE       # Cosine similarity (most common for text)
Distance.EUCLID       # L2 / Euclidean distance
Distance.DOT          # Dot product
Distance.MANHATTAN    # L1 distance (sum of absolute differences)
```

### Collection with HNSW Tuning

```python
from qdrant_client.models import HnswConfigDiff

client.create_collection(
    collection_name="articles",
    vectors_config=VectorParams(
        size=768,
        distance=Distance.COSINE,
        hnsw_config=HnswConfigDiff(
            m=32,                # connections per node (default: 16)
            ef_construct=200,    # build quality (default: 100)
            full_scan_threshold=10000,  # below this, skip HNSW, do flat scan
        ),
    ),
)
```

### Multiple Vector Types (Named Vectors)

One point can have MULTIPLE vectors — useful when you embed different fields separately:

```python
from qdrant_client.models import VectorParams, Distance

client.create_collection(
    collection_name="articles",
    vectors_config={
        "title": VectorParams(size=384, distance=Distance.COSINE),
        "content": VectorParams(size=768, distance=Distance.COSINE),
        "image": VectorParams(size=512, distance=Distance.COSINE),
    },
)
```

**Interview insight**: Named vectors let you search by different aspects of the same document. "Find articles with similar titles" vs "Find articles with similar content" — different embeddings, same collection.

### Collection Operations

```python
# List all collections
collections = client.get_collections()
print(collections.collections)

# Get collection info (stats, config)
info = client.get_collection("articles")
print(f"Points: {info.points_count}")
print(f"Vectors: {info.vectors_count}")
print(f"Segments: {info.segments_count}")
print(f"Status: {info.status}")

# Delete a collection
client.delete_collection("articles")

# Check if collection exists
exists = client.collection_exists("articles")
```

---

## Points

A point = ID + vector(s) + payload. It's the fundamental unit in Qdrant.

### Insert Points (Upsert)

```python
from qdrant_client.models import PointStruct

# Upsert = insert or update (idempotent!)
client.upsert(
    collection_name="articles",
    points=[
        PointStruct(
            id=1,
            vector=[0.1, 0.2, 0.3, ...],  # 768 floats
            payload={
                "title": "Introduction to AI",
                "author": "Jane Smith",
                "category": "tech",
                "published": "2024-01-15",
                "views": 15000,
                "tags": ["ai", "machine-learning", "beginner"],
            },
        ),
        PointStruct(
            id=2,
            vector=[0.4, 0.5, 0.6, ...],
            payload={
                "title": "Cooking with Python",
                "author": "Bob Chef",
                "category": "food",
                "published": "2024-03-20",
                "views": 8500,
                "tags": ["cooking", "recipes"],
            },
        ),
    ],
)
```

### ID Types

```python
# Integer IDs
PointStruct(id=1, ...)
PointStruct(id=42, ...)

# UUID strings
PointStruct(id="550e8400-e29b-41d4-a716-446655440000", ...)

# Use UUIDs when:
#  - Multiple ingestion services (no coordination needed)
#  - IDs come from external systems
#  - You need globally unique IDs

# Use integers when:
#  - Simple, sequential data
#  - Slightly more memory efficient
```

### Named Vector Points

```python
# For collections with multiple vector types
client.upsert(
    collection_name="articles",
    points=[
        PointStruct(
            id=1,
            vector={
                "title": [0.1, 0.2, ...],    # 384 dims
                "content": [0.4, 0.5, ...],   # 768 dims
            },
            payload={"title": "Introduction to AI"},
        ),
    ],
)
```

### Batch Upsert (Production Pattern)

```python
from qdrant_client.models import PointStruct, Batch

# Method 1: List of PointStruct (cleaner)
BATCH_SIZE = 100

for i in range(0, len(all_points), BATCH_SIZE):
    batch = all_points[i:i + BATCH_SIZE]
    client.upsert(
        collection_name="articles",
        points=batch,
    )
    print(f"Upserted {min(i + BATCH_SIZE, len(all_points))}/{len(all_points)}")

# Method 2: Batch object (slightly more efficient)
client.upsert(
    collection_name="articles",
    points=Batch(
        ids=[1, 2, 3],
        vectors=[[0.1, ...], [0.4, ...], [0.7, ...]],
        payloads=[
            {"title": "AI Intro"},
            {"title": "ML Guide"},
            {"title": "DL Deep Dive"},
        ],
    ),
)
```

### Retrieve Points

```python
# Get points by ID
points = client.retrieve(
    collection_name="articles",
    ids=[1, 2, 3],
    with_vectors=True,   # include vectors in response
    with_payload=True,    # include payloads in response
)

for point in points:
    print(f"ID: {point.id}, Title: {point.payload['title']}")
```

### Delete Points

```python
from qdrant_client.models import PointIdsList, FilterSelector, Filter, FieldCondition, MatchValue

# Delete by IDs
client.delete(
    collection_name="articles",
    points_selector=PointIdsList(points=[1, 2, 3]),
)

# Delete by filter (powerful!)
client.delete(
    collection_name="articles",
    points_selector=FilterSelector(
        filter=Filter(
            must=[
                FieldCondition(key="category", match=MatchValue(value="deprecated")),
            ]
        )
    ),
)
```

### Update Payload (Without Re-embedding)

```python
# Set payload fields (merge with existing)
client.set_payload(
    collection_name="articles",
    payload={"views": 16000, "last_updated": "2024-06-01"},
    points=[1],  # point IDs
)

# Overwrite entire payload
client.overwrite_payload(
    collection_name="articles",
    payload={"title": "Updated Title", "category": "tech"},
    points=[1],
)

# Delete specific payload keys
client.delete_payload(
    collection_name="articles",
    keys=["deprecated_field"],
    points=[1],
)
```

**Key insight**: You can update metadata without re-embedding! Vectors are expensive to generate — payload updates are cheap.

---

## Payloads Deep Dive

Payloads are arbitrary JSON attached to points. They're what makes vector databases useful beyond pure similarity search.

### Supported Data Types

```python
payload = {
    # String
    "title": "Introduction to AI",

    # Integer
    "views": 15000,

    # Float
    "score": 4.85,

    # Boolean
    "published": True,

    # Array (of same type)
    "tags": ["ai", "ml", "beginner"],

    # Nested object
    "author": {
        "name": "Jane Smith",
        "email": "jane@example.com",
    },

    # Datetime (stored as string, use ISO 8601)
    "created_at": "2024-01-15T10:30:00Z",

    # Null
    "deleted_at": None,

    # Geo point
    "location": {"lat": 40.7128, "lon": -74.0060},
}
```

### Payload Indexes

By default, payloads are NOT indexed — filtering scans all payloads. For large collections, create indexes on fields you filter frequently:

```python
from qdrant_client.models import PayloadSchemaType

# Index a keyword field (exact match)
client.create_payload_index(
    collection_name="articles",
    field_name="category",
    field_schema=PayloadSchemaType.KEYWORD,
)

# Index an integer field (range queries)
client.create_payload_index(
    collection_name="articles",
    field_name="views",
    field_schema=PayloadSchemaType.INTEGER,
)

# Index a float field
client.create_payload_index(
    collection_name="articles",
    field_name="score",
    field_schema=PayloadSchemaType.FLOAT,
)

# Index a boolean field
client.create_payload_index(
    collection_name="articles",
    field_name="published",
    field_schema=PayloadSchemaType.BOOL,
)

# Index a geo field (spatial queries)
client.create_payload_index(
    collection_name="articles",
    field_name="location",
    field_schema=PayloadSchemaType.GEO,
)

# Index a text field (full-text search!)
client.create_payload_index(
    collection_name="articles",
    field_name="title",
    field_schema=PayloadSchemaType.TEXT,
)

# Index nested fields
client.create_payload_index(
    collection_name="articles",
    field_name="author.name",
    field_schema=PayloadSchemaType.KEYWORD,
)
```

**Performance impact**:

```
Without payload index:
  Filter "category = tech" on 1M points → scan all 1M payloads → ~50ms

With payload index:
  Filter "category = tech" on 1M points → bitmap lookup → ~0.1ms
```

**Rule**: Always index fields you filter on in production.

---

## Search (Query Points)

### Basic Similarity Search

```python
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, 0.3, ...],  # query vector
    limit=10,                      # top K results
)

for point in results.points:
    print(f"ID: {point.id}, Score: {point.score:.4f}")
    print(f"  Title: {point.payload['title']}")
```

### Search with Filters

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

# Must match (AND)
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        must=[
            FieldCondition(key="category", match=MatchValue(value="tech")),
            FieldCondition(key="published", match=MatchValue(value=True)),
        ]
    ),
    limit=10,
)

# Should match (OR — at least one must match)
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        should=[
            FieldCondition(key="category", match=MatchValue(value="tech")),
            FieldCondition(key="category", match=MatchValue(value="science")),
        ]
    ),
    limit=10,
)

# Must not (exclusion)
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        must_not=[
            FieldCondition(key="category", match=MatchValue(value="spam")),
        ]
    ),
    limit=10,
)
```

### Range Filters

```python
from qdrant_client.models import Range

# Numeric range
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        must=[
            FieldCondition(
                key="views",
                range=Range(gte=1000, lte=50000),  # 1000 <= views <= 50000
            ),
        ]
    ),
    limit=10,
)
```

### Array Filters (Tags)

```python
from qdrant_client.models import MatchAny, MatchExcept

# Match any tag in list
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        must=[
            FieldCondition(
                key="tags",
                match=MatchAny(any=["ai", "machine-learning"]),
            ),
        ]
    ),
    limit=10,
)

# Exclude specific tags
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        must=[
            FieldCondition(
                key="tags",
                match=MatchExcept(except_=["deprecated", "draft"]),
            ),
        ]
    ),
    limit=10,
)
```

### Geo Filters

```python
from qdrant_client.models import GeoRadius, GeoPoint

# Find articles near New York (within 50km)
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        must=[
            FieldCondition(
                key="location",
                geo_radius=GeoRadius(
                    center=GeoPoint(lat=40.7128, lon=-74.0060),
                    radius=50000,  # meters
                ),
            ),
        ]
    ),
    limit=10,
)
```

### Nested Filters

```python
# Filter on nested payload fields
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        must=[
            FieldCondition(
                key="author.name",
                match=MatchValue(value="Jane Smith"),
            ),
        ]
    ),
    limit=10,
)
```

### Complex Combined Filters

```python
# Tech articles from 2024 with >5000 views, NOT from spam authors
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        must=[
            FieldCondition(key="category", match=MatchValue(value="tech")),
            FieldCondition(key="views", range=Range(gte=5000)),
            FieldCondition(key="published", match=MatchValue(value=True)),
        ],
        must_not=[
            FieldCondition(key="author.name", match=MatchValue(value="SpamBot")),
        ],
    ),
    limit=10,
)
```

### Search with Named Vectors

```python
# Search by title similarity
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],       # 384 dims (title vector)
    using="title",                 # which named vector to search
    limit=10,
)

# Search by content similarity
results = client.query_points(
    collection_name="articles",
    query=[0.4, 0.5, ...],       # 768 dims (content vector)
    using="content",
    limit=10,
)
```

### Control What's Returned

```python
from qdrant_client.models import WithPayload

# Only return specific payload fields (saves bandwidth)
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    with_payload=WithPayload(include=["title", "category"]),
    limit=10,
)

# Exclude heavy payload fields
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    with_payload=WithPayload(exclude=["full_text"]),
    limit=10,
)

# No payload at all (fastest)
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    with_payload=False,
    limit=10,
)

# Include vectors in response (usually not needed)
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    with_vectors=True,
    limit=10,
)
```

### Search with Score Threshold

```python
# Only return results above a minimum similarity
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],
    score_threshold=0.7,  # ignore results below 0.7 similarity
    limit=10,
)
```

---

## Scroll (Pagination)

For browsing all points (not similarity search):

```python
# First page
results, next_offset = client.scroll(
    collection_name="articles",
    limit=100,
    with_payload=True,
    with_vectors=False,
)

# Next page
while next_offset is not None:
    results, next_offset = client.scroll(
        collection_name="articles",
        offset=next_offset,
        limit=100,
        with_payload=True,
        with_vectors=False,
    )
    # process results...
```

---

## Count

```python
# Count all points
count = client.count(collection_name="articles")
print(f"Total: {count.count}")

# Count with filter
count = client.count(
    collection_name="articles",
    count_filter=Filter(
        must=[FieldCondition(key="category", match=MatchValue(value="tech"))],
    ),
)
print(f"Tech articles: {count.count}")
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Collection | Container for same-dimension vectors; configure distance + HNSW params |
| Point | ID + vector + payload; upsert is idempotent |
| Payload | Arbitrary JSON metadata; index fields you filter on |
| Named vectors | Multiple embeddings per point (title, content, image) |
| Filters | must (AND), should (OR), must_not (NOT); combine freely |
| Payload indexes | Create on frequently filtered fields; huge performance impact |
| Batch upsert | Always batch (100-1000 per batch) in production |
| Scroll | For pagination / full scan (not similarity-based) |

---

## Resources

- [Qdrant Collections](https://qdrant.tech/documentation/concepts/collections/) — official docs
- [Qdrant Points](https://qdrant.tech/documentation/concepts/points/) — all point operations
- [Qdrant Filtering](https://qdrant.tech/documentation/concepts/filtering/) — complete filter reference
- [Qdrant Payload Indexing](https://qdrant.tech/documentation/concepts/indexing/#payload-index) — index types and performance
