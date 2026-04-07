# 4. Quantization & Performance — Making Qdrant Fast and Lean

---

## ELI5

Imagine each vector is a photo. A full-quality photo (32-bit float per dimension) takes a lot of space. Quantization is like converting photos to thumbnails — they take 4-8x less space, look slightly blurry, but you can still tell what's in them.

This lets you fit millions more vectors in the same amount of RAM, with only a tiny accuracy loss.

---

## Why Quantization?

### The Memory Problem

```
1 million vectors × 768 dimensions × 4 bytes (float32) = 3.07 GB just for vectors
+ HNSW graph overhead (~1.5x) = ~4.6 GB
+ Payloads = ~5-6 GB total

10 million vectors = ~50-60 GB → needs a beefy machine
100 million vectors = ~500-600 GB → impossible on a single machine with float32
```

Quantization shrinks vectors:

```
Scalar quantization (8-bit):  4 bytes → 1 byte  = 4x reduction
Binary quantization (1-bit):  4 bytes → 1 bit   = 32x reduction
Product quantization (PQ):    4 bytes → ~0.5 byte = 8x reduction
```

---

## Scalar Quantization (SQ)

### How It Works

Convert each float32 value to an int8 (0-255):

```
Original float32:  [-0.45,  0.78,  0.12, -0.33,  0.91]
                      ↓ scale to 0-255 range ↓
Quantized int8:    [   35,   217,   144,    58,   238 ]

Formula:
  quantized = round((value - min_value) / (max_value - min_value) × 255)
```

4 bytes → 1 byte per dimension = **4x memory reduction**.

### Enable in Qdrant

```python
from qdrant_client.models import (
    VectorParams, Distance, ScalarQuantization,
    ScalarQuantizationConfig, ScalarType, QuantizationSearchParams,
)

# At collection creation
client.create_collection(
    collection_name="articles_sq",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    quantization_config=ScalarQuantization(
        scalar=ScalarQuantizationConfig(
            type=ScalarType.INT8,
            quantile=0.99,      # clip outliers beyond 99th percentile
            always_ram=True,    # keep quantized vectors in RAM (originals on disk)
        ),
    ),
)

# Search with quantization (re-score from original vectors for accuracy)
results = client.query_points(
    collection_name="articles_sq",
    query=[0.1, 0.2, ...],
    search_params=QueryParams(
        quantization=QuantizationSearchParams(
            rescore=True,      # re-rank top results using original float32 vectors
            oversampling=1.5,  # fetch 1.5x more candidates before re-scoring
        ),
    ),
    limit=10,
)
```

### How Rescore Works

```
Step 1: Search quantized vectors (fast, in RAM)
  → Get top 15 candidates (oversampling=1.5 × limit=10)

Step 2: Re-score 15 candidates using original float32 vectors (accurate, from disk/mmap)
  → Re-rank by true similarity

Step 3: Return top 10

Result: Nearly exact accuracy at quantized speed
```

### Tradeoffs

| Aspect | Without SQ | With SQ | With SQ + Rescore |
|--------|-----------|---------|-------------------|
| Memory | 100% | 25% | 25% + originals on disk |
| Speed | Baseline | ~2-4x faster | ~1.5-3x faster |
| Recall | 100% | 95-99% | 99%+ |

---

## Binary Quantization (BQ)

### How It Works

Convert each float to a single bit (0 or 1):

```
Original: [-0.45,  0.78,  0.12, -0.33,  0.91]
Binary:   [    0,     1,     1,     0,     1 ]

Rule: negative → 0, non-negative → 1
```

4 bytes → 1/8 byte = **32x memory reduction**. Search uses Hamming distance (count differing bits), which is blazing fast on modern CPUs.

### When to Use

Binary quantization works well ONLY with certain embedding models that produce balanced distributions. Works best with:
- OpenAI `text-embedding-3-*` (designed for it)
- Cohere embed v3
- Some Sentence Transformer models

Does NOT work well with models that produce mostly positive values.

### Enable in Qdrant

```python
from qdrant_client.models import BinaryQuantization, BinaryQuantizationConfig

client.create_collection(
    collection_name="articles_bq",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    quantization_config=BinaryQuantization(
        binary=BinaryQuantizationConfig(
            always_ram=True,
        ),
    ),
)

# MUST use rescore with binary quantization (too lossy otherwise)
results = client.query_points(
    collection_name="articles_bq",
    query=[0.1, 0.2, ...],
    search_params=QueryParams(
        quantization=QuantizationSearchParams(
            rescore=True,
            oversampling=2.0,  # higher oversampling needed for BQ
        ),
    ),
    limit=10,
)
```

