# 5. Hands-On Exercises — Build Your Intuition

---

These exercises go from simple to complex. Each builds on the previous.

---

## Exercise 1: Generate and Visualize Embeddings

**Goal**: See how similar texts get similar vectors.

```python
"""
Exercise 1: Generate embeddings and measure similarity.
Run: uv run python exercises/01_embeddings_basic.py
Requires: pip install sentence-transformers numpy
"""
import numpy as np
from sentence_transformers import SentenceTransformer

# Load a small, fast model
model = SentenceTransformer("all-MiniLM-L6-v2")  # 384 dimensions

# Generate embeddings for different sentences
sentences = [
    "The cat sat on the mat",
    "A kitten was sitting on the rug",        # similar to sentence 1
    "Python is a programming language",        # different topic
    "JavaScript is used for web development",  # similar to sentence 3
    "The dog played in the park",              # animal-related like 1,2
]

embeddings = model.encode(sentences)

print(f"Shape: {embeddings.shape}")  # (5, 384)
print(f"Dimensions per embedding: {embeddings.shape[1]}")

# Compute cosine similarity between all pairs
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

print("\n--- Similarity Matrix ---")
for i in range(len(sentences)):
    for j in range(i + 1, len(sentences)):
        sim = cosine_similarity(embeddings[i], embeddings[j])
        print(f"  [{i}] vs [{j}]: {sim:.4f}  |  '{sentences[i][:30]}...' vs '{sentences[j][:30]}...'")
```

**What to observe**:
- Cat/kitten sentences have high similarity (~0.7-0.8)
- Python/JavaScript sentences have high similarity (~0.5-0.7)
- Cat vs Python sentences have low similarity (~0.1-0.2)

---

## Exercise 2: Normalization and Metric Comparison

**Goal**: See how normalization affects different similarity metrics.

```python
"""
Exercise 2: Compare cosine, L2, and dot product with/without normalization.
Run: uv run python exercises/02_metrics.py
"""
import numpy as np
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")

# Two similar sentences, one much longer
short = "Machine learning is great"
long = "Machine learning is a branch of artificial intelligence that focuses on building systems that learn from data. It encompasses supervised learning, unsupervised learning, and reinforcement learning approaches."

emb_short = model.encode(short)
emb_long = model.encode(long)

# Raw metrics
cosine = np.dot(emb_short, emb_long) / (np.linalg.norm(emb_short) * np.linalg.norm(emb_long))
l2 = np.linalg.norm(emb_short - emb_long)
dot = np.dot(emb_short, emb_long)

print("--- Raw (not normalized) ---")
print(f"Cosine similarity: {cosine:.4f}")
print(f"L2 distance:       {l2:.4f}")
print(f"Dot product:       {dot:.4f}")
print(f"Magnitude (short): {np.linalg.norm(emb_short):.4f}")
print(f"Magnitude (long):  {np.linalg.norm(emb_long):.4f}")

# Normalize
emb_short_norm = emb_short / np.linalg.norm(emb_short)
emb_long_norm = emb_long / np.linalg.norm(emb_long)

cosine_n = np.dot(emb_short_norm, emb_long_norm) / (np.linalg.norm(emb_short_norm) * np.linalg.norm(emb_long_norm))
dot_n = np.dot(emb_short_norm, emb_long_norm)

print("\n--- Normalized ---")
print(f"Cosine similarity: {cosine_n:.4f}")
print(f"Dot product:       {dot_n:.4f}")
print(f"Cosine ≈ Dot?      {abs(cosine_n - dot_n) < 0.0001}")
```

**What to observe**:
- After normalization, cosine similarity = dot product
- Magnitudes become 1.0 after normalization
- Sentence Transformers models usually output already-normalized vectors (verify!)

---

## Exercise 3: Indexing Algorithm Comparison with FAISS

**Goal**: See recall vs speed tradeoffs between Flat, IVF, and HNSW.

