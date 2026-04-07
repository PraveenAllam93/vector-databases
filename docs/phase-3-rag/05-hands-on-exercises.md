# 5. Hands-On Exercises — Build a Complete RAG System

---

## Prerequisites

```bash
docker compose up -d
uv add qdrant-client sentence-transformers openai tiktoken fastapi uvicorn
```

Set your OpenAI key: `export OPENAI_API_KEY="sk-..."`

(Exercises 1-3 work without OpenAI. Exercise 4-5 need it for the LLM.)

---

## Exercise 1: Chunking Comparison

**Goal**: See how different chunking strategies affect retrieval quality.

```python
"""
Exercise 1: Compare chunking strategies on real text.
Run: uv run python exercises/11_chunking_comparison.py
"""
import re
import numpy as np
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")

# A multi-topic document
document = """
# Docker Containers

Docker is a platform for containerization. It packages applications with their
dependencies into standardized units called containers. Containers are lightweight
because they share the host OS kernel, unlike virtual machines which need a full OS.

Docker images are built from Dockerfiles. Each instruction in a Dockerfile creates
a layer. Layers are cached, making rebuilds fast. The most common instructions are
FROM, RUN, COPY, and CMD.

## Docker Networking

Docker provides several networking modes. Bridge mode is the default — containers
get their own IP on a virtual bridge network. Host mode shares the host's network
stack directly. Overlay mode enables multi-host networking for Docker Swarm.

## Docker Volumes

Data in containers is ephemeral — it's lost when the container is removed. Volumes
solve this by persisting data outside the container lifecycle. Bind mounts map a
host directory into the container. Named volumes are managed by Docker itself.

# Kubernetes

Kubernetes (K8s) orchestrates container deployments at scale. It handles scheduling,
scaling, load balancing, and self-healing. The basic unit is a Pod, which wraps one
or more containers.

Services expose Pods to network traffic. A ClusterIP service is internal-only.
A NodePort service exposes on each node's IP. A LoadBalancer service provisions
an external load balancer in cloud environments.

## Kubernetes Scaling

Horizontal Pod Autoscaler (HPA) adjusts replica count based on CPU or custom metrics.
Vertical Pod Autoscaler (VPA) adjusts resource requests. Cluster Autoscaler adds
or removes nodes based on pending pods.
"""

# Strategy 1: Fixed-size (400 chars)
def fixed_chunk(text, size=400, overlap=80):
    chunks = []
    start = 0
    while start < len(text):
        chunks.append(text[start:start + size].strip())
        start += size - overlap
    return [c for c in chunks if c]

# Strategy 2: Paragraph-based
def paragraph_chunk(text):
    return [p.strip() for p in text.split("\n\n") if p.strip() and len(p.strip()) > 20]

# Strategy 3: Section-based (by headers)
def section_chunk(text):
    sections = re.split(r'(^#{1,2}\s+.+$)', text, flags=re.MULTILINE)
    chunks = []
    current = ""
    for part in sections:
        if re.match(r'^#{1,2}\s+', part):
            if current.strip():
                chunks.append(current.strip())
            current = part + "\n"
        else:
            current += part
    if current.strip():
        chunks.append(current.strip())
    return chunks

# Generate chunks
strategies = {
    "Fixed (400 chars)": fixed_chunk(document),
    "Paragraph": paragraph_chunk(document),
    "Section (headers)": section_chunk(document),
}

# Test queries
queries = [
    "How does Docker networking work?",
    "What is Kubernetes autoscaling?",
    "How do Docker volumes persist data?",
]

for query in queries:
    query_emb = model.encode(query)
    print(f"\n{'='*60}")
    print(f"Query: '{query}'")
    print(f"{'='*60}")

    for strategy_name, chunks in strategies.items():
        chunk_embs = model.encode(chunks)
        similarities = [
            np.dot(query_emb, emb) / (np.linalg.norm(query_emb) * np.linalg.norm(emb))
            for emb in chunk_embs
        ]
        best_idx = np.argmax(similarities)
        print(f"\n  {strategy_name}:")
        print(f"    Best score: {similarities[best_idx]:.4f}")
        print(f"    Best chunk: '{chunks[best_idx][:100]}...'")
        print(f"    Num chunks: {len(chunks)}")
```

---

## Exercise 2: Ingestion Pipeline

**Goal**: Build and run the complete ingestion pipeline.

```python
"""
Exercise 2: Ingest documents into Qdrant.
Run: uv run python exercises/12_ingestion.py
"""
import hashlib
import uuid
from datetime import datetime, timezone
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct, PayloadSchemaType,
)
from sentence_transformers import SentenceTransformer

client = QdrantClient(host="localhost", port=6333)
model = SentenceTransformer("all-MiniLM-L6-v2")

COLLECTION = "exercise_rag"

# Setup collection
if client.collection_exists(COLLECTION):
    client.delete_collection(COLLECTION)

client.create_collection(
    collection_name=COLLECTION,
    vectors_config=VectorParams(size=384, distance=Distance.COSINE),
)
client.create_payload_index(COLLECTION, "source_doc", PayloadSchemaType.KEYWORD)
client.create_payload_index(COLLECTION, "doc_id", PayloadSchemaType.KEYWORD)

# Sample documents (in real life, these come from files)
documents = {
    "docker_guide.md": """Docker is a platform for building and running containers.
Containers package applications with their dependencies into isolated environments.
They share the host OS kernel, making them lightweight compared to virtual machines.

Docker images are built from Dockerfiles using layered instructions.
Common instructions include FROM (base image), RUN (execute commands),
COPY (add files), and CMD (default command).

Docker Compose defines multi-container applications in a YAML file.
It manages service dependencies, networking, and volumes declaratively.
Run 'docker compose up' to start all services.""",

    "kubernetes_guide.md": """Kubernetes orchestrates container deployments at scale.
The basic deployment unit is a Pod, which wraps one or more containers.
Deployments manage Pod replicas and rolling updates.

Services expose Pods to network traffic using selectors.
ClusterIP is internal-only, NodePort exposes on each node,
and LoadBalancer provisions cloud load balancers.

ConfigMaps store configuration data as key-value pairs.
Secrets store sensitive data like passwords and API keys.
Both can be mounted as files or environment variables in Pods.""",

    "python_guide.md": """Python is a high-level programming language emphasizing readability.
It supports multiple paradigms: procedural, object-oriented, and functional.
Python uses dynamic typing and automatic memory management.

FastAPI is a modern Python web framework for building APIs.
It uses type hints for automatic validation and documentation.
Async support makes it suitable for high-concurrency applications.

Virtual environments isolate project dependencies.
Use 'python -m venv .venv' or 'uv venv' to create one.
Activate with 'source .venv/bin/activate' on macOS/Linux.""",
}

# Ingest each document
total_chunks = 0
for doc_name, text in documents.items():
    doc_id = str(uuid.uuid4())

    # Chunk by paragraph
    paragraphs = [p.strip() for p in text.strip().split("\n\n") if p.strip()]

    # Embed
    embeddings = model.encode(paragraphs)

    # Create points
    points = []
    for idx, (para, emb) in enumerate(zip(paragraphs, embeddings)):
        content_hash = hashlib.sha256(para.encode()).hexdigest()[:16]
        points.append(PointStruct(
            id=f"{doc_id}_{idx}_{content_hash}",
            vector=emb.tolist(),
            payload={
                "text": para,
                "source_doc": doc_name,
                "doc_id": doc_id,
                "chunk_index": idx,
                "total_chunks": len(paragraphs),
                "ingested_at": datetime.now(timezone.utc).isoformat(),
            },
        ))

    client.upsert(COLLECTION, points)
    total_chunks += len(points)
    print(f"Ingested {doc_name}: {len(points)} chunks")

print(f"\nTotal: {total_chunks} chunks in collection '{COLLECTION}'")

# Verify
info = client.get_collection(COLLECTION)
print(f"Collection has {info.points_count} points")

# Test a quick search
query = "How do I create a Docker container?"
query_vec = model.encode(query).tolist()

results = client.query_points(
    collection_name=COLLECTION,
    query=query_vec,
    limit=3,
)

print(f"\nTest search: '{query}'")
for p in results.points:
    print(f"  [{p.score:.4f}] [{p.payload['source_doc']}] {p.payload['text'][:80]}...")
```

---

## Exercise 3: Two-Stage Retrieval (Search + Re-rank)

**Goal**: See how re-ranking improves results.

```python
"""
Exercise 3: Compare retrieval with and without re-ranking.
Run: uv run python exercises/13_reranking.py
Requires: Exercise 2 data in Qdrant
"""
from qdrant_client import QdrantClient
from sentence_transformers import SentenceTransformer, CrossEncoder

client = QdrantClient(host="localhost", port=6333)
embedder = SentenceTransformer("all-MiniLM-L6-v2")
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

COLLECTION = "exercise_rag"

queries = [
    "How do I expose a service to the internet in Kubernetes?",
    "What's the difference between containers and virtual machines?",
    "How do I manage secrets in my application?",
]

for query in queries:
    print(f"\n{'='*60}")
    print(f"Query: '{query}'")
    print(f"{'='*60}")

    # Stage 1: Vector retrieval (top 10)
    query_vec = embedder.encode(query).tolist()
    results = client.query_points(
        collection_name=COLLECTION,
        query=query_vec,
        limit=10,
        with_payload=True,
    )

    candidates = [
        {"text": p.payload["text"], "score": p.score, "source": p.payload["source_doc"]}
        for p in results.points
    ]

    print("\n  --- Vector Search (bi-encoder) ---")
    for i, c in enumerate(candidates[:5]):
        print(f"  {i+1}. [{c['score']:.4f}] [{c['source']}] {c['text'][:70]}...")

    # Stage 2: Re-rank
    pairs = [[query, c["text"]] for c in candidates]
    rerank_scores = reranker.predict(pairs)

    for c, score in zip(candidates, rerank_scores):
        c["rerank_score"] = float(score)

    reranked = sorted(candidates, key=lambda x: x["rerank_score"], reverse=True)

    print("\n  --- After Re-ranking (cross-encoder) ---")
    for i, c in enumerate(reranked[:5]):
        print(f"  {i+1}. [{c['rerank_score']:.4f}] [{c['source']}] {c['text'][:70]}...")

    # Show position changes
    print("\n  --- Position Changes ---")
    for i, c in enumerate(reranked[:3]):
        old_pos = next(j for j, x in enumerate(candidates) if x["text"] == c["text"])
        if old_pos != i:
            print(f"  '{c['text'][:50]}...' moved from #{old_pos+1} → #{i+1}")
```

---

## Exercise 4: Complete RAG Pipeline

**Goal**: Wire retrieval into LLM generation for grounded answers.

```python
"""
Exercise 4: Full RAG — retrieve context, generate grounded answer.
Run: uv run python exercises/14_full_rag.py
Requires: OPENAI_API_KEY set, Exercise 2 data in Qdrant
"""
from qdrant_client import QdrantClient
from sentence_transformers import SentenceTransformer, CrossEncoder
from openai import OpenAI

# Setup
qdrant = QdrantClient(host="localhost", port=6333)
embedder = SentenceTransformer("all-MiniLM-L6-v2")
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
llm = OpenAI()

COLLECTION = "exercise_rag"

SYSTEM_PROMPT = """You are a helpful technical assistant. Answer questions based ONLY on the provided context.

Rules:
- Use ONLY information from the context to answer
- Cite sources using [Source N] notation
- If the context doesn't contain the answer, say so clearly
- Be concise"""


def rag_pipeline(question: str, top_k: int = 3) -> dict:
    """Complete RAG pipeline: retrieve → re-rank → generate."""

    # 1. Retrieve
    query_vec = embedder.encode(question).tolist()
    results = qdrant.query_points(
        collection_name=COLLECTION,
        query=query_vec,
        limit=15,
        with_payload=True,
    )

    candidates = [
        {"text": p.payload["text"], "source": p.payload["source_doc"], "score": p.score}
        for p in results.points
    ]

    # 2. Re-rank
    if candidates:
        pairs = [[question, c["text"]] for c in candidates]
        scores = reranker.predict(pairs)
        for c, s in zip(candidates, scores):
            c["rerank_score"] = float(s)
        candidates = sorted(candidates, key=lambda x: x["rerank_score"], reverse=True)

    top_chunks = candidates[:top_k]

    # 3. Build context
    context_parts = []
    for i, chunk in enumerate(top_chunks, 1):
        context_parts.append(f"[Source {i}: {chunk['source']}]\n{chunk['text']}")
    context = "\n\n---\n\n".join(context_parts)

    # 4. Generate
    response = llm.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": f"Context:\n{context}\n\n---\nQuestion: {question}"},
        ],
        temperature=0.1,
    )

    answer = response.choices[0].message.content

    return {
        "question": question,
        "answer": answer,
        "sources": [
            {"text": c["text"][:100] + "...", "source": c["source"], "score": c["rerank_score"]}
            for c in top_chunks
        ],
    }


# Test questions
questions = [
    "How do I define a multi-container application with Docker?",
    "What are the different ways to expose a Kubernetes service?",
    "How do I set up a Python development environment?",
    "What's the best way to store sensitive configuration in Kubernetes?",
]

for q in questions:
    print(f"\n{'='*60}")
    result = rag_pipeline(q)
    print(f"Q: {result['question']}")
    print(f"\nA: {result['answer']}")
    print(f"\nSources:")
    for s in result["sources"]:
        print(f"  [{s['score']:.3f}] {s['source']}: {s['text']}")
```

