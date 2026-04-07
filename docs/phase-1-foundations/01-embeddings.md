# 1. Embeddings — The Foundation of Vector Databases

---

## ELI5 (Explain Like I'm 5)

Imagine you have a giant library. Instead of organizing books by author name (alphabetical), you organize them by **what they're about**. Books about dogs go near books about cats. Books about rockets go near books about space.

An **embedding** is like giving every book a special address in this magical library — not a shelf number, but a **position in space** where similar things are close together.

In computers, this "address" is a **list of numbers** (a vector). The computer learns these numbers so that similar meanings get similar numbers.

```
"king"  → [0.2, 0.8, 0.1, 0.5, ...]   (hundreds of numbers)
"queen" → [0.21, 0.79, 0.12, 0.48, ...] (very similar numbers!)
"car"   → [0.9, 0.1, 0.7, 0.3, ...]    (very different numbers)
```

---

## What Exactly is a Vector?

A **vector** is just an ordered list of numbers. That's it.

```
1D vector: [5]              → a point on a line
2D vector: [3, 4]           → a point on a plane (x=3, y=4)
3D vector: [1, 2, 3]        → a point in 3D space
768D vector: [0.1, 0.3, ...] → a point in 768-dimensional space
```

**Dimensionality** = how many numbers in the list.

Real-world embeddings typically have **384 to 3072 dimensions** depending on the model.

---

## What is an Embedding?

An embedding is a **learned vector representation** of data (text, image, audio) where:

1. **Similar items** have vectors that are **close together**
2. **Different items** have vectors that are **far apart**
3. **Relationships are preserved** — the direction between vectors captures meaning

### The Famous Example: Word Arithmetic

```
king - man + woman ≈ queen
```

This works because the embedding model learned that the difference between "king" and "man" captures the concept of "royalty", and applying that to "woman" gives "queen".

```
vector("king") - vector("man") + vector("woman") ≈ vector("queen")
[0.2, 0.8, 0.1] - [0.1, 0.7, 0.0] + [0.15, 0.72, 0.05] ≈ [0.25, 0.82, 0.15]
```

---

## How Are Embeddings Generated?

### Step 1: A Neural Network Learns Patterns

During training, a model (like a Transformer) reads millions/billions of text examples. It learns:
- "dog" and "puppy" appear in similar contexts → similar vectors
- "bank" (river) and "bank" (finance) appear in different contexts → different vectors

### Step 2: The Model Encodes Input into a Fixed-Size Vector

```
Input: "The quick brown fox jumps over the lazy dog"
                    ↓
          [Neural Network]
                    ↓
Output: [0.12, -0.45, 0.78, 0.33, ..., -0.21]  (768 dimensions)
```

No matter how long or short your input is, the output is always the **same number of dimensions**.

### Types of Embedding Models

| Model | Dimensions | Provider | Best For |
|-------|-----------|----------|----------|
| text-embedding-3-small | 1536 | OpenAI | General purpose, cost-effective |
| text-embedding-3-large | 3072 | OpenAI | Highest quality |
| BGE-large-en-v1.5 | 1024 | BAAI (open source) | Self-hosted, no API costs |
| all-MiniLM-L6-v2 | 384 | Sentence Transformers | Lightweight, fast |
| Instructor-XL | 768 | Open source | Task-specific instructions |
| Cohere embed-v3 | 1024 | Cohere | Multilingual |

---

## Deep Dive: How Transformer Embeddings Work

### Tokenization

Text is first split into **tokens** (sub-words):

```
"embeddings are cool" → ["embed", "##dings", "are", "cool"]
```

Each token gets an initial embedding from a lookup table (random at first, learned during training).

### Self-Attention

The transformer's self-attention mechanism lets each token "look at" every other token to understand context:

```
"I went to the bank to deposit money"
  → "bank" attends to "deposit" and "money"
  → embedding of "bank" shifts toward "financial institution"

"I sat on the river bank"
  → "bank" attends to "river"
  → embedding of "bank" shifts toward "edge of river"
```

### Pooling

After the transformer processes all tokens, we need ONE vector for the whole sentence. Common strategies:

1. **CLS token**: Use the special [CLS] token's output (BERT-style)
2. **Mean pooling**: Average all token vectors (most common, usually best)
3. **Max pooling**: Take element-wise max across all tokens

```
Token embeddings: [[0.1, 0.3, 0.5], [0.2, 0.4, 0.6], [0.3, 0.5, 0.7]]

Mean pooling: [(0.1+0.2+0.3)/3, (0.3+0.4+0.5)/3, (0.5+0.6+0.7)/3]
            = [0.2, 0.4, 0.6]
```

---

## What Can Be Embedded?

