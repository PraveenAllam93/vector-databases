claude.md — Vector Database & RAG System (Production Blueprint)

Purpose

This document serves as a production-grade blueprint for building, scaling, and maintaining vector database systems and RAG pipelines.

⸻

1. Core Concept

A vector database stores high-dimensional embeddings and enables semantic similarity search using ANN (Approximate Nearest Neighbor) algorithms.

Pipeline

Raw Data → Embeddings → Index → Query → Retrieve → Re-rank → LLM → Response

⸻

2. System Architecture (Production)

High-Level Design
	•	API Layer (FastAPI)
	•	Embedding Service (OpenAI / BGE / Instructor)
	•	Vector DB (Qdrant / Milvus)
	•	Metadata Store (Postgres / Redis)
	•	Queue (Kafka / RabbitMQ)
	•	Cache Layer (Redis)
	•	Observability (Prometheus + Grafana + OpenTelemetry)

⸻

Deployment Pattern
	•	Docker + Kubernetes
	•	Horizontal scaling based on QPS
	•	Separate services:
	•	ingestion
	•	query
	•	indexing

⸻

3. Vector Database Selection

Decision Matrix

Use Case	Recommendation
Local Dev	Chroma
Mid-scale Prod	Qdrant
Large-scale	Milvus / Vespa
Research	FAISS


⸻

4. Indexing Deep Dive

HNSW (Default Choice)
	•	Graph-based ANN
	•	Logarithmic complexity

Tunables
	•	M: graph connectivity
	•	efConstruction: build quality
	•	efSearch: query accuracy vs latency

⸻

IVF
	•	Cluster-based
	•	Requires tuning nlist, nprobe

⸻

PQ / IVFPQ
	•	Memory optimization
	•	Used at scale (millions/billions vectors)

⸻

Flat Index
	•	Exact search
	•	Only for small datasets

⸻

5. Similarity Metrics

Metric	Use Case
Cosine	NLP embeddings
L2	Spatial
Dot Product	Transformers

Rule

Always normalize embeddings when using cosine similarity.

⸻

6. Data Flow (Critical)

Ingestion Pipeline
	•	Chunking strategy
	•	Embedding generation
	•	Deduplication
	•	Versioning (VERY IMPORTANT)
	•	Batch + streaming ingestion

Query Pipeline
	•	Query embedding
	•	ANN search
	•	Metadata filtering
	•	Re-ranking (cross encoder)

⸻

7. Hybrid Search (MANDATORY IN PROD)
	•	Combine:
	•	BM25 (keyword)
	•	Vector similarity

Improves precision significantly.

⸻

8. Race Conditions & Concurrency

Problems
	•	Concurrent writes
	•	Index rebuild conflicts
	•	Read-after-write inconsistency

Solutions
	•	Idempotent writes
	•	Write queues (Kafka)
	•	Versioned embeddings
	•	Snapshot isolation

⸻

9. Security

Authentication
	•	API Keys
	•	JWT
	•	OAuth2

Authorization
	•	RBAC
	•	Multi-tenant isolation (namespaces)

Encryption

At Rest
	•	AES-256
	•	Disk encryption (EBS / Persistent Disk)

In Transit
	•	TLS 1.2+
	•	mTLS internal

⸻

10. Observability

Metrics
	•	Latency (p95/p99)
	•	Recall@K
	•	Throughput
	•	Index build time

Tools
	•	Prometheus
	•	Grafana
	•	OpenTelemetry

Logging
	•	Query traces
	•	Failures
	•	Slow queries

⸻

11. Scaling Strategy

Horizontal Scaling
	•	Sharding by:
	•	user
	•	domain
	•	hash(vector_id)

Replication
	•	Leader-follower
	•	Multi-region

⸻

12. Storage & Backup
	•	Snapshot backups
	•	Incremental backups
	•	S3 / GCS storage

⸻

13. Performance Tuning

Key Knobs
	•	efSearch (HNSW)
	•	nprobe (IVF)

Tradeoff:
	•	Higher → better recall
	•	Lower → faster latency

⸻

14. RAG System Design

Architecture

User → Embed → Retrieve → Re-rank → LLM → Response

Enhancements
	•	Query rewriting
	•	Context compression
	•	Multi-query retrieval

⸻

15. Failure Scenarios

Issue	Cause	Fix
Low recall	Bad index params	Tune efSearch
High latency	Large search space	Reduce probes
Stale data	Missing versioning	Add version control
Memory blowup	No compression	Use PQ


⸻

16. Evaluation Metrics
	•	Recall@K
	•	Precision@K
	•	MRR
	•	NDCG

⸻

17. Best Practices
	•	Normalize vectors
	•	Use hybrid search
	•	Version embeddings
	•	Monitor recall vs latency
	•	Avoid brute force in prod

⸻

18. Example Production Stack
	•	Embeddings: OpenAI / BGE
	•	Vector DB: Qdrant / Milvus
	•	API: FastAPI
	•	Queue: Kafka
	•	Cache: Redis
	•	Observability: Prometheus + Grafana
	•	Storage: S3

⸻

19. Mental Model

“Vector DB = Semantic Search Engine optimized for high-dimensional similarity queries”

⸻

20. Next Steps
	•	Implement Qdrant locally
	•	Build RAG with FastAPI
	•	Add observability
	•	Benchmark recall vs latency

⸻

End of claude.md