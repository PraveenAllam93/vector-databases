# 4. RAG with FastAPI — Building the Complete System

---

## ELI5

We've learned how to store documents (ingestion), find relevant ones (retrieval), and pick the best matches (re-ranking). Now we wire it all together into an API that:

1. Takes a user's question
2. Finds relevant context from the vector DB
3. Sends context + question to an LLM
4. Returns a grounded answer with sources

This is a complete RAG (Retrieval-Augmented Generation) system exposed via a FastAPI web server.

---

## Architecture

```
User Question
      │
      ▼
┌─────────────────┐
│   FastAPI API     │
│   POST /ask       │
└────────┬──────────┘
         │
    ┌────▼────┐
    │  Query   │──── Optional: rewrite query
    │  Layer   │──── Optional: expand to sub-queries
    └────┬────┘
         │
    ┌────▼─────────────┐
    │   Retrieval        │
    │   (Qdrant search)  │──── Top 20 candidates
    └────┬───────────────┘
         │
    ┌────▼─────────────┐
    │   Re-ranking       │
    │   (Cross-encoder)  │──── Top 5 re-ranked
    └────┬───────────────┘
         │
    ┌────▼─────────────┐
    │   Generation       │
    │   (LLM + context)  │──── Grounded answer
    └────┬───────────────┘
         │
         ▼
    Response + Sources
```

---

## Project Structure

```
rag_api/
├── main.py              # FastAPI app entry point
├── config.py            # Settings and configuration
├── models.py            # Pydantic request/response models
├── services/
│   ├── embedder.py      # Embedding service
│   ├── retriever.py     # Vector DB retrieval + re-ranking
│   ├── generator.py     # LLM generation
│   └── ingestion.py     # Document ingestion
├── routers/
│   ├── ask.py           # /ask endpoint
│   └── ingest.py        # /ingest endpoint
└── docker-compose.yml   # Qdrant + API
```

---

## Configuration

```python
# rag_api/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Qdrant
    qdrant_host: str = "localhost"
    qdrant_port: int = 6333
    collection_name: str = "documents"

    # Embedding
    embedding_model: str = "all-MiniLM-L6-v2"
    embedding_dim: int = 384

    # Retrieval
    retrieval_top_k: int = 20
    rerank_top_k: int = 5
    rerank_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"

    # LLM
    openai_api_key: str = ""
    llm_model: str = "gpt-4o-mini"
    max_context_tokens: int = 3000
    temperature: float = 0.1

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## Pydantic Models

```python
# rag_api/models.py
from pydantic import BaseModel

class AskRequest(BaseModel):
    question: str
    filter: dict | None = None  # optional metadata filters
    top_k: int = 5

class Source(BaseModel):
    text: str
    score: float
    source_doc: str
    chunk_index: int | None = None

class AskResponse(BaseModel):
    answer: str
    sources: list[Source]
    query: str

class IngestRequest(BaseModel):
    text: str
    source_doc: str
    doc_id: str | None = None
    metadata: dict | None = None

class IngestResponse(BaseModel):
    doc_id: str
    chunks_stored: int
```

---

## Embedding Service

```python
# rag_api/services/embedder.py
from sentence_transformers import SentenceTransformer
from config import settings

class EmbeddingService:
    def __init__(self):
        self.model = SentenceTransformer(settings.embedding_model)

    def embed(self, text: str) -> list[float]:
        """Embed a single text."""
        return self.model.encode(text).tolist()

    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        """Embed a batch of texts."""
        embeddings = self.model.encode(texts, batch_size=64, show_progress_bar=False)
        return embeddings.tolist()

# Singleton — load model once
embedder = EmbeddingService()
```

---

## Retrieval Service

```python
# rag_api/services/retriever.py
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue
from sentence_transformers import CrossEncoder
from config import settings
from services.embedder import embedder

class RetrievalService:
    def __init__(self):
        self.client = QdrantClient(
            host=settings.qdrant_host,
            port=settings.qdrant_port,
        )
        self.reranker = CrossEncoder(settings.rerank_model)

    def search(
        self,
        query: str,
        top_k: int = 5,
        filters: dict | None = None,
    ) -> list[dict]:
        """Two-stage retrieval: vector search → re-rank."""
        # Stage 1: Vector search
        query_vector = embedder.embed(query)

        search_kwargs = {
            "collection_name": settings.collection_name,
            "query": query_vector,
            "limit": settings.retrieval_top_k,
            "with_payload": True,
        }

        if filters:
            search_kwargs["query_filter"] = Filter(
                must=[
                    FieldCondition(key=k, match=MatchValue(value=v))
                    for k, v in filters.items()
                ]
            )

        results = self.client.query_points(**search_kwargs)

        candidates = [
            {
                "text": p.payload["text"],
                "vector_score": p.score,
                "source_doc": p.payload.get("source_doc", "unknown"),
                "chunk_index": p.payload.get("chunk_index"),
            }
            for p in results.points
        ]

        if not candidates:
            return []

        # Stage 2: Re-rank
        pairs = [[query, c["text"]] for c in candidates]
        scores = self.reranker.predict(pairs)

        for candidate, score in zip(candidates, scores):
            candidate["rerank_score"] = float(score)

        reranked = sorted(candidates, key=lambda x: x["rerank_score"], reverse=True)
        return reranked[:top_k]

