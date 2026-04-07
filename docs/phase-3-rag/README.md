# Phase 3: RAG Pipeline

## Reading Order

| # | Topic | File | Key Takeaway |
|---|-------|------|-------------|
| 1 | [Chunking Strategies](01-chunking-strategies.md) | Fixed, recursive, sentence, semantic, structure, parent-child — tradeoffs and sizing |
| 2 | [Ingestion Pipeline](02-ingestion-pipeline.md) | Parse → clean → chunk → embed → store. Dedup, versioning, error handling |
| 3 | [Retrieval & Re-Ranking](03-retrieval-and-reranking.md) | Two-stage pipeline, cross-encoders, query expansion, evaluation metrics |
| 4 | [RAG with FastAPI](04-rag-with-fastapi.md) | Complete API: /ask + /ingest, caching, query rewriting, streaming |
| 5 | [Hands-On Exercises](05-hands-on-exercises.md) | 5 exercises building up to a working RAG API server |

## Setup

```bash
docker compose up -d
uv add qdrant-client sentence-transformers openai tiktoken fastapi uvicorn
export OPENAI_API_KEY="sk-..."
```

## Interview Quick Reference

**"Walk me through a RAG pipeline."**
→ User query → embed query → vector DB retrieves top-K candidates → cross-encoder re-ranks → top chunks become context → LLM generates answer grounded in that context → return answer with sources.

**"Why not just give the LLM all the documents?"**
→ Context window limits (cost, latency, lost-in-the-middle problem). Retrieval selects only relevant chunks. Re-ranking ensures the best ones are at the top.

**"What chunking strategy do you use?"**
→ Depends on the data. Recursive splitting at 400 tokens with 80 overlap is a good default. For high-quality RAG, semantic chunking (split where meaning changes) or parent-child (small chunks for matching, big chunks for context).

**"What is a cross-encoder and why re-rank?"**
→ Bi-encoders (embedding models) encode query and doc separately — fast but approximate. Cross-encoders see query + doc together — 10-100x slower but catch nuances like negation and causality. Use bi-encoder to narrow 1M→20, then cross-encoder to pick the best 5.

**"How do you evaluate RAG quality?"**
→ Retrieval: Recall@K, Precision@K, MRR, NDCG. Generation: faithfulness (does answer match context?), relevance (does answer address the question?). Use RAGAS framework for automated evaluation.

**"How do you handle model upgrades?"**
→ Embedding versioning. Track which model generated each vector. When upgrading: build new collection with new model, validate quality, blue-green switch, delete old collection.
