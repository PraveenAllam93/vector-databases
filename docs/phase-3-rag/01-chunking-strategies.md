# 1. Chunking Strategies — Breaking Documents Into Searchable Pieces

---

## ELI5

Imagine reading a 300-page textbook and someone asks "What is photosynthesis?" You don't hand them the whole book — you open to the right chapter, find the right paragraph, and point at it.

**Chunking** is how we break a big document into small, focused pieces so the vector database can find the RIGHT piece, not the whole book.

Too big = diluted meaning (embedding averages everything out).
Too small = lost context (a sentence alone might not make sense).
Just right = focused meaning with enough context.

---

## Why Chunking Matters

### The Dilution Problem

```
Full document embedding (10 pages about Python, Docker, AND ML):
  Query: "How does Docker work?"
  Similarity: 0.45 (moderate — the doc is about many things)

Chunked paragraph embedding (just the Docker section):
  Query: "How does Docker work?"
  Similarity: 0.89 (high — focused match!)
```

### The Context Problem

```
Sentence: "It uses a layered architecture."
  → What uses? Networking? Docker? Neural networks? No context.

Paragraph: "Docker images are built using a layered architecture.
            Each layer represents a Dockerfile instruction.
            Layers are cached and shared between images."
  → Clear: Docker images use layered architecture.
```

**Goal**: Find the sweet spot where each chunk is focused enough to match well, but has enough context to be useful.

---

## Chunking Methods

### 1. Fixed-Size Chunking (Simplest)

Split text into chunks of N tokens/characters with overlap.

```
Document: "A B C D E F G H I J K L M N O P"

Chunk size: 5, Overlap: 2

Chunk 1: "A B C D E"
Chunk 2: "D E F G H"      ← overlaps with chunk 1
Chunk 3: "G H I J K"      ← overlaps with chunk 2
Chunk 4: "J K L M N"
Chunk 5: "M N O P"
```

```python
def fixed_size_chunk(text: str, chunk_size: int = 500, overlap: int = 100) -> list[str]:
    """Split text into fixed-size character chunks with overlap."""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start += chunk_size - overlap
    return chunks
```

**Pros**: Simple, predictable size, works for any text.
**Cons**: Splits mid-sentence, mid-paragraph, mid-thought. No semantic awareness.

**When to use**: Quick prototype, uniform documents, when you don't know the structure.

---

### 2. Recursive Character Splitting (LangChain Default)

Try to split on natural boundaries, falling back to smaller separators:

```
Try splitting on: "\n\n" (paragraphs)
  → If chunks too big, try: "\n" (lines)
    → If still too big, try: ". " (sentences)
      → If still too big, try: " " (words)
        → Last resort: "" (characters)
```

```python
def recursive_split(
    text: str,
    chunk_size: int = 500,
    overlap: int = 100,
    separators: list[str] = ["\n\n", "\n", ". ", " ", ""],
) -> list[str]:
    """Recursively split text using natural boundaries."""
    chunks = []
    separator = separators[0]
    remaining_separators = separators[1:]

    # Split by current separator
    parts = text.split(separator) if separator else list(text)

    current_chunk = ""
    for part in parts:
        candidate = current_chunk + separator + part if current_chunk else part

        if len(candidate) <= chunk_size:
            current_chunk = candidate
        else:
            if current_chunk:
                chunks.append(current_chunk.strip())
            # If this single part is too big, recurse with smaller separators
            if len(part) > chunk_size and remaining_separators:
                chunks.extend(recursive_split(part, chunk_size, overlap, remaining_separators))
                current_chunk = ""
            else:
                current_chunk = part

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks
```

**Pros**: Respects natural boundaries (paragraphs, sentences). More coherent chunks.
**Cons**: Chunk sizes vary. Still not semantically aware.

**When to use**: General-purpose default. Works well for articles, docs, books.

---

### 3. Sentence-Based Chunking

Group sentences until reaching the size limit.

```python
import re

def sentence_chunk(text: str, max_sentences: int = 5, overlap_sentences: int = 1) -> list[str]:
    """Chunk by grouping sentences."""
    # Simple sentence splitting (use spaCy or nltk for production)
    sentences = re.split(r'(?<=[.!?])\s+', text)

    chunks = []
    for i in range(0, len(sentences), max_sentences - overlap_sentences):
        chunk = " ".join(sentences[i:i + max_sentences])
        if chunk.strip():
            chunks.append(chunk.strip())

    return chunks
```

**Pros**: Never splits mid-sentence. Natural reading flow.
**Cons**: Sentence detection can be tricky (abbreviations, decimals). Still arbitrary grouping.

---

### 4. Semantic Chunking (Smart)

Split where the MEANING changes. Embed each sentence, and split where consecutive sentence embeddings have low similarity.

```
Sentences and their similarity to next sentence:

"Docker uses containers."              → sim with next: 0.85
"Containers package applications."     → sim with next: 0.82
"They share the host OS kernel."       → sim with next: 0.25  ← TOPIC CHANGE
"Python is a programming language."    → sim with next: 0.78
"It supports multiple paradigms."      → sim with next: 0.80

Split at similarity drop:
  Chunk 1: "Docker uses containers. Containers package... They share..."
  Chunk 2: "Python is a programming language. It supports..."
```

```python
import numpy as np
from sentence_transformers import SentenceTransformer
import re

def semantic_chunk(
    text: str,
    model: SentenceTransformer,
    similarity_threshold: float = 0.5,
    max_chunk_size: int = 1000,
) -> list[str]:
    """Split text where semantic meaning changes."""
    sentences = re.split(r'(?<=[.!?])\s+', text)
    if len(sentences) <= 1:
        return [text]

    embeddings = model.encode(sentences)

    # Calculate similarity between consecutive sentences
    similarities = []
    for i in range(len(embeddings) - 1):
        sim = np.dot(embeddings[i], embeddings[i + 1]) / (
            np.linalg.norm(embeddings[i]) * np.linalg.norm(embeddings[i + 1])
        )
        similarities.append(sim)

    # Split at low-similarity points
    chunks = []
    current_chunk = [sentences[0]]

    for i, sim in enumerate(similarities):
        current_text = " ".join(current_chunk)
        if sim < similarity_threshold or len(current_text) > max_chunk_size:
            chunks.append(current_text)
            current_chunk = [sentences[i + 1]]
        else:
            current_chunk.append(sentences[i + 1])

    if current_chunk:
        chunks.append(" ".join(current_chunk))

    return chunks
```

**Pros**: Topic-aware splitting. Chunks are semantically coherent.
**Cons**: Slower (requires embedding every sentence). Threshold tuning needed.

**When to use**: High-quality RAG where retrieval precision matters most.

---

### 5. Document-Structure Chunking

Use the document's own structure (headings, sections, code blocks).

```python
import re

def markdown_chunk(text: str, max_chunk_size: int = 1000) -> list[dict]:
    """Chunk markdown by headers, preserving hierarchy."""
    sections = re.split(r'(^#{1,3}\s+.+$)', text, flags=re.MULTILINE)

    chunks = []
    current_header = ""
    current_content = ""

    for part in sections:
        if re.match(r'^#{1,3}\s+', part):
            # Save previous section
            if current_content.strip():
                chunks.append({
                    "header": current_header,
                    "content": current_content.strip(),
                    "text": f"{current_header}\n{current_content.strip()}" if current_header else current_content.strip(),
                })
            current_header = part.strip()
            current_content = ""
        else:
            current_content += part

    # Last section
    if current_content.strip():
        chunks.append({
            "header": current_header,
            "content": current_content.strip(),
            "text": f"{current_header}\n{current_content.strip()}" if current_header else current_content.strip(),
        })

    return chunks
```

**Pros**: Preserves document organization. Headers provide natural context.
**Cons**: Only works with structured documents (Markdown, HTML, code).

---

## Overlap: Why and How Much

Overlap ensures important information at chunk boundaries isn't lost:

```
Without overlap:
  Chunk 1: "...the authentication module uses JWT tokens."
  Chunk 2: "These tokens expire after 24 hours and must be refreshed..."

  Query: "How long do JWT tokens last?"
  → Chunk 2 matches but loses the JWT context from chunk 1

With overlap:
  Chunk 1: "...the authentication module uses JWT tokens."
  Chunk 2: "module uses JWT tokens. These tokens expire after 24 hours..."

  Query: "How long do JWT tokens last?"
  → Chunk 2 has full context ✓
```

**Guidelines**:

| Chunk Size | Recommended Overlap |
|-----------|-------------------|
| 200 tokens | 20-40 tokens (10-20%) |
| 500 tokens | 50-100 tokens (10-20%) |
| 1000 tokens | 100-200 tokens (10-20%) |

**Rule**: 10-20% overlap is the sweet spot. More wastes storage and can confuse retrieval with near-duplicates.

---

## Chunk Size: The Key Decision

### Too Small (< 100 tokens)

```
Chunk: "It uses containers."
Problem: No context. What uses containers? Docker? Kubernetes? Podman?
Result: High recall (matches many queries), low precision (matches wrong queries too)
```

### Too Large (> 1000 tokens)

```
Chunk: [Entire 2-page section about Docker, Kubernetes, AND serverless]
Problem: Embedding averages all three topics. Diluted signal.
Result: Low recall for specific queries
```

### Sweet Spot (200-500 tokens)

```
Chunk: "Docker uses containers to package applications with their
        dependencies. Containers share the host OS kernel, making
        them lightweight compared to VMs. A Docker image is built
        from a Dockerfile using layered instructions."
Result: Focused meaning, enough context, matches well ✓
```

### Practical Sizing Guide

| Use Case | Chunk Size | Why |
|----------|-----------|-----|
| Q&A / Support | 200-300 tokens | Precise answers, specific facts |
| Documentation | 300-500 tokens | Balance of context and precision |
| Legal / Research | 500-1000 tokens | Need full context around clauses |
| Code | By function/class | Natural boundaries |

---

## Metadata Enrichment

Always attach metadata to chunks — this powers filtered search and traceability:

```python
chunk = {
    # The text that gets embedded
    "text": "Docker uses containers to package applications...",

    # Metadata (stored as payload in Qdrant)
    "metadata": {
        "source_doc": "docker_guide.md",
        "doc_id": "doc_123",
        "chunk_index": 3,
        "total_chunks": 12,
        "section_header": "## Container Basics",
        "char_start": 1500,
        "char_end": 2100,
        "token_count": 85,
        "created_at": "2024-06-01T10:00:00Z",
        "embedding_model": "text-embedding-3-small",
        "embedding_version": "v2",
    }
}
```

**Critical metadata fields**:
- `source_doc` / `doc_id` → trace back to the original document
- `chunk_index` → reconstruct order, fetch neighboring chunks
- `section_header` → provides context about what section this chunk belongs to
- `embedding_model` + `embedding_version` → for re-embedding when you upgrade models

---

## Parent-Child Chunking (Advanced)

Store two levels: large "parent" chunks for context, small "child" chunks for precise matching.

```
Parent (1000 tokens): Full section about Docker containers
  ├── Child (200 tokens): "Docker uses containers to package..."
  ├── Child (200 tokens): "Containers share the host OS kernel..."
  └── Child (200 tokens): "Docker images are built from Dockerfiles..."

Search flow:
  1. Query matches Child 2 (precise match)
  2. Return Parent chunk to the LLM (full context)
```

```python
def parent_child_chunk(text: str, parent_size: int = 1000, child_size: int = 200) -> list[dict]:
    """Create parent-child chunk hierarchy."""
    # Create parent chunks
    parents = fixed_size_chunk(text, chunk_size=parent_size, overlap=0)

    all_chunks = []
    for parent_idx, parent_text in enumerate(parents):
        parent_id = f"parent_{parent_idx}"

        # Create child chunks within each parent
        children = fixed_size_chunk(parent_text, chunk_size=child_size, overlap=50)

        for child_idx, child_text in enumerate(children):
            all_chunks.append({
                "text": child_text,           # embed this (small, precise)
                "parent_text": parent_text,    # return this to LLM (big, contextual)
                "parent_id": parent_id,
                "child_index": child_idx,
            })

    return all_chunks
```

**Interview insight**: Parent-child chunking solves the precision-context tradeoff. You search with small chunks (precise matching) but feed the LLM large chunks (full context). This is a common production pattern.

---

## Summary

| Strategy | Complexity | Quality | When to Use |
|----------|-----------|---------|-------------|
| Fixed-size | Low | Moderate | Quick prototype |
| Recursive | Low | Good | General-purpose default |
| Sentence-based | Low | Good | Conversational content |
| Semantic | Medium | Excellent | High-quality RAG |
| Structure-based | Medium | Excellent | Structured docs (MD, HTML) |
| Parent-child | High | Best | Production systems needing precision + context |

**Default recommendation**: Start with recursive splitting at 400 tokens with 80 token overlap. Move to semantic or parent-child when you need better retrieval quality.

---

## Resources

- [Chunking Strategies (Pinecone)](https://www.pinecone.io/learn/chunking-strategies/) — visual comparison
- [LangChain Text Splitters](https://python.langchain.com/docs/how_to/#text-splitters) — practical implementations
- [Unstructured.io](https://unstructured.io/) — library for parsing PDFs, HTML, DOCX into chunks
- [Greg Kamradt: Chunking Experiments](https://www.youtube.com/watch?v=8OJC21T2SL4) — YouTube deep dive with benchmarks