retriever = RetrievalService()
```

---

## Generation Service

```python
# rag_api/services/generator.py
from openai import OpenAI
from config import settings

class GenerationService:
    def __init__(self):
        self.client = OpenAI(api_key=settings.openai_api_key)

    def generate(self, query: str, context_chunks: list[dict]) -> str:
        """Generate an answer using retrieved context."""
        # Build context string
        context_parts = []
        for i, chunk in enumerate(context_chunks, 1):
            source = chunk.get("source_doc", "unknown")
            context_parts.append(f"[Source {i}: {source}]\n{chunk['text']}")

        context = "\n\n---\n\n".join(context_parts)

        # System prompt
        system_prompt = """You are a helpful assistant that answers questions based on the provided context.

Rules:
- Answer ONLY based on the provided context
- If the context doesn't contain enough information, say "I don't have enough information to answer this question"
- Cite sources using [Source N] notation
- Be concise and direct
- Do not make up information"""

        user_prompt = f"""Context:
{context}

---

Question: {query}

Answer:"""

        response = self.client.chat.completions.create(
            model=settings.llm_model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt},
            ],
            temperature=settings.temperature,
            max_tokens=1000,
        )

        return response.choices[0].message.content

generator = GenerationService()
```

---

## API Endpoints

```python
# rag_api/routers/ask.py
from fastapi import APIRouter, HTTPException
from models import AskRequest, AskResponse, Source
from services.retriever import retriever
from services.generator import generator

router = APIRouter()

@router.post("/ask", response_model=AskResponse)
async def ask(request: AskRequest):
    """Ask a question and get a grounded answer with sources."""
    # Retrieve relevant chunks
    chunks = retriever.search(
        query=request.question,
        top_k=request.top_k,
        filters=request.filter,
    )

    if not chunks:
        return AskResponse(
            answer="I couldn't find any relevant information to answer your question.",
            sources=[],
            query=request.question,
        )

    # Generate answer
    answer = generator.generate(request.question, chunks)

    # Build sources
    sources = [
        Source(
            text=chunk["text"][:200] + "..." if len(chunk["text"]) > 200 else chunk["text"],
            score=chunk["rerank_score"],
            source_doc=chunk["source_doc"],
            chunk_index=chunk.get("chunk_index"),
        )
        for chunk in chunks
    ]

    return AskResponse(
        answer=answer,
        sources=sources,
        query=request.question,
    )
```

```python
# rag_api/routers/ingest.py
from fastapi import APIRouter, HTTPException
from models import IngestRequest, IngestResponse
from services.ingestion import ingestion_pipeline

router = APIRouter()

@router.post("/ingest", response_model=IngestResponse)
async def ingest(request: IngestRequest):
    """Ingest a document into the vector database."""
    result = ingestion_pipeline.ingest_text(
        text=request.text,
        source_doc=request.source_doc,
        doc_id=request.doc_id,
        extra_metadata=request.metadata,
    )

    return IngestResponse(
        doc_id=result["doc_id"],
        chunks_stored=result["chunks_stored"],
    )

@router.delete("/documents/{doc_id}")
async def delete_document(doc_id: str):
    """Delete all chunks for a document."""
    ingestion_pipeline.delete_document(doc_id)
    return {"status": "deleted", "doc_id": doc_id}