| Data Type | Example | Common Models |
|-----------|---------|---------------|
| Text (words, sentences, documents) | "How to train a dog" | OpenAI, BGE, Sentence Transformers |
| Images | A photo of a dog | CLIP, ResNet |
| Audio | A dog barking sound | Whisper embeddings, CLAP |
| Code | `def hello(): print("hi")` | CodeBERT, StarCoder |
| Multi-modal | Text + Image together | CLIP, BLIP-2 |

---

## Key Properties of Good Embeddings

### 1. Semantic Similarity is Preserved

```
similarity("happy", "joyful") = 0.92   (high — similar meaning)
similarity("happy", "sad")    = 0.31   (low — opposite meaning)
similarity("happy", "table")  = 0.05   (very low — unrelated)
```

### 2. Dense, Not Sparse

**Sparse** (old approach — bag of words):
```
"cat sat" → [0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0]  (mostly zeros, 10000+ dims)
```

**Dense** (embeddings):
```
"cat sat" → [0.12, -0.45, 0.78, 0.33, -0.21, ...]  (every dimension has info, 384-3072 dims)
```

Dense embeddings are:
- More compact (less memory)
- Capture semantic meaning (sparse can't)
- Enable fuzzy/semantic matching

### 3. Normalization Matters

**Normalization** = scaling a vector so its length (magnitude) = 1.

```
Original:    [3, 4]     → magnitude = sqrt(9 + 16) = 5
Normalized:  [0.6, 0.8] → magnitude = sqrt(0.36 + 0.64) = 1
```

**Why normalize?**
- Cosine similarity assumes normalized vectors
- Without normalization, longer documents get artificially higher scores
- Most embedding models output normalized vectors, but always verify

```python
import numpy as np

vector = np.array([3, 4])
normalized = vector / np.linalg.norm(vector)
# [0.6, 0.8]
```

---

## Embedding Dimensions: Why Does Size Matter?

| Dimensions | Tradeoff |
|-----------|----------|
| Low (128-384) | Faster search, less memory, less detail |
| Medium (512-1024) | Good balance |
| High (1536-3072) | More detail, slower search, more memory |

**Rule of thumb**: Start with the model's default dimensions. Only reduce if you hit memory/latency constraints.

OpenAI's `text-embedding-3` models support **dimension reduction** via the API — you can request fewer dimensions:

```python
# Full 1536 dimensions
response = client.embeddings.create(model="text-embedding-3-small", input="hello")

# Reduced to 512 dimensions (faster, less memory, slightly less accurate)
response = client.embeddings.create(model="text-embedding-3-small", input="hello", dimensions=512)
```

---

## Common Pitfalls

### 1. Mixing Embedding Models
Never store vectors from different models in the same collection. `text-embedding-3-small` and `BGE-large` produce incompatible vectors — even if dimensions match, the spaces are different.

### 2. Not Versioning Embeddings
When you upgrade your embedding model, ALL existing vectors become stale. You must re-embed everything. Track which model version generated each vector.

### 3. Ignoring Input Length Limits
Every model has a max token limit:

| Model | Max Tokens |
|-------|-----------|
| text-embedding-3-small | 8191 |
| BGE-large | 512 |
| all-MiniLM-L6-v2 | 256 |

Text beyond the limit is silently truncated — you lose information without any error.

### 4. Not Chunking Long Documents
A single embedding for a 10-page document loses detail. Instead:
- Split into chunks (256-512 tokens each)
- Embed each chunk separately
- Store all chunks with metadata linking back to the source document

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Vector | An ordered list of numbers |
| Embedding | A learned vector where similar meanings = close vectors |
| Dimensions | Number of values in the vector (384-3072 typical) |
| Generation | Neural networks (Transformers) encode input into fixed-size vectors |
| Normalization | Scale vectors to unit length — critical for cosine similarity |
| Pooling | Converting per-token embeddings into one sentence embedding |
| Dense vs Sparse | Embeddings are dense (all values meaningful) vs old bag-of-words (mostly zeros) |
| Versioning | Always track which model generated each embedding |

---

## Resources

- [What Are Word Embeddings?](https://www.tensorflow.org/text/guide/word_embeddings) — TensorFlow guide with visuals
- [Sentence Transformers Documentation](https://www.sbert.net/) — best open-source embedding library
- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings) — production API usage
- [Jay Alammar: The Illustrated Word2Vec](https://jalammar.github.io/illustrated-word2vec/) — visual explanation of how embeddings learn
- [Jay Alammar: The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) — how attention works under the hood
- [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) — benchmark comparing all embedding models
- [Pinecone: What Are Embeddings?](https://www.pinecone.io/learn/vector-embeddings/) — practical introduction
