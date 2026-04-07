# Vector Databases — End-to-End Learning Curriculum

A production-grade learning path covering vector databases, RAG pipelines, distributed systems, security, infrastructure, and observability. Built for interview preparation and real-world production use.

---

## Curriculum

| Phase | Topic | Directory |
|-------|-------|-----------|
| 1 | Foundations (embeddings, similarity, indexing, Chroma) | [docs/phase-1-foundations](docs/phase-1-foundations/) |
| 2 | Production Qdrant (setup, CRUD, hybrid search, quantization) | [docs/phase-2-qdrant](docs/phase-2-qdrant/) |
| 3 | RAG Pipeline (chunking, ingestion, retrieval, FastAPI) | [docs/phase-3-rag](docs/phase-3-rag/) |
| 4 | System Design (CAP, Raft, replication, sharding, race conditions, HA) | [docs/phase-4-system-design](docs/phase-4-system-design/) |
| 5 | Security (AuthN/AuthZ, multi-tenant isolation, encryption, secrets) | [docs/phase-5-security](docs/phase-5-security/) |
| 6 | Infrastructure & CI/CD (Docker, Kubernetes, GitHub Actions, AWS/GCP) | [docs/phase-6-infrastructure](docs/phase-6-infrastructure/) |
| 7 | Observability & Evaluation (Prometheus, OTel, Recall@K, benchmarking) | [docs/phase-7-observability](docs/phase-7-observability/) |

---

## Quick Start

```bash
# Start Qdrant locally
docker compose up -d

# Qdrant Web UI
open http://localhost:6333/dashboard
```

---

## Key Interview Topics by Phase

**Phase 1** — What is cosine similarity? When do you use L2? Why normalize vectors?

**Phase 2** — How does HNSW work? What do M and efSearch control? How does Qdrant's WAL work?

**Phase 3** — What chunking strategy do you use? How do you handle deduplication? Explain the two-stage retrieval pipeline.

**Phase 4** — Explain CAP theorem for vector DBs. How does Raft elect a leader? What race conditions occur during ingestion? How do you shard 500M vectors across 1000 tenants?

**Phase 5** — How do you prevent data leaks in multi-tenancy? What's the difference between authentication and authorization? How do you manage secrets in Kubernetes?

**Phase 6** — How do you achieve zero-downtime deployments? What's the difference between StatefulSet and Deployment? How do you use OIDC in GitHub Actions instead of static AWS credentials?

**Phase 7** — How do you measure search quality? What is Recall@K? How do you tune efSearch? How do you correlate a slow request across metrics, traces, and logs?