```python
"""
Exercise 3: Compare FAISS index types on synthetic data.
Run: uv run python exercises/03_indexing.py
Requires: pip install faiss-cpu numpy
"""
import time
import numpy as np
import faiss

# Generate synthetic data
np.random.seed(42)
n_vectors = 100_000
d = 128  # dimensions
data = np.random.random((n_vectors, d)).astype("float32")
queries = np.random.random((100, d)).astype("float32")
k = 10  # top-k

# --- Ground Truth (Flat / Brute Force) ---
index_flat = faiss.IndexFlatL2(d)
index_flat.add(data)

start = time.time()
gt_distances, gt_indices = index_flat.search(queries, k)
flat_time = time.time() - start
print(f"Flat (exact):  {flat_time*1000:.1f}ms for {len(queries)} queries")

# --- IVF ---
nlist = 100
index_ivf = faiss.IndexIVFFlat(faiss.IndexFlatL2(d), d, nlist)
index_ivf.train(data)
index_ivf.add(data)

for nprobe in [1, 5, 10, 50]:
    index_ivf.nprobe = nprobe
    start = time.time()
    distances, indices = index_ivf.search(queries, k)
    ivf_time = time.time() - start

    # Calculate recall
    recall = np.mean([
        len(set(gt_indices[i]) & set(indices[i])) / k
        for i in range(len(queries))
    ])
    print(f"IVF (nprobe={nprobe:2d}): {ivf_time*1000:.1f}ms, recall@{k}: {recall:.3f}")

# --- HNSW ---
for M in [16, 32, 64]:
    index_hnsw = faiss.IndexHNSWFlat(d, M)
    index_hnsw.hnsw.efConstruction = 200
    index_hnsw.add(data)

    for ef in [50, 100, 200]:
        index_hnsw.hnsw.efSearch = ef
        start = time.time()
        distances, indices = index_hnsw.search(queries, k)
        hnsw_time = time.time() - start

        recall = np.mean([
            len(set(gt_indices[i]) & set(indices[i])) / k
            for i in range(len(queries))
        ])
        print(f"HNSW (M={M:2d}, ef={ef:3d}): {hnsw_time*1000:.1f}ms, recall@{k}: {recall:.3f}")

print("\n--- Memory Usage ---")
print(f"Flat:  {index_flat.ntotal * d * 4 / 1024 / 1024:.1f} MB")
print(f"IVF:   {index_ivf.ntotal * d * 4 / 1024 / 1024:.1f} MB (+ centroid overhead)")
print(f"HNSW:  ~{index_hnsw.ntotal * d * 4 / 1024 / 1024 * 1.5:.1f} MB (+ graph overhead)")
```

**What to observe**:
- Flat is slowest but 100% recall
- IVF gets faster with lower nprobe but recall drops
- HNSW gives excellent recall even at low latency
- Higher M and efSearch = better recall but more memory/time

---

## Exercise 4: Your First Vector Database (Chroma)

**Goal**: Build a mini semantic search engine.

```python
"""
Exercise 4: Semantic search with Chroma.
Run: uv run python exercises/04_chroma_search.py
Requires: pip install chromadb sentence-transformers
"""
import chromadb
from sentence_transformers import SentenceTransformer

# Sample knowledge base
documents = [
    {"id": "1", "text": "Python is a high-level programming language known for its simplicity", "category": "programming"},
    {"id": "2", "text": "JavaScript is the language of the web, running in browsers", "category": "programming"},
    {"id": "3", "text": "Machine learning models learn patterns from data", "category": "ai"},
    {"id": "4", "text": "Neural networks are inspired by the human brain", "category": "ai"},
    {"id": "5", "text": "Docker containers package applications with their dependencies", "category": "devops"},
    {"id": "6", "text": "Kubernetes orchestrates container deployments at scale", "category": "devops"},
    {"id": "7", "text": "PostgreSQL is a powerful open-source relational database", "category": "database"},
    {"id": "8", "text": "Redis is an in-memory data store used for caching", "category": "database"},
    {"id": "9", "text": "Vector databases store embeddings for similarity search", "category": "database"},
    {"id": "10", "text": "REST APIs use HTTP methods to expose resources", "category": "programming"},
]

# Initialize
model = SentenceTransformer("all-MiniLM-L6-v2")
client = chromadb.PersistentClient(path="./chroma_exercise")

# Delete collection if it exists (for reruns)
try:
    client.delete_collection("knowledge_base")
except ValueError:
    pass

collection = client.create_collection(
    name="knowledge_base",
    metadata={"hnsw:space": "cosine"}
)

# Embed and store
embeddings = model.encode([doc["text"] for doc in documents]).tolist()

collection.add(
    ids=[doc["id"] for doc in documents],
    embeddings=embeddings,
    metadatas=[{"category": doc["category"]} for doc in documents],
    documents=[doc["text"] for doc in documents],
)

print(f"Stored {collection.count()} documents\n")

# --- Semantic Search ---
queries = [
    "How do I build web applications?",
    "What is deep learning?",
    "How to deploy applications to production?",
    "Which database should I use for caching?",
]

for query in queries:
    query_embedding = model.encode(query).tolist()

    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=3,
    )

    print(f"Query: '{query}'")
    for i, (doc, distance) in enumerate(zip(results["documents"][0], results["distances"][0])):
        similarity = 1 - distance  # Chroma returns cosine distance
        print(f"  {i+1}. [{similarity:.3f}] {doc}")
    print()

# --- Filtered Search ---
print("--- Filtered: only 'database' category ---")
query = "fast data storage"
query_embedding = model.encode(query).tolist()

results = collection.query(
    query_embeddings=[query_embedding],
    n_results=3,
    where={"category": "database"}
)

for i, (doc, distance) in enumerate(zip(results["documents"][0], results["distances"][0])):
    similarity = 1 - distance
    print(f"  {i+1}. [{similarity:.3f}] {doc}")
```

