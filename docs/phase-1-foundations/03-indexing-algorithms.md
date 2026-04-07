# 3. Indexing Algorithms — How to Search Billions of Vectors Fast

---

## ELI5

Imagine you have 1 billion trading cards and someone asks: "Find the 10 cards most similar to this one."

- **Brute Force (Flat)**: Compare every single card one by one. Accurate but takes forever.
- **IVF (Inverted File Index)**: Sort cards into labeled bins (sports, animals, vehicles). Only search the relevant bins. Faster, might miss some.
- **HNSW (Hierarchical Navigable Small World)**: Build a web of connections between similar cards. Start anywhere, follow links toward the target. Very fast, very accurate.
- **PQ (Product Quantization)**: Compress each card description into shorthand. Uses way less memory, slightly less accurate.

---

## The Core Problem: Why Not Just Compare Everything?

For 1 million vectors with 768 dimensions:

```
Brute force comparisons = 1,000,000
Each comparison         = 768 multiplications + 768 additions
Total operations        = ~1.5 BILLION per query

At 10 GHz effective throughput → ~150ms per query
```

At 100 million vectors: **~15 seconds per query**. Unacceptable.

**ANN (Approximate Nearest Neighbor)** algorithms trade a tiny bit of accuracy for massive speed improvements:

```
Exact search:  100% accuracy, O(n) time
ANN search:    95-99% accuracy, O(log n) time  ← this is the sweet spot
```

---

## Algorithm 1: Flat Index (Brute Force)

### How It Works

Compare the query vector against EVERY vector in the database. Return the closest ones.

```
Query: Q = [0.1, 0.5, 0.3]

Compare with:
  V1 = [0.2, 0.4, 0.3] → distance = 0.14
  V2 = [0.9, 0.1, 0.7] → distance = 1.02
  V3 = [0.1, 0.6, 0.2] → distance = 0.14
  ...
  V1000000 = [...]      → distance = 0.87

Sort by distance → return top K
```

### Properties

| Property | Value |
|----------|-------|
| Accuracy | 100% (exact) |
| Time complexity | O(n × d) where n=vectors, d=dimensions |
| Memory | O(n × d) |
| Build time | O(1) — no index to build |
| When to use | < 10,000 vectors, benchmarking, ground truth |

### In FAISS

```python
import faiss
import numpy as np

d = 768  # dimensions
index = faiss.IndexFlatL2(d)      # L2 distance
# or
index = faiss.IndexFlatIP(d)      # Inner product (dot product)

vectors = np.random.random((10000, d)).astype('float32')
index.add(vectors)

query = np.random.random((1, d)).astype('float32')
distances, indices = index.search(query, k=10)  # find 10 nearest
```

---

## Algorithm 2: IVF (Inverted File Index)

### ELI5

Divide the space into neighborhoods (clusters). At query time, only search nearby neighborhoods.

### How It Works

**Build Phase:**
1. Run k-means clustering on all vectors → creates `nlist` cluster centroids
2. Assign each vector to its nearest centroid

```
Cluster 1 (centroid C1): [V5, V12, V33, V87, ...]
Cluster 2 (centroid C2): [V1, V8, V45, V92, ...]
Cluster 3 (centroid C3): [V3, V17, V61, V74, ...]
...
```

**Search Phase:**
1. Find the `nprobe` nearest centroids to the query
2. Only search vectors in those clusters

```
Query Q:
  → Nearest centroids: C2, C5, C7 (nprobe=3)
  → Search only vectors in clusters 2, 5, 7
  → Skip all other clusters

Without IVF: search 1,000,000 vectors
With IVF:    search ~30,000 vectors (nprobe=3, nlist=100)
```

### Key Parameters

| Parameter | What It Does | Typical Values |
|-----------|-------------|----------------|
| `nlist` | Number of clusters | sqrt(n) to 4×sqrt(n) |
| `nprobe` | Clusters searched at query time | 1-20% of nlist |

**Tradeoff:**

```
nprobe=1:    Fastest, lowest recall (~60-70%)
nprobe=10:   Good balance (~90-95%)
nprobe=100:  Slow, near-exact (~99%)
nprobe=nlist: Same as brute force
```

### Visualization

```
        ┌────────────┐
        │  Cluster 1 │  ← nprobe=1: only search this
        │   • • •    │
        │  • Q • •   │  ← Query lands in cluster 1
        │   • •      │
        ├────────────┤
        │  Cluster 2 │  ← nprobe=2: also search this
        │  • • • •   │     (might have close vectors near the border!)
        │   • •      │
        ├────────────┤
        │  Cluster 3 │  ← nprobe=3: also search this
        │  • •       │
        │   •        │
        └────────────┘
```

### The Border Problem

A vector near the border of cluster 1 might be closer to Q than some vectors inside cluster 1. If we only search cluster 1 (nprobe=1), we miss it. Higher nprobe fixes this at the cost of speed.