### Tradeoffs

| Aspect | Without BQ | With BQ + Rescore |
|--------|-----------|-------------------|
| Memory | 100% | ~3% + originals on disk |
| Speed | Baseline | ~5-10x faster (first pass) |
| Recall | 100% | 95-99% (with rescore) |
| Model requirement | Any | Must have balanced distributions |

---

## Product Quantization (PQ)

### How It Works

Same concept as FAISS PQ from Phase 1:
1. Split vector into sub-vectors
2. Cluster each sub-vector independently
3. Store cluster IDs instead of values

### Enable in Qdrant

```python
from qdrant_client.models import ProductQuantization, ProductQuantizationConfig, CompressionRatio

client.create_collection(
    collection_name="articles_pq",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    quantization_config=ProductQuantization(
        product=ProductQuantizationConfig(
            compression=CompressionRatio.X16,  # 16x compression
            always_ram=True,
        ),
    ),
)
```

### When to Use PQ vs SQ vs BQ

```
Default choice:
  → Scalar Quantization (SQ) — 4x compression, minimal accuracy loss

Maximum compression with compatible models:
  → Binary Quantization (BQ) — 32x compression, needs rescore

Large dimensions + extreme scale:
  → Product Quantization (PQ) — 8-16x compression, configurable
```

---

## Memory-Mapped Vectors (mmap)

Instead of quantization, you can keep full-precision vectors on disk and let the OS manage caching:

```python
from qdrant_client.models import OptimizersConfigDiff

# Enable mmap for vectors
client.update_collection(
    collection_name="articles",
    optimizer_config=OptimizersConfigDiff(
        memmap_threshold=20000,  # mmap segments with >20K vectors
    ),
)
```

**How it works:**

```
RAM:  [HNSW graph] [quantized vectors] [hot payload indexes]
Disk: [original vectors (mmap'd)] [cold payloads]

OS page cache: frequently accessed vectors cached in RAM automatically
```

**Combine with quantization** for best of both worlds:
- Quantized vectors in RAM (fast first-pass search)
- Original vectors on disk/mmap (accurate rescore)

---

## HNSW Tuning for Performance

### efSearch: The Main Latency Knob

```python
from qdrant_client.models import SearchParams

# Fast search (lower recall)
results = client.query_points(
    collection_name="articles",
    query=[...],
    search_params=SearchParams(hnsw_ef=50),
    limit=10,
)

# Accurate search (higher latency)
results = client.query_points(
    collection_name="articles",
    query=[...],
    search_params=SearchParams(hnsw_ef=256),
    limit=10,
)
```

**Benchmarking guide:**

```
efSearch=50:   ~1-2ms,  recall@10 ≈ 92%
efSearch=100:  ~2-5ms,  recall@10 ≈ 97%
efSearch=200:  ~5-10ms, recall@10 ≈ 99%
efSearch=500:  ~10-20ms,recall@10 ≈ 99.5%

Rule: efSearch must be >= limit (k). Start at 128, tune from there.
```

### Collection-Level HNSW Defaults

```python
from qdrant_client.models import HnswConfigDiff

# Set at creation time
client.create_collection(
    collection_name="articles",
    vectors_config=VectorParams(
        size=768,
        distance=Distance.COSINE,
        hnsw_config=HnswConfigDiff(
            m=32,                    # connections per node
            ef_construct=200,        # build quality
            full_scan_threshold=10000,  # flat scan for small segments
        ),
    ),
)

# Update after creation
client.update_collection(
    collection_name="articles",
    hnsw_config=HnswConfigDiff(
        m=48,  # increase for better recall (requires re-indexing)
    ),
)
```

---

## Benchmarking: Recall vs Latency

### How to Measure