**What to observe**:
- "web applications" matches JavaScript (semantic, not keyword!)
- "deep learning" matches neural networks AND machine learning
- Metadata filtering restricts results to specific categories
- Similarity scores show how confident the match is

---

## Exercise 5: Chunking Strategies

**Goal**: See why chunking matters for long documents.

```python
"""
Exercise 5: Compare single-embedding vs chunked embeddings.
Run: uv run python exercises/05_chunking.py
Requires: pip install sentence-transformers numpy
"""
import numpy as np
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")

# A long document about multiple topics
long_document = """
Python was created by Guido van Rossum and first released in 1991.
It emphasizes code readability with its notable use of significant whitespace.
Python supports multiple programming paradigms including procedural, object-oriented, and functional programming.

Machine learning is a subset of artificial intelligence.
It uses statistical techniques to give computers the ability to learn from data.
Common algorithms include decision trees, random forests, and neural networks.

Docker is a platform for developing, shipping, and running applications in containers.
Containers allow developers to package an application with all dependencies.
Docker uses OS-level virtualization to deliver software in packages called containers.
"""

# Approach 1: Single embedding for the whole document
single_embedding = model.encode(long_document)

# Approach 2: Chunk by paragraph and embed each
paragraphs = [p.strip() for p in long_document.strip().split("\n\n") if p.strip()]
chunk_embeddings = model.encode(paragraphs)

# Query about Docker
query = "How do containers work?"
query_embedding = model.encode(query)

# Compare
sim_single = np.dot(query_embedding, single_embedding) / (
    np.linalg.norm(query_embedding) * np.linalg.norm(single_embedding)
)

print(f"Query: '{query}'")
print(f"\nSingle embedding similarity: {sim_single:.4f}")
print(f"\nChunked similarities:")
for i, (para, emb) in enumerate(zip(paragraphs, chunk_embeddings)):
    sim = np.dot(query_embedding, emb) / (
        np.linalg.norm(query_embedding) * np.linalg.norm(emb)
    )
    print(f"  Chunk {i+1}: {sim:.4f} | '{para[:60]}...'")

print(f"\nNotice: The Docker chunk scores highest ({np.max([np.dot(query_embedding, e) / (np.linalg.norm(query_embedding) * np.linalg.norm(e)) for e in chunk_embeddings]):.4f})")
print(f"        But single embedding dilutes the signal ({sim_single:.4f})")
```

**What to observe**:
- The single embedding for the whole document has moderate similarity to any specific query
- Individual chunks score much higher for their specific topic
- This is why we chunk documents before embedding — precision matters

---

## Next Steps

After completing these exercises, you should:

1. Understand what embeddings are and how similarity is measured
2. Know the tradeoffs between indexing algorithms
3. Be comfortable using Chroma for semantic search
4. Understand why chunking is critical

Move on to **Phase 2** to set up Qdrant with Docker and build a production-quality search system.
