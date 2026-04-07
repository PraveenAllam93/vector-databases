# 3. Retrieval & Re-Ranking — Finding the Best Context for the LLM

---

## ELI5

Imagine you ask a librarian for books about "training dogs."

**Step 1 — Retrieval** (fast, rough): The librarian quickly grabs 20 books from the "pets" section that seem relevant. Fast, but some might be about cats or hamsters.

**Step 2 — Re-ranking** (slow, precise): The librarian reads the back cover of each book and carefully re-orders them. The most relevant ones go to the top. Slow, but much more accurate.

This two-stage approach is how production RAG works. The vector database does the fast retrieval, then a re-ranker does the precise ordering.

---

## The Two-Stage Pipeline

```
Query: "How do I configure Docker networking?"
                    │
                    ▼
    ┌──────────────────────────────┐
    │  Stage 1: RETRIEVAL (fast)    │
    │  Vector DB → top 20 chunks    │
    │  ~5ms, approximate matching   │
    └──────────────┬───────────────┘
                    │
                    ▼
    ┌──────────────────────────────┐
    │  Stage 2: RE-RANKING (slow)   │
    │  Cross-encoder → reorder 20   │
    │  ~100-500ms, precise matching │
    └──────────────┬───────────────┘
                    │
                    ▼
    ┌──────────────────────────────┐
    │  Top 3-5 chunks → LLM         │
    └──────────────────────────────┘
```

**Why not just use the re-ranker on everything?**

```
Re-ranker on 1M documents: 1,000,000 × 50ms = 578 days
Re-ranker on 20 candidates:      20 × 50ms = 1 second

Vector DB narrows 1M → 20, then re-ranker picks the best 3-5.
```

---

## Stage 1: Retrieval

### Basic Retrieval

```python
from qdrant_client import QdrantClient
from sentence_transformers import SentenceTransformer

client = QdrantClient(host="localhost", port=6333)
model = SentenceTransformer("all-MiniLM-L6-v2")

def retrieve(query: str, collection: str, top_k: int = 20) -> list[dict]:
    """Retrieve top-K candidate chunks from vector DB."""
    query_vector = model.encode(query).tolist()

    results = client.query_points(
        collection_name=collection,
        query=query_vector,
        limit=top_k,
        with_payload=True,
    )

    return [
        {
            "id": point.id,
            "text": point.payload["text"],
            "score": point.score,
            "metadata": {k: v for k, v in point.payload.items() if k != "text"},
        }
        for point in results.points
    ]
```

### Retrieval with Query Expansion

Sometimes a single query misses relevant documents. Expand the query to cast a wider net:

```python
def retrieve_with_expansion(
    query: str,
    collection: str,
    top_k: int = 20,
    expand: bool = True,
) -> list[dict]:
    """Retrieve with multiple query variations for better recall."""
    queries = [query]

    if expand:
        # Simple expansion: add the query phrased differently
        # In production, use an LLM to generate variations
        expansions = generate_query_variations(query)
        queries.extend(expansions)

    all_results = {}
    for q in queries:
        results = retrieve(q, collection, top_k=top_k)
        for r in results:
            # Keep highest score per document
            if r["id"] not in all_results or r["score"] > all_results[r["id"]]["score"]:
                all_results[r["id"]] = r

    # Sort by score and return top_k
    ranked = sorted(all_results.values(), key=lambda x: x["score"], reverse=True)
    return ranked[:top_k]


def generate_query_variations(query: str) -> list[str]:
    """Generate query variations using an LLM."""
    from openai import OpenAI
    client = OpenAI()

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Generate 3 alternative phrasings of the search query. Return one per line, no numbering."},
            {"role": "user", "content": query},
        ],
        temperature=0.7,
    )

    return response.choices[0].message.content.strip().split("\n")
```

### Multi-Query Retrieval

Decompose complex queries into sub-queries:

```python
def multi_query_retrieve(query: str, collection: str, top_k: int = 20) -> list[dict]:
    """Decompose complex queries into sub-queries for better coverage."""
    from openai import OpenAI
    llm = OpenAI()

    # Decompose
    response = llm.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": (
                "Break the user's question into 2-3 simpler sub-questions "
                "that would help find all relevant information. "
                "Return one sub-question per line."
            )},
            {"role": "user", "content": query},
        ],
    )
    sub_queries = response.choices[0].message.content.strip().split("\n")
    sub_queries.append(query)  # include original

    # Retrieve for each sub-query
    all_results = {}
    for q in sub_queries:
        results = retrieve(q, collection, top_k=10)
        for r in results:
            if r["id"] not in all_results or r["score"] > all_results[r["id"]]["score"]:
                all_results[r["id"]] = r

    ranked = sorted(all_results.values(), key=lambda x: x["score"], reverse=True)
    return ranked[:top_k]
```

**Example:**

```
Original: "Compare Docker networking modes and their security implications"

Sub-queries:
  1. "What are Docker networking modes?"
  2. "Docker bridge vs host vs overlay network"
  3. "Docker network security best practices"

Each sub-query finds different relevant chunks → better coverage
```

---

## Stage 2: Re-Ranking

### Why Re-Ranking?

**Bi-encoder** (embedding model): Encodes query and document SEPARATELY. Fast but approximate.

```
Query embedding:    encode("Docker networking")     → [0.1, 0.3, ...]
Document embedding: encode("Container bridge mode") → [0.2, 0.4, ...]
Similarity: dot(query_emb, doc_emb) = 0.72
```

**Cross-encoder** (re-ranker): Encodes query AND document TOGETHER. Sees both at once, catches nuances.

```
Cross-encoder input: ["Docker networking", "Container bridge mode"]
                            ↓
                    [Transformer processes BOTH together]
                            ↓
                    Relevance score: 0.89
```

The cross-encoder can understand relationships between query and document that bi-encoders miss:

```
Query: "What causes Docker containers to lose network connectivity?"
Doc A: "Docker networking uses bridge mode by default" → bi-encoder: 0.78
Doc B: "DNS resolution failures cause container network issues" → bi-encoder: 0.65

After cross-encoder re-ranking:
Doc B: 0.92 (directly answers the question!)
Doc A: 0.61 (related but doesn't answer the question)
```

### Cross-Encoder Re-Ranking

```python
from sentence_transformers import CrossEncoder

# Load cross-encoder model
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query: str, candidates: list[dict], top_k: int = 5) -> list[dict]:
    """Re-rank candidates using a cross-encoder."""
    if not candidates:
        return []

    # Create query-document pairs
    pairs = [[query, c["text"]] for c in candidates]

    # Score all pairs
    scores = reranker.predict(pairs)

    # Attach scores and sort
    for candidate, score in zip(candidates, scores):
        candidate["rerank_score"] = float(score)

    reranked = sorted(candidates, key=lambda x: x["rerank_score"], reverse=True)
    return reranked[:top_k]
```

### Popular Re-Ranking Models

| Model | Size | Speed | Quality | Best For |
|-------|------|-------|---------|----------|
| cross-encoder/ms-marco-MiniLM-L-6-v2 | 80MB | Fast (~5ms/pair) | Good | General purpose |
| cross-encoder/ms-marco-TinyBERT-L-2-v2 | 17MB | Fastest | Decent | Latency-critical |
| BAAI/bge-reranker-v2-m3 | 560MB | Moderate | Excellent | Multilingual |
| Cohere Rerank API | Cloud | API call | Excellent | Production (no GPU) |
| mixedbread-ai/mxbai-rerank-large-v1 | 1.3GB | Slow | Best | Maximum quality |

### Cohere Rerank (Cloud API)

```python
import cohere

co = cohere.Client("your-api-key")

def rerank_cohere(query: str, candidates: list[dict], top_k: int = 5) -> list[dict]:
    """Re-rank using Cohere's API."""
    response = co.rerank(
        model="rerank-english-v3.0",
        query=query,
        documents=[c["text"] for c in candidates],
        top_n=top_k,
    )

    reranked = []
    for result in response.results:
        candidate = candidates[result.index]
        candidate["rerank_score"] = result.relevance_score
        reranked.append(candidate)

    return reranked
```

---

## Complete Retrieval Pipeline

```python
class RetrievalPipeline:
    """Two-stage retrieval: vector search → cross-encoder re-ranking."""

    def __init__(
        self,
        qdrant_client: QdrantClient,
        collection_name: str,
        embedding_model: str = "all-MiniLM-L6-v2",
        rerank_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
        retrieval_top_k: int = 20,
        rerank_top_k: int = 5,
    ):
        self.client = qdrant_client
        self.collection = collection_name
        self.retrieval_top_k = retrieval_top_k
        self.rerank_top_k = rerank_top_k

        from sentence_transformers import SentenceTransformer, CrossEncoder
        self.embedder = SentenceTransformer(embedding_model)
        self.reranker = CrossEncoder(rerank_model)

    def search(self, query: str, filter_params: dict | None = None) -> list[dict]:
        """Full retrieval pipeline: embed → search → re-rank."""
        # Stage 1: Vector retrieval
        query_vector = self.embedder.encode(query).tolist()

        search_kwargs = {
            "collection_name": self.collection,
            "query": query_vector,
            "limit": self.retrieval_top_k,
            "with_payload": True,
        }

        if filter_params:
            from qdrant_client.models import Filter, FieldCondition, MatchValue
            search_kwargs["query_filter"] = Filter(
                must=[
                    FieldCondition(key=k, match=MatchValue(value=v))
                    for k, v in filter_params.items()
                ]
            )

        results = self.client.query_points(**search_kwargs)

        candidates = [
            {
                "id": p.id,
                "text": p.payload["text"],
                "vector_score": p.score,
                "metadata": {k: v for k, v in p.payload.items() if k != "text"},
            }
            for p in results.points
        ]

        if not candidates:
            return []

        # Stage 2: Cross-encoder re-ranking
        pairs = [[query, c["text"]] for c in candidates]
        rerank_scores = self.reranker.predict(pairs)

        for candidate, score in zip(candidates, rerank_scores):
            candidate["rerank_score"] = float(score)

        reranked = sorted(candidates, key=lambda x: x["rerank_score"], reverse=True)
        return reranked[:self.rerank_top_k]
```

---

## Context Window Management

After retrieval, you need to fit the context into the LLM's window:

### Token Counting

```python
import tiktoken

def count_tokens(text: str, model: str = "gpt-4o") -> int:
    """Count tokens for a given text."""
    encoder = tiktoken.encoding_for_model(model)
    return len(encoder.encode(text))

def fit_context(chunks: list[dict], max_tokens: int = 4000) -> list[dict]:
    """Select chunks that fit within the token budget."""
    selected = []
    total_tokens = 0

    for chunk in chunks:
        chunk_tokens = count_tokens(chunk["text"])
        if total_tokens + chunk_tokens > max_tokens:
            break
        selected.append(chunk)
        total_tokens += chunk_tokens

    return selected
```

### Context Compression

Reduce chunk text to only the relevant parts:

```python
def compress_context(query: str, chunks: list[dict]) -> list[dict]:
    """Use LLM to extract only relevant sentences from each chunk."""
    from openai import OpenAI
    client = OpenAI()

    compressed = []
    for chunk in chunks:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": (
                    "Extract ONLY the sentences from the following text that are "
                    "relevant to answering the question. Return the relevant sentences "
                    "verbatim. If nothing is relevant, return 'NOT RELEVANT'."
                )},
                {"role": "user", "content": f"Question: {query}\n\nText: {chunk['text']}"},
            ],
            temperature=0,
        )

        compressed_text = response.choices[0].message.content
        if compressed_text != "NOT RELEVANT":
            compressed.append({**chunk, "text": compressed_text})

    return compressed
```

---

## Retrieval Evaluation

### Metrics

```python
def recall_at_k(retrieved_ids: list, relevant_ids: set, k: int) -> float:
    """What fraction of relevant docs did we find in top K?"""
    retrieved_set = set(retrieved_ids[:k])
    return len(retrieved_set & relevant_ids) / len(relevant_ids)

def precision_at_k(retrieved_ids: list, relevant_ids: set, k: int) -> float:
    """What fraction of retrieved docs are actually relevant?"""
    retrieved_set = set(retrieved_ids[:k])
    return len(retrieved_set & relevant_ids) / k

def mrr(retrieved_ids: list, relevant_ids: set) -> float:
    """Mean Reciprocal Rank — how high is the first relevant result?"""
    for i, doc_id in enumerate(retrieved_ids):
        if doc_id in relevant_ids:
            return 1.0 / (i + 1)
    return 0.0

def ndcg_at_k(retrieved_ids: list, relevance_scores: dict, k: int) -> float:
    """Normalized Discounted Cumulative Gain."""
    import numpy as np

    # DCG
    dcg = 0
    for i, doc_id in enumerate(retrieved_ids[:k]):
        rel = relevance_scores.get(doc_id, 0)
        dcg += rel / np.log2(i + 2)  # +2 because i is 0-indexed

    # Ideal DCG
    ideal_rels = sorted(relevance_scores.values(), reverse=True)[:k]
    idcg = sum(rel / np.log2(i + 2) for i, rel in enumerate(ideal_rels))

    return dcg / idcg if idcg > 0 else 0.0
```

### Quick Explanation of Each Metric

```
Query: "Docker networking"
Relevant docs: {D1, D3, D7}

Retrieved: [D3, D5, D1, D8, D7]

Recall@3:    {D3, D1} ∩ {D1, D3, D7} / 3 relevant = 2/3 = 0.67
             "We found 2 of 3 relevant docs in top 3"

Precision@3: {D3, D5, D1} ∩ {D1, D3, D7} / 3 retrieved = 2/3 = 0.67
             "2 of our top 3 results are relevant"

MRR:         First relevant doc is D3 at position 1 → 1/1 = 1.0
             "First relevant result was at position 1"

NDCG:        Accounts for graded relevance (some docs more relevant than others)
             and position (higher rank = more credit)
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Two-stage pipeline | Retrieve (fast, wide) → Re-rank (slow, precise) |
| Bi-encoder (Stage 1) | Encodes query and doc separately. Fast, approximate |
| Cross-encoder (Stage 2) | Encodes query + doc together. Slow, precise |
| Retrieval top-K | Fetch 20-50 candidates from vector DB |
| Re-rank top-K | Narrow to 3-5 best for the LLM |
| Query expansion | Generate query variations for better recall |
| Multi-query | Decompose complex queries into sub-questions |
| Context compression | Extract only relevant sentences before sending to LLM |
| Metrics | Recall@K, Precision@K, MRR, NDCG — always measure |

---

## Resources

- [Cross-Encoders vs Bi-Encoders (SBERT)](https://www.sbert.net/examples/applications/cross-encoder/README.html) — authoritative comparison
- [Cohere Rerank](https://docs.cohere.com/docs/reranking) — production re-ranking API
- [RAG Evaluation (RAGAS)](https://docs.ragas.io/) — framework for evaluating RAG systems
- [LlamaIndex Re-Ranking](https://docs.llamaindex.ai/en/stable/module_guides/querying/node_postprocessors/) — framework approach
- [Pinecone: Rerankers](https://www.pinecone.io/learn/series/rag/rerankers/) — practical guide