```python
"""
Benchmark recall@K vs latency for different settings.
"""
import time
import numpy as np
from qdrant_client import QdrantClient
from qdrant_client.models import SearchParams, QuantizationSearchParams

client = QdrantClient(host="localhost", port=6333)

# Step 1: Get ground truth (exact search)
def get_ground_truth(query_vector, k=10):
    """Use high efSearch for near-exact results."""
    results = client.query_points(
        collection_name="articles",
        query=query_vector,
        search_params=SearchParams(hnsw_ef=512, exact=False),
        limit=k,
    )
    return set(p.id for p in results.points)

# Or use exact search
def get_exact_truth(query_vector, k=10):
    results = client.query_points(
        collection_name="articles",
        query=query_vector,
        search_params=SearchParams(exact=True),  # brute force
        limit=k,
    )
    return set(p.id for p in results.points)

# Step 2: Benchmark different settings
def benchmark(query_vectors, k=10, ef_values=[50, 100, 200, 500]):
    for ef in ef_values:
        latencies = []
        recalls = []

        for q in query_vectors:
            gt = get_ground_truth(q, k)

            start = time.perf_counter()
            results = client.query_points(
                collection_name="articles",
                query=q,
                search_params=SearchParams(hnsw_ef=ef),
                limit=k,
            )
            latencies.append(time.perf_counter() - start)

            retrieved = set(p.id for p in results.points)
            recalls.append(len(gt & retrieved) / k)

        avg_latency = np.mean(latencies) * 1000
        p99_latency = np.percentile(latencies, 99) * 1000
        avg_recall = np.mean(recalls)

        print(f"ef={ef:3d}: recall@{k}={avg_recall:.3f}, "
              f"avg={avg_latency:.1f}ms, p99={p99_latency:.1f}ms")
```

### What Good Looks Like

```
Target for production:
  - recall@10 >= 0.95 (95%)
  - p99 latency <= 50ms
  - p95 latency <= 20ms

Typical results (1M vectors, 768d, HNSW M=32):
  ef=50:  recall=0.92, p99=3ms   ← too low recall
  ef=100: recall=0.97, p99=5ms   ← good balance ✓
  ef=200: recall=0.99, p99=10ms  ← great recall, still fast ✓
  ef=500: recall=0.995, p99=25ms ← diminishing returns
```

---

## Optimizer Settings

Qdrant automatically optimizes segments in the background. You can tune this:

```python
from qdrant_client.models import OptimizersConfigDiff

client.update_collection(
    collection_name="articles",
    optimizer_config=OptimizersConfigDiff(
        # Merge small segments (reduces segment count, improves search)
        default_segment_number=2,

        # Max segment size before splitting
        max_segment_size=200000,

        # Threshold to switch from flat to HNSW index
        indexing_threshold=20000,

        # How often to flush WAL to segments
        flush_interval_sec=5,

        # Vacuum deleted points
        vacuum_min_vector_number=1000,
        max_optimization_threads=2,
    ),
)
```

---

## Performance Checklist for Production

```
□ Payload indexes created on all filtered fields
□ Quantization enabled (SQ as default, BQ for compatible models)
□ Rescore enabled with oversampling (1.5-2.0)
□ efSearch tuned via benchmarking (start at 128)
□ M set appropriately (32 for most, 48-64 for high recall needs)
□ mmap enabled for segments above threshold
□ Batch size optimized for ingestion (100-1000)
□ gRPC client preferred for bulk operations
□ Payload projection used (only return needed fields)
□ Score threshold set to filter irrelevant results
```

---

## Summary

| Technique | Compression | Speed Gain | Recall Impact | When to Use |
|-----------|------------|------------|---------------|-------------|
| Scalar Quantization (SQ) | 4x | 2-4x | Minimal (99%+) | Default choice |
| Binary Quantization (BQ) | 32x | 5-10x | Moderate (95-99%) | Compatible models only |
| Product Quantization (PQ) | 8-16x | 3-5x | Moderate (90-97%) | Extreme scale |
| mmap | N/A (full precision) | OS-managed | None | When disk > RAM budget |
| efSearch tuning | N/A | Direct | Direct tradeoff | Always tune |
| Rescore | N/A | Slight cost | Recovers accuracy | Always use with quantization |

**Golden rule**: SQ + rescore + efSearch=128 is the right starting point for 90% of production workloads.

---

## Resources

- [Qdrant Quantization](https://qdrant.tech/documentation/guides/quantization/) — official guide
- [Qdrant Optimizer](https://qdrant.tech/documentation/concepts/optimizer/) — segment optimization
- [Qdrant Binary Quantization](https://qdrant.tech/articles/binary-quantization/) — deep dive blog
- [Qdrant Benchmarks](https://qdrant.tech/benchmarks/) — official performance comparisons
