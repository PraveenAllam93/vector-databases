Phase 1 — Foundations
- Embeddings: what they are, how they're generated, dimensions, normalization
- Similarity metrics: cosine, L2, dot product — when to use which
- Indexing algorithms: HNSW, IVF, PQ, flat — tradeoffs and tuning
- Hands-on: Chroma locally, store/query vectors

Phase 2 — Production Vector DB (Qdrant)
- Qdrant setup with Docker
- Collections, points, payloads, metadata filtering
- Hybrid search: BM25 + vector similarity
- Benchmarking: recall@K vs latency

Phase 3 — RAG Pipeline
- FastAPI API layer
- Chunking strategies, embedding versioning
- Query → retrieve → re-rank (cross-encoder) → LLM response
- Caching with Redis

Phase 4 — System Design & Distributed Systems
- Leader-follower replication
- Sharding strategies (by user, domain, hash)
- Race conditions: concurrent writes, index rebuild conflicts, read-after-write
- Solutions: idempotent writes, write queues (Kafka), snapshot isolation
- Multi-region deployment

Phase 5 — Security
- AuthN: API keys, JWT, OAuth2
- AuthZ: RBAC, multi-tenant namespace isolation
- Encryption: AES-256 at rest, TLS/mTLS in transit

Phase 6 — Infrastructure & CI/CD
- Docker Compose for local multi-service setup
- Kubernetes: separate ingestion/query/indexing services, horizontal scaling
- GitHub Actions: CI pipeline, automated tests, deployment
- Cloud deployment (AWS/GCP)

Phase 7 — Observability & Evaluation
- Prometheus + Grafana + OpenTelemetry
- Key metrics: p95/p99 latency, recall@K, throughput
- Query tracing, slow query logging
- Evaluation: Recall@K, Precision@K, MRR, NDCG