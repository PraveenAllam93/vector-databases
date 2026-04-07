# 2. Ingestion Pipeline — Getting Data Into the Vector DB

---

## ELI5

Before you can search, you need to fill the database. The ingestion pipeline is like a factory assembly line:

```
Raw documents come in the door →
  Get cleaned up →
    Chopped into chunks →
      Stamped with metadata →
        Converted to numbers (embedded) →
          Stored in the vector DB
```

If the factory is sloppy (duplicates, stale data, wrong embeddings), everything downstream breaks.

---

## Pipeline Architecture

```
┌──────────┐    ┌───────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
│  Source   │───▶│ Parse &   │───▶│  Chunk   │───▶│  Embed    │───▶│  Store   │
│  (files,  │    │  Clean    │    │          │    │           │    │ (Qdrant) │
│   API,    │    │           │    │          │    │           │    │          │
│   DB)     │    └───────────┘    └──────────┘    └───────────┘    └──────────┘
└──────────┘         │                 │                │                │
                     ▼                 ▼                ▼                ▼
                Strip HTML        Add metadata    Batch embed       Upsert with
                Fix encoding      Track source    Rate limit        dedup check
                Extract text      Add chunk_id    Handle errors     Version tag
```

---

## Step 1: Parse & Clean

### Text Extraction

```python
from pathlib import Path

def extract_text(file_path: str) -> str:
    """Extract text from various file formats."""
    path = Path(file_path)
    suffix = path.suffix.lower()

    if suffix == ".txt":
        return path.read_text(encoding="utf-8")

    elif suffix == ".md":
        return path.read_text(encoding="utf-8")

    elif suffix == ".pdf":
        # pip install pymupdf
        import fitz
        doc = fitz.open(file_path)
        return "\n\n".join(page.get_text() for page in doc)

    elif suffix == ".html":
        from bs4 import BeautifulSoup
        html = path.read_text(encoding="utf-8")
        soup = BeautifulSoup(html, "html.parser")
        # Remove script and style elements
        for tag in soup(["script", "style", "nav", "footer"]):
            tag.decompose()
        return soup.get_text(separator="\n", strip=True)

    elif suffix == ".docx":
        # pip install python-docx
        import docx
        doc = docx.Document(file_path)
        return "\n\n".join(p.text for p in doc.paragraphs if p.text.strip())

    else:
        raise ValueError(f"Unsupported format: {suffix}")
```

### Text Cleaning

```python
import re
import unicodedata

def clean_text(text: str) -> str:
    """Clean extracted text for embedding."""
    # Normalize unicode (é → e, etc.)
    text = unicodedata.normalize("NFKD", text)

    # Fix common encoding issues
    text = text.encode("utf-8", errors="ignore").decode("utf-8")

    # Collapse multiple whitespace/newlines
    text = re.sub(r'\n{3,}', '\n\n', text)
    text = re.sub(r' {2,}', ' ', text)

    # Remove null bytes and control characters
    text = re.sub(r'[\x00-\x08\x0b\x0c\x0e-\x1f]', '', text)

    return text.strip()
```

---

## Step 2: Chunk with Metadata

```python
import hashlib
import uuid
from datetime import datetime, timezone

def create_chunks(
    text: str,
    source_doc: str,
    doc_id: str | None = None,
    chunk_size: int = 400,
    chunk_overlap: int = 80,
    embedding_model: str = "text-embedding-3-small",
) -> list[dict]:
    """Chunk text and attach metadata."""
    doc_id = doc_id or str(uuid.uuid4())

    # Use recursive splitting (from Phase 3, doc 01)
    raw_chunks = recursive_split(text, chunk_size=chunk_size, overlap=chunk_overlap)

    chunks = []
    for idx, chunk_text in enumerate(raw_chunks):
        # Create deterministic chunk ID (same content = same ID = idempotent)
        content_hash = hashlib.sha256(chunk_text.encode()).hexdigest()[:16]
        chunk_id = f"{doc_id}_{idx}_{content_hash}"

        chunks.append({
            "chunk_id": chunk_id,
            "text": chunk_text,
            "metadata": {
                "source_doc": source_doc,
                "doc_id": doc_id,
                "chunk_index": idx,
                "total_chunks": len(raw_chunks),
                "char_count": len(chunk_text),
                "content_hash": content_hash,
                "embedding_model": embedding_model,
                "ingested_at": datetime.now(timezone.utc).isoformat(),
            },
        })

    return chunks
```

### Why Deterministic Chunk IDs?

```python
# Content-based hash means:
# Same document → same chunks → same IDs → upsert is idempotent
#
# Re-running ingestion on the same doc doesn't create duplicates
# It just overwrites with identical data (no-op effectively)

chunk_id = f"{doc_id}_{chunk_index}_{content_hash}"
```

---

## Step 3: Embed

### Batch Embedding with Rate Limiting

```python
import time
from openai import OpenAI

def embed_chunks(
    chunks: list[dict],
    model: str = "text-embedding-3-small",
    batch_size: int = 100,
    max_retries: int = 3,
) -> list[dict]:
    """Embed chunks in batches with retry logic."""
    client = OpenAI()
    all_embeddings = []

    for i in range(0, len(chunks), batch_size):
        batch = chunks[i:i + batch_size]
        texts = [c["text"] for c in batch]

        for attempt in range(max_retries):
            try:
                response = client.embeddings.create(
                    model=model,
                    input=texts,
                )
                embeddings = [item.embedding for item in response.data]
                all_embeddings.extend(embeddings)
                break
            except Exception as e:
                if attempt < max_retries - 1:
                    wait = 2 ** attempt  # exponential backoff: 1s, 2s, 4s
                    print(f"Retry {attempt + 1}/{max_retries} after {wait}s: {e}")
                    time.sleep(wait)
                else:
                    raise

        print(f"Embedded {min(i + batch_size, len(chunks))}/{len(chunks)}")

    # Attach embeddings to chunks
    for chunk, embedding in zip(chunks, all_embeddings):
        chunk["embedding"] = embedding

    return chunks
```

### Local Embedding (No API Costs)

```python
from sentence_transformers import SentenceTransformer

def embed_chunks_local(chunks: list[dict], model_name: str = "all-MiniLM-L6-v2") -> list[dict]:
    """Embed chunks using a local model."""
    model = SentenceTransformer(model_name)

    texts = [c["text"] for c in chunks]
    embeddings = model.encode(texts, show_progress_bar=True, batch_size=64)

    for chunk, embedding in zip(chunks, embeddings):
        chunk["embedding"] = embedding.tolist()

    return chunks
```

---

## Step 4: Store in Qdrant

```python
from qdrant_client import QdrantClient
from qdrant_client.models import PointStruct

def store_chunks(
    client: QdrantClient,
    collection_name: str,
    chunks: list[dict],
    batch_size: int = 100,
) -> int:
    """Store embedded chunks in Qdrant."""
    points = [
        PointStruct(
            id=chunk["chunk_id"],
            vector=chunk["embedding"],
            payload={
                "text": chunk["text"],
                **chunk["metadata"],
            },
        )
        for chunk in chunks
    ]

    stored = 0
    for i in range(0, len(points), batch_size):
        batch = points[i:i + batch_size]
        client.upsert(collection_name=collection_name, points=batch)
        stored += len(batch)
        print(f"Stored {stored}/{len(points)}")

    return stored
```

---

## Complete Ingestion Pipeline

```python
"""
Complete ingestion pipeline — from files to searchable vectors.
"""
from pathlib import Path
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PayloadSchemaType

class IngestionPipeline:
    def __init__(
        self,
        qdrant_url: str = "localhost",
        qdrant_port: int = 6333,
        collection_name: str = "documents",
        embedding_model: str = "all-MiniLM-L6-v2",
        embedding_dim: int = 384,
        chunk_size: int = 400,
        chunk_overlap: int = 80,
    ):
        self.client = QdrantClient(host=qdrant_url, port=qdrant_port)
        self.collection_name = collection_name
        self.embedding_model_name = embedding_model
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap

        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer(embedding_model)

        self._ensure_collection(embedding_dim)

    def _ensure_collection(self, dim: int):
        """Create collection if it doesn't exist."""
        if not self.client.collection_exists(self.collection_name):
            self.client.create_collection(
                collection_name=self.collection_name,
                vectors_config=VectorParams(size=dim, distance=Distance.COSINE),
            )
            # Index frequently-filtered fields
            self.client.create_payload_index(
                self.collection_name, "source_doc", PayloadSchemaType.KEYWORD
            )
            self.client.create_payload_index(
                self.collection_name, "doc_id", PayloadSchemaType.KEYWORD
            )
            print(f"Created collection: {self.collection_name}")

    def ingest_file(self, file_path: str) -> int:
        """Ingest a single file into the vector DB."""
        # Parse
        text = extract_text(file_path)
        text = clean_text(text)

        if not text.strip():
            print(f"Skipping empty file: {file_path}")
            return 0

        # Chunk
        chunks = create_chunks(
            text,
            source_doc=Path(file_path).name,
            chunk_size=self.chunk_size,
            chunk_overlap=self.chunk_overlap,
            embedding_model=self.embedding_model_name,
        )

        # Embed
        chunks = embed_chunks_local(chunks, model_name=self.embedding_model_name)

        # Store
        return store_chunks(self.client, self.collection_name, chunks)

    def ingest_directory(self, dir_path: str, glob_pattern: str = "**/*.md") -> int:
        """Ingest all matching files in a directory."""
        files = list(Path(dir_path).glob(glob_pattern))
        total = 0

        for file_path in files:
            print(f"\nProcessing: {file_path}")
            count = self.ingest_file(str(file_path))
            total += count
            print(f"  → {count} chunks stored")

        print(f"\nTotal: {total} chunks from {len(files)} files")
        return total

    def delete_document(self, doc_id: str):
        """Delete all chunks from a specific document."""
        from qdrant_client.models import Filter, FieldCondition, MatchValue, FilterSelector

        self.client.delete(
            collection_name=self.collection_name,
            points_selector=FilterSelector(
                filter=Filter(
                    must=[FieldCondition(key="doc_id", match=MatchValue(value=doc_id))]
                )
            ),
        )
        print(f"Deleted all chunks for doc_id: {doc_id}")

    def re_ingest_document(self, file_path: str, doc_id: str):
        """Delete old chunks and re-ingest (for updates)."""
        self.delete_document(doc_id)
        return self.ingest_file(file_path)
```

---

## Deduplication Strategies

### Content-Based Dedup

```python
def is_duplicate(chunk_text: str, existing_hashes: set) -> bool:
    """Check if this chunk already exists."""
    content_hash = hashlib.sha256(chunk_text.encode()).hexdigest()[:16]
    if content_hash in existing_hashes:
        return True
    existing_hashes.add(content_hash)
    return False
```

### Semantic Dedup (Expensive but Thorough)

```python
def semantic_dedup(chunks: list[dict], threshold: float = 0.95) -> list[dict]:
    """Remove chunks that are too similar to each other."""
    import numpy as np

    kept = []
    kept_embeddings = []

    for chunk in chunks:
        emb = np.array(chunk["embedding"])

        is_dup = False
        for existing_emb in kept_embeddings:
            sim = np.dot(emb, existing_emb) / (
                np.linalg.norm(emb) * np.linalg.norm(existing_emb)
            )
            if sim > threshold:
                is_dup = True
                break

        if not is_dup:
            kept.append(chunk)
            kept_embeddings.append(emb)

    print(f"Dedup: {len(chunks)} → {len(kept)} chunks ({len(chunks) - len(kept)} removed)")
    return kept
```

---

## Embedding Versioning

**Why version?** When you switch embedding models (e.g., from `all-MiniLM-L6-v2` to `text-embedding-3-small`), old vectors become incompatible with new queries. You MUST re-embed everything.

### Strategy 1: Version in Metadata

```python
payload = {
    "text": "...",
    "embedding_model": "all-MiniLM-L6-v2",
    "embedding_version": "v1",
}

# When upgrading:
# 1. Ingest with new model → version "v2"
# 2. Query both v1 and v2 during transition
# 3. Delete v1 vectors once v2 is validated
```

### Strategy 2: Versioned Collections

```python
# More clean separation
collection_v1 = "documents_v1"  # all-MiniLM-L6-v2
collection_v2 = "documents_v2"  # text-embedding-3-small

# Blue-green deployment:
# 1. Build v2 collection in background
# 2. Validate recall quality
# 3. Switch traffic from v1 → v2
# 4. Delete v1
```

---

## Error Handling & Resilience

```python
import logging

logger = logging.getLogger(__name__)

def ingest_with_retry(pipeline, file_path: str, max_retries: int = 3) -> int:
    """Ingest a file with retry logic and error tracking."""
    for attempt in range(max_retries):
        try:
            return pipeline.ingest_file(file_path)
        except Exception as e:
            logger.error(f"Attempt {attempt + 1}/{max_retries} failed for {file_path}: {e}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
            else:
                logger.error(f"FAILED permanently: {file_path}")
                # In production: send to dead-letter queue, alert, etc.
                return 0
```

---

## Summary

| Stage | What Happens | Key Considerations |
|-------|-------------|-------------------|
| Parse | Extract text from files | Handle PDF, HTML, DOCX, Markdown |
| Clean | Normalize text | Unicode, whitespace, encoding |
| Chunk | Split into searchable pieces | 400 tokens, 80 overlap, recursive splitting |
| Metadata | Tag chunks with source info | doc_id, chunk_index, content_hash, model version |
| Embed | Convert to vectors | Batch, retry, rate limit |
| Store | Upsert into Qdrant | Batch upsert, idempotent via content-hash IDs |
| Dedup | Remove near-duplicates | Content hash or semantic similarity |
| Version | Track embedding model version | Critical for model upgrades |

---

## Resources

- [Unstructured.io](https://unstructured.io/) — universal document parser
- [LangChain Document Loaders](https://python.langchain.com/docs/how_to/#document-loaders) — 100+ file format loaders
- [LlamaIndex Ingestion Pipeline](https://docs.llamaindex.ai/en/stable/module_guides/loading/ingestion_pipeline/) — framework approach