### In FAISS

```python
nlist = 100  # number of clusters
quantizer = faiss.IndexFlatL2(d)
index = faiss.IndexIVFFlat(quantizer, d, nlist)

index.train(vectors)  # k-means clustering (required!)
index.add(vectors)

index.nprobe = 10  # search 10 nearest clusters
distances, indices = index.search(query, k=10)
```

---

## Algorithm 3: HNSW (Hierarchical Navigable Small World)

### ELI5

Imagine a city with express highways, regular roads, and local streets. To get somewhere:
1. Start on the highway (few stops, big jumps)
2. Exit to a regular road (medium jumps)
3. Take local streets to your exact destination (small, precise jumps)

### How It Works

HNSW builds a multi-layer graph:

```
Layer 3 (sparse):    A -------- D                    (express highways)
                     |          |
Layer 2 (medium):    A --- C -- D --- F              (regular roads)
                     |    |    |     |
Layer 1 (dense):     A-B-C-D-E-F-G-H-I-J            (local streets)
                     |/|\|/|\|/|\|/|\|/|
Layer 0 (all):       A B C D E F G H I J K L M N    (every node)
```

**Search Phase:**
1. Start at the top layer (few nodes, long-distance connections)
2. Greedily find the nearest node at this layer
3. Drop to the next layer (more nodes, shorter connections)
4. Repeat until layer 0
5. Return the nearest neighbors found at layer 0

```
Searching for Q:

Layer 3: A → D (D is closer to Q)
Layer 2: D → F (F is closer to Q)
Layer 1: F → G → H (H is closest at this layer)
Layer 0: H → K → L (L is the true nearest neighbor!)

Total comparisons: ~7 (instead of scanning all N)
```

### Key Parameters

| Parameter | What It Does | Typical Values | Tradeoff |
|-----------|-------------|----------------|----------|
| `M` | Max connections per node per layer | 16-64 | Higher → better recall, more memory |
| `efConstruction` | Search width during build | 100-500 | Higher → better graph quality, slower build |
| `efSearch` | Search width during query | 50-500 | Higher → better recall, slower query |

### Tuning Guide

```
Use Case                        M     efConstruction  efSearch
───────────────────────────────────────────────────────────────
Speed-critical (< 5ms)         16     100             50
Balanced                       32     200             100
Accuracy-critical (> 99%)      48     400             300
Maximum recall                 64     500             500
```

### Why HNSW is the Default Choice

| Property | HNSW | IVF |
|----------|------|-----|
| Recall at low latency | Excellent | Good |
| Build time | Moderate | Fast (k-means) |
| Memory | Higher (graph) | Lower |
| Supports deletion? | Yes (mark & rebuild) | Difficult |
| Dynamic inserts? | Yes | No (need retrain) |
| No training needed? | Yes | No |

HNSW wins on almost everything except memory. For most production workloads, it's the right default.

### In FAISS / Qdrant

```python
# FAISS
index = faiss.IndexHNSWFlat(d, M=32)  # M=32 connections
index.hnsw.efConstruction = 200
index.hnsw.efSearch = 100
index.add(vectors)
distances, indices = index.search(query, k=10)
```

```python
# Qdrant
from qdrant_client.models import VectorParams, Distance

client.create_collection(
    collection_name="my_collection",
    vectors_config=VectorParams(
        size=768,
        distance=Distance.COSINE,
        hnsw_config={
            "m": 32,
            "ef_construct": 200,
        }
    ),
)
# efSearch is set per query in Qdrant
```

---

## Algorithm 4: PQ (Product Quantization)

### ELI5

Instead of storing the full address of each house, store just the ZIP code and street name. You can still find the right neighborhood, but you might be a few houses off. Uses way less memory.

### How It Works

1. **Split** each vector into `m` sub-vectors
2. **Cluster** each sub-vector space independently (256 clusters each = 1 byte per sub-vector)
3. **Store** cluster IDs instead of original vectors

```
Original vector (768 dimensions):
[0.1, 0.3, 0.5, 0.2, ..., 0.7, 0.4, 0.8, 0.1]

Split into 8 sub-vectors (96 dims each):
[0.1, 0.3, ..., 0.5]  [0.2, ..., 0.7]  ... [0.4, 0.8, ..., 0.1]
      sub-1                 sub-2                   sub-8

Quantize each to nearest centroid:
    ID=42              ID=187            ...     ID=73

Compressed: [42, 187, ..., 73] → just 8 bytes! (vs 3072 bytes original)
```

**Compression ratio**: 768 × 4 bytes = 3,072 bytes → 8 bytes = **384x compression**

### Parameters

| Parameter | What It Does | Typical Values |
|-----------|-------------|----------------|
| `m` (sub-vectors) | How many segments | 8, 16, 32 |
| `nbits` | Bits per sub-quantizer | 8 (256 centroids) |