---

## Exercise 5: RAG API Server

**Goal**: Run the full RAG system as a FastAPI server.

```python
"""
Exercise 5: RAG as a FastAPI service.
Run: uv run uvicorn exercises.15_rag_api:app --reload --port 8000

Test:
  curl http://localhost:8000/health
  curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" \
    -d '{"question": "How do Docker containers work?"}'
"""
from contextlib import asynccontextmanager
from fastapi import FastAPI
from pydantic import BaseModel
from qdrant_client import QdrantClient
from sentence_transformers import SentenceTransformer, CrossEncoder
from openai import OpenAI


# --- Models ---
class AskRequest(BaseModel):
    question: str
    top_k: int = 3

class Source(BaseModel):
    text: str
    score: float
    source_doc: str

class AskResponse(BaseModel):
    answer: str
    sources: list[Source]


# --- Services (loaded once at startup) ---
services = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    services["qdrant"] = QdrantClient(host="localhost", port=6333)
    services["embedder"] = SentenceTransformer("all-MiniLM-L6-v2")
    services["reranker"] = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
    services["llm"] = OpenAI()
    print("All models loaded!")
    yield
    print("Shutting down...")


app = FastAPI(title="RAG API", lifespan=lifespan)

COLLECTION = "exercise_rag"


@app.get("/health")
async def health():
    count = services["qdrant"].count(COLLECTION)
    return {"status": "healthy", "documents": count.count}


@app.post("/ask", response_model=AskResponse)
async def ask(request: AskRequest):
    # Retrieve
    query_vec = services["embedder"].encode(request.question).tolist()
    results = services["qdrant"].query_points(
        collection_name=COLLECTION,
        query=query_vec,
        limit=15,
        with_payload=True,
    )

    candidates = [
        {"text": p.payload["text"], "source": p.payload["source_doc"], "score": p.score}
        for p in results.points
    ]

    # Re-rank
    if candidates:
        pairs = [[request.question, c["text"]] for c in candidates]
        scores = services["reranker"].predict(pairs)
        for c, s in zip(candidates, scores):
            c["rerank_score"] = float(s)
        candidates = sorted(candidates, key=lambda x: x["rerank_score"], reverse=True)

    top_chunks = candidates[:request.top_k]

    # Generate
    context = "\n\n---\n\n".join(
        f"[Source {i}: {c['source']}]\n{c['text']}"
        for i, c in enumerate(top_chunks, 1)
    )

    response = services["llm"].chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Answer based ONLY on the context. Cite [Source N]. Be concise."},
            {"role": "user", "content": f"Context:\n{context}\n\n---\nQuestion: {request.question}"},
        ],
        temperature=0.1,
    )

    return AskResponse(
        answer=response.choices[0].message.content,
        sources=[
            Source(
                text=c["text"][:150] + "...",
                score=c.get("rerank_score", c["score"]),
                source_doc=c["source"],
            )
            for c in top_chunks
        ],
    )
```

**Test it:**

```bash
# Terminal 1: Start the server
uv run uvicorn exercises.15_rag_api:app --reload --port 8000

# Terminal 2: Query it
curl -s http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "How do I manage multiple containers?"}' | python -m json.tool
```

---

## Next Steps

After these exercises you have a working RAG system. Move to **Phase 4** to learn about distributed systems, leader-follower replication, race conditions, and scaling.
