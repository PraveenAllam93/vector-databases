# Phase 2: Production Vector DB (Qdrant)

## Reading Order

| # | Topic | File | Key Takeaway |
|---|-------|------|-------------|
| 1 | [Qdrant Setup](01-qdrant-setup.md) | Docker setup, architecture, ports, storage internals (WAL, segments, mmap) |
| 2 | [Collections & Points](02-collections-and-points.md) | Data model, CRUD, payload indexes, filtering (must/should/must_not), named vectors |
| 3 | [Hybrid Search](03-hybrid-search.md) | Dense + sparse vectors, BM25, SPLADE, RRF fusion, why it's mandatory in prod |
| 4 | [Quantization & Performance](04-quantization-and-performance.md) | SQ/BQ/PQ compression, mmap, efSearch tuning, benchmarking recall vs latency |
| 5 | [Hands-On Exercises](05-hands-on-exercises.md) | 5 exercises — CRUD, named vectors, hybrid search, quantization, multi-tenancy |

## Setup

```bash
# Start Qdrant
docker compose up -d

# Install Python dependencies
uv add qdrant-client sentence-transformers fastembed

# Verify
curl http://localhost:6333/healthz
open http://localhost:6333/dashboard
```

## Interview Quick Reference

**"How does Qdrant store data internally?"**
→ WAL for durability (write-ahead log), data in immutable segments, HNSW index built on sealed segments, mmap for vectors that exceed RAM.

**"How does metadata filtering work with HNSW?"**
→ Qdrant uses inline filtering during HNSW traversal — at each graph hop, it checks if the candidate passes the filter. This avoids the pre-filter (sparse graph) and post-filter (missing results) problems.

**"What is hybrid search and why is it important?"**
→ Combines dense vectors (semantic similarity) with sparse vectors (keyword/BM25). Dense misses exact terms ("E-4021"), sparse misses synonyms. Hybrid gets both. Results fused with RRF (Reciprocal Rank Fusion).

**"How do you reduce memory usage?"**
→ Scalar quantization (4x), binary quantization (32x, compatible models only), or product quantization (8-16x). Always use rescore from original vectors to recover accuracy. mmap lets you store vectors on disk with OS page caching.

**"How do you handle multi-tenancy?"**
→ Single collection with `tenant_id` payload field + keyword index. Every query MUST include tenant filter. More efficient than separate collections (shared HNSW graph, less overhead).