**Tradeoff:**
- More sub-vectors (larger `m`) → less compression, better accuracy
- Fewer sub-vectors (smaller `m`) → more compression, less accuracy

### IVFPQ: The Best of Both Worlds

Combine IVF (cluster-based search) + PQ (compressed storage):

```
Step 1: IVF narrows search to relevant clusters
Step 2: PQ allows storing billions of vectors in memory
Step 3: Search compressed vectors within the cluster
Step 4: (Optional) Re-rank top results against original vectors
```

This is how you search **billions** of vectors on a single machine.

### In FAISS

```python
# PQ alone
index = faiss.IndexPQ(d, m=8, nbits=8)
index.train(vectors)
index.add(vectors)

# IVFPQ (the production combo)
nlist = 1000
m = 8  # sub-vectors
quantizer = faiss.IndexFlatL2(d)
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8)
index.train(vectors)
index.add(vectors)
index.nprobe = 10
distances, indices = index.search(query, k=10)
```

---

## Algorithm Comparison Matrix

| Algorithm | Time Complexity | Memory | Accuracy | Build Time | Dynamic Updates |
|-----------|----------------|--------|----------|------------|-----------------|
| Flat | O(n) | Full vectors | 100% (exact) | None | Yes |
| IVF | O(n/nlist × nprobe) | Full vectors + centroids | 90-99% | Fast (k-means) | Needs retrain |
| HNSW | O(log n) | Full vectors + graph | 95-99% | Moderate | Yes |
| PQ | O(n) but faster | Compressed (10-100x smaller) | 85-95% | Moderate | Needs retrain |
| IVFPQ | O(n/nlist × nprobe) | Compressed + centroids | 85-95% | Moderate | Needs retrain |

---

## Decision Flowchart

```
How many vectors?
  │
  ├─ < 10K → Flat (brute force is fine)
  │
  ├─ 10K - 1M → HNSW (best accuracy/speed, ~4x memory overhead)
  │
  ├─ 1M - 100M → HNSW if RAM allows, else IVFPQ
  │
  └─ 100M+ → IVFPQ (or distributed HNSW across shards)
```

```
What matters most?
  │
  ├─ Accuracy → HNSW with high M, efSearch
  │
  ├─ Speed → IVF with low nprobe, or HNSW with low efSearch
  │
  ├─ Memory → PQ or IVFPQ
  │
  └─ Dynamic updates → HNSW (supports insert/delete without rebuild)
```

---

## Recall vs Latency: The Fundamental Tradeoff

Every ANN algorithm has knobs that trade recall for speed:

```
Recall@10 (%)
100 |                              ●  Flat (exact)
    |                        ●  HNSW (ef=500)
 95 |                   ●  HNSW (ef=100)
    |              ●  IVF (nprobe=50)
 90 |         ●  IVF (nprobe=10)
    |    ●  PQ (m=32)
 85 | ●  PQ (m=8)
    |
    └────────────────────────────────────
    0.1ms   1ms    5ms   10ms   50ms  100ms
                  Latency
```

**Recall@K** = "Of the true top-K nearest neighbors, how many did we actually find?"

```
True top-10:  [V3, V7, V12, V45, V67, V89, V102, V155, V200, V341]
ANN returned: [V3, V7, V12, V45, V67, V89, V102, V155, V203, V500]

Recall@10 = 8/10 = 80%  (missed V200 and V341)
```

---

## Summary

| Algorithm | One-Line | Use When |
|-----------|----------|----------|
| Flat | Compare everything | Small datasets, ground truth |
| IVF | Cluster then search | Need fast build, moderate scale |
| HNSW | Multi-layer graph navigation | Default choice for most production use |
| PQ | Compress vectors | Memory constrained, billions of vectors |
| IVFPQ | Cluster + compress | Maximum scale on limited hardware |

**Default recommendation**: Start with **HNSW** (M=32, efConstruction=200, efSearch=100). Tune from there based on recall/latency benchmarks.

---

## Resources

- [FAISS Wiki](https://github.com/facebookresearch/faiss/wiki) — authoritative guide on all index types
- [Pinecone: Nearest Neighbor Indexes](https://www.pinecone.io/learn/series/faiss/vector-indexes/) — visual FAISS guide
- [HNSW Paper (Malkov & Yashunin, 2018)](https://arxiv.org/abs/1603.09320) — original HNSW paper
- [James Briggs: FAISS Tutorial](https://www.youtube.com/watch?v=sKyvsdEv6rk) — practical YouTube walkthrough
- [Ann-Benchmarks](http://ann-benchmarks.com/) — performance comparison of all ANN algorithms
- [Jeff Johnson: Billion-Scale Similarity Search with FAISS](https://engineering.fb.com/2017/03/29/data-infrastructure/faiss-a-library-for-efficient-similarity-search/) — Facebook Engineering blog