```

---

## Main Application

```python
# rag_api/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from routers import ask, ingest

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown events."""
    # Startup: models are loaded lazily in services
    print("RAG API starting up...")
    yield
    # Shutdown
    print("RAG API shutting down...")

app = FastAPI(
    title="RAG API",
    description="Retrieval-Augmented Generation API",
    version="0.1.0",
    lifespan=lifespan,
)

app.include_router(ask.router, tags=["Ask"])
app.include_router(ingest.router, tags=["Ingest"])

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

---

## Docker Compose — Full Stack

```yaml
# docker-compose.yml
services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - ./qdrant_storage:/qdrant/storage:z
    restart: unless-stopped

  rag-api:
    build: ./rag_api
    container_name: rag-api
    ports:
      - "8000:8000"
    environment:
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - qdrant
    restart: unless-stopped
```

```dockerfile
# rag_api/Dockerfile
FROM python:3.13-slim

WORKDIR /app

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

COPY pyproject.toml .
RUN uv sync --no-dev

COPY . .

CMD ["uv", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Caching with Redis

Add caching to avoid re-computing embeddings and re-querying for identical questions:

```python
# rag_api/services/cache.py
import json
import hashlib
import redis

class RAGCache:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.ttl = 3600  # 1 hour

    def _key(self, query: str, filters: dict | None = None) -> str:
        """Generate cache key from query + filters."""
        raw = json.dumps({"query": query, "filters": filters}, sort_keys=True)
        return f"rag:{hashlib.sha256(raw.encode()).hexdigest()}"

    def get(self, query: str, filters: dict | None = None) -> dict | None:
        """Get cached response."""
        cached = self.redis.get(self._key(query, filters))
        if cached:
            return json.loads(cached)
        return None

    def set(self, query: str, response: dict, filters: dict | None = None):
        """Cache a response."""
        self.redis.setex(
            self._key(query, filters),
            self.ttl,
            json.dumps(response),
        )

    def invalidate_by_doc(self, doc_id: str):
        """Invalidate all cache entries (simple approach: flush all)."""
        # In production, use more granular invalidation
        self.redis.flushdb()
```

### Integrate into the /ask endpoint:

```python
@router.post("/ask", response_model=AskResponse)
async def ask(request: AskRequest):
    # Check cache first
    cached = cache.get(request.question, request.filter)
    if cached:
        return AskResponse(**cached)

    # ... normal retrieval + generation ...

    response = AskResponse(answer=answer, sources=sources, query=request.question)

    # Cache the result
    cache.set(request.question, response.model_dump(), request.filter)

    return response
```

---

## Query Rewriting

Improve retrieval by rewriting the user's query before searching:

```python
# rag_api/services/query_rewriter.py
from openai import OpenAI

class QueryRewriter:
    def __init__(self):
        self.client = OpenAI()

    def rewrite(self, query: str, conversation_history: list[dict] | None = None) -> str:
        """Rewrite query for better retrieval."""
        messages = [
            {"role": "system", "content": (
                "Rewrite the user's question to be more specific and self-contained "
                "for a vector search system. Expand abbreviations, resolve pronouns "
                "using conversation history if provided, and add relevant context. "
                "Return ONLY the rewritten query, nothing else."
            )},
        ]

        # Add conversation history for context
        if conversation_history:
            for msg in conversation_history[-3:]:  # last 3 turns
                messages.append(msg)

        messages.append({"role": "user", "content": query})

        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            temperature=0,
            max_tokens=200,
        )

        return response.choices[0].message.content.strip()
```

**Example:**

```
Conversation:
  User: "Tell me about Docker"
  Assistant: "Docker is a containerization platform..."
  User: "How does it handle networking?"

Without rewriting: "How does it handle networking?"
  → Vector search finds generic networking docs

With rewriting: "How does Docker handle container networking?"
  → Vector search finds Docker-specific networking docs ✓
```

---

## Streaming Responses

For better UX, stream the LLM response:

```python
from fastapi.responses import StreamingResponse

@router.post("/ask/stream")
async def ask_stream(request: AskRequest):
    """Stream the answer as it's generated."""
    chunks = retriever.search(query=request.question, top_k=request.top_k)

    if not chunks:
        async def empty():
            yield "I couldn't find any relevant information."
        return StreamingResponse(empty(), media_type="text/plain")

    # Build context
    context = build_context(request.question, chunks)

    # Stream from LLM
    from openai import OpenAI
    client = OpenAI()

    stream = client.chat.completions.create(
        model=settings.llm_model,
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": context},
        ],
        temperature=settings.temperature,
        stream=True,
    )

    async def generate():
        for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

    return StreamingResponse(generate(), media_type="text/plain")
```

---

## Testing the API

```bash
# Start the stack
docker compose up -d

# Health check
curl http://localhost:8000/health

# Ingest a document
curl -X POST http://localhost:8000/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Docker uses containers to package applications. Containers share the host OS kernel and are more lightweight than VMs. Docker networking uses bridge mode by default, allowing containers to communicate on the same host.",
    "source_doc": "docker_guide.md"
  }'

# Ask a question
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "How does Docker networking work?"}'

# Ask with filter
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{
    "question": "How does Docker networking work?",
    "filter": {"source_doc": "docker_guide.md"}
  }'
```

---

## Summary

| Component | Role |
|-----------|------|
| FastAPI | HTTP API layer — /ask, /ingest, /health |
| Embedding Service | Converts text to vectors (sentence-transformers) |
| Retrieval Service | Vector search (Qdrant) + re-ranking (cross-encoder) |
| Generation Service | LLM generates grounded answers (OpenAI) |
| Query Rewriter | Improves queries with context resolution |
| Cache (Redis) | Avoid re-computing for identical queries |
| Streaming | Better UX for long answers |

---

## Resources

- [FastAPI Documentation](https://fastapi.tiangolo.com/) — comprehensive framework docs
- [OpenAI Cookbook: RAG](https://cookbook.openai.com/examples/how_to_build_a_rag_system) — official RAG guide
- [LangChain RAG Tutorial](https://python.langchain.com/docs/tutorials/rag/) — framework approach
- [LlamaIndex RAG](https://docs.llamaindex.ai/en/stable/understanding/rag/) — alternative framework
