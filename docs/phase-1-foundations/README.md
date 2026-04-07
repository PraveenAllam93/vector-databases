# Phase 1: Foundations

## Reading Order

| # | Topic | File | Key Takeaway |
|---|-------|------|-------------|
| 1 | [Embeddings](01-embeddings.md) | What vectors are, how they're generated, why normalization matters |
| 2 | [Similarity Metrics](02-similarity-metrics.md) | Cosine vs L2 vs Dot Product — when to use which |
| 3 | [Indexing Algorithms](03-indexing-algorithms.md) | Flat, IVF, HNSW, PQ — how to search billions of vectors fast |
| 4 | [Vector Databases](04-vector-databases.md) | Chroma, Qdrant, Milvus, FAISS — selection and usage |
| 5 | [Hands-On Exercises](05-hands-on-exercises.md) | 5 exercises to build intuition with real code |

## Setup for Exercises

```bash
cd vector-databases
uv add sentence-transformers numpy faiss-cpu chromadb
```

## Interview Quick Reference

**"What is a vector embedding?"**
→ A learned fixed-size numerical representation where similar meanings = close vectors in high-dimensional space.

**"Why not brute force search?"**
→ O(n) doesn't scale. At 1M vectors with 768 dims, that's ~1.5 billion operations per query. ANN algorithms (HNSW, IVF) achieve 95-99% recall at O(log n).

**"Cosine vs L2 vs Dot Product?"**
→ Cosine = angle only (ignore magnitude), best for text. L2 = absolute distance, best for spatial. Dot Product = angle + magnitude, used when model trains with it. Normalized cosine = dot product.

**"Why HNSW over IVF?"**
→ No training needed, supports dynamic inserts/deletes, better recall at low latency. Only downside is ~1.5x memory overhead for the graph.

**"What does a vector database add over FAISS?"**
→ Persistence, metadata filtering, CRUD operations, replication, sharding, authentication, APIs, backups.
