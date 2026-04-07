# 2. Similarity Metrics — How Do We Measure "Closeness"?

---

## ELI5

You have two drawings. How do you decide if they look alike?

- **Option A (Cosine)**: Do they point in the same direction? Ignore how big or small they are — just check the angle. Two arrows pointing the same way = similar.
- **Option B (L2 / Euclidean)**: How far apart are the tips? Measure with a ruler. Closer tips = more similar.
- **Option C (Dot Product)**: How much do they overlap when you lay one on top of the other? More overlap = more similar.

Each method gives slightly different answers. The "best" one depends on your data.

---

## The Three Core Metrics

### 1. Cosine Similarity

**What it measures**: The angle between two vectors (ignoring magnitude).

```
           B
          /
         / θ (angle)
        /
       A----------->

Small angle θ → high similarity
Large angle θ → low similarity
```

**Formula**:

```
cosine_similarity(A, B) = (A · B) / (||A|| × ||B||)

Where:
  A · B    = dot product = sum of (a_i × b_i)
  ||A||    = magnitude = sqrt(sum of a_i²)
```

**Range**: -1 to +1
- `1.0` = identical direction (same meaning)
- `0.0` = perpendicular (unrelated)
- `-1.0` = opposite direction (opposite meaning)

**Example**:

```
A = [1, 2, 3]
B = [2, 4, 6]    (same direction, different magnitude)
C = [3, 1, 0]    (different direction)

cosine(A, B) = (2+8+18) / (sqrt(14) × sqrt(56))
             = 28 / 28
             = 1.0  ← identical direction!

cosine(A, C) = (3+2+0) / (sqrt(14) × sqrt(10))
             = 5 / 11.83
             = 0.42  ← somewhat related
```

**When to use**: NLP embeddings (text). Most embedding models are trained with cosine similarity in mind.

**Critical rule**: If using cosine similarity, **normalize your vectors first**. Once normalized, cosine similarity = dot product (faster to compute).

---

### 2. Euclidean Distance (L2)

**What it measures**: Straight-line distance between two points.

```
    B •
    |  \
    |   \  distance
    |    \
    A •---•
```

**Formula**:

```
L2(A, B) = sqrt(sum of (a_i - b_i)²)
```

**Range**: 0 to infinity
- `0` = identical
- Higher = more different

**Note**: This is a **distance** (lower = more similar), not a **similarity** (higher = more similar). Some systems convert it:

```
similarity = 1 / (1 + distance)
```

**Example**:

```
A = [1, 2]
B = [4, 6]

L2 = sqrt((4-1)² + (6-2)²)
   = sqrt(9 + 16)
   = sqrt(25)
   = 5.0
```

**When to use**: When magnitude matters. Good for spatial data, image feature vectors, or when you care about the "absolute position" in the embedding space.

**Problem with L2 for text**: A longer document might have a larger magnitude vector. L2 would say it's "far" from a short document about the same topic, even though the meaning is similar. Cosine ignores this magnitude difference.

---

### 3. Dot Product (Inner Product)

**What it measures**: A combination of alignment AND magnitude.

**Formula**:

```
dot(A, B) = sum of (a_i × b_i) = a1×b1 + a2×b2 + ... + an×bn
```

**Range**: -infinity to +infinity
- Higher = more similar
- Can be negative

**Example**:

```
A = [1, 2, 3]
B = [4, 5, 6]

dot(A, B) = 1×4 + 2×5 + 3×6
          = 4 + 10 + 18
          = 32
```

**When to use**: When your model was trained with dot product (many Transformer models, including some OpenAI models). Also when magnitude carries useful information (e.g., a "more confident" embedding should rank higher).

**Key insight**: If vectors are normalized (length = 1), then:

```
dot product = cosine similarity
```

This is why normalization is so important — it unifies two metrics into one, and dot product is computationally cheaper.

---

## Side-by-Side Comparison

| Property | Cosine | L2 (Euclidean) | Dot Product |
|----------|--------|----------------|-------------|
| Measures | Angle | Distance | Alignment + Magnitude |
| Range | [-1, 1] | [0, ∞) | (-∞, +∞) |
| Similar = | High value (→1) | Low value (→0) | High value |
| Ignores magnitude? | Yes | No | No |
| Best for | NLP / text | Spatial data | Transformers |
| Normalized vectors | = Dot Product | Works fine | = Cosine |

---

## Deep Dive: Why Cosine Dominates NLP

### The Magnitude Problem

Consider two documents about machine learning:
- Doc A: A 50-word summary → embedding magnitude = 2.1
- Doc B: A 5000-word textbook chapter → embedding magnitude = 8.7

With **L2 distance**: They'd appear far apart because of magnitude difference, even though the topic is the same.

With **Cosine similarity**: Magnitude is divided out. Only the direction matters. Both point "toward machine learning" in the embedding space → high similarity.

### Visualization (2D simplified)

```
        ^
        |     B (5,5) — large magnitude
        |    /
        |   /
        |  /
        | / ← same angle = high cosine similarity
        |/
   A (1,1) — small magnitude
   -----+------------>

Cosine(A, B) = 1.0 (identical direction)
L2(A, B) = sqrt(16+16) = 5.66 (appears distant!)
```

---

## Deep Dive: When L2 Beats Cosine

### Spatial / Geographic Data

```
Location A = [40.7128, -74.0060]  (New York)
Location B = [40.7580, -73.9855]  (Midtown Manhattan)
Location C = [34.0522, -118.2437] (Los Angeles)

L2(A, B) = 0.05    ← nearby (correct!)
L2(A, C) = 44.89   ← far away (correct!)

Cosine(A, B) = 0.9999  ← similar
Cosine(A, C) = 0.9998  ← also similar?! (wrong — they're far apart)
```

Cosine fails here because all US coordinates "point in roughly the same direction" from origin. L2 correctly captures the absolute distance.

### Image Feature Vectors

When comparing raw pixel features or CNN outputs where magnitude = intensity/confidence, L2 is often more appropriate.

---

## Deep Dive: Dot Product and Maximum Inner Product Search (MIPS)

Some models (especially newer Transformer-based models) are explicitly trained to maximize the dot product between relevant query-document pairs. In these cases:

- A high dot product means "relevant"
- The magnitude encodes "confidence" or "importance"

This leads to **Maximum Inner Product Search (MIPS)** — a specific type of nearest neighbor search optimized for dot product.

**Example**: In a recommendation system, item embeddings might have larger magnitudes for more "popular" items. Dot product naturally boosts these, which might be desirable.

---

## Practical Decision Flowchart

```
Start
  │
  ├─ Using text embeddings (OpenAI, BGE, Sentence Transformers)?
  │    → Use COSINE similarity
  │    → Normalize vectors
  │    → (Internally, use dot product for speed since normalized cosine = dot product)
  │
  ├─ Spatial data (GPS, pixel coordinates, physical measurements)?
  │    → Use L2 (Euclidean)
  │
  ├─ Model documentation says "use dot product"?
  │    → Use DOT PRODUCT
  │    → Don't normalize (magnitude is meaningful)
  │
  └─ Not sure?
       → Use COSINE (safest default for most embedding models)
```

---

## Cosine Distance vs Cosine Similarity

Be careful with terminology — some databases use **distance** (lower = better), others use **similarity** (higher = better):

```
cosine_distance = 1 - cosine_similarity

similarity = 0.85  →  distance = 0.15
similarity = 0.20  →  distance = 0.80
```

Check your vector DB's documentation. For example:
- **Qdrant**: Supports both `Cosine` (similarity) and can convert internally
- **Chroma**: Uses `cosine` distance by default (lower = closer)
- **FAISS**: Uses `IndexFlatIP` for dot product, `IndexFlatL2` for L2

---

## Performance Considerations

| Metric | Computation Cost | Notes |
|--------|-----------------|-------|
| Dot Product | Fastest | Single multiplication + sum |
| Cosine | Medium | Dot product + 2 magnitude calculations |
| L2 | Medium | Subtraction + squaring + square root |

**Optimization**: Pre-normalize your vectors at ingestion time. Then use dot product for search (which now equals cosine similarity). This gives you cosine semantics at dot product speed.

```python
# At ingestion time
normalized_vector = vector / np.linalg.norm(vector)
store(normalized_vector)

# At query time — use dot product (fast) to get cosine similarity results
results = index.search_by_dot_product(query_normalized)
```

---

## Summary

| Metric | One-Line | When to Use |
|--------|----------|-------------|
| Cosine Similarity | "Are they pointing the same way?" | Text embeddings (default choice) |
| L2 / Euclidean | "How far apart are they?" | Spatial data, when magnitude matters |
| Dot Product | "How aligned AND how strong?" | Transformer models, recommendations |

**Golden Rule**: Normalize vectors + use dot product = cosine similarity at maximum speed.

---

## Resources

- [Pinecone: Vector Similarity Explained](https://www.pinecone.io/learn/vector-similarity/) — interactive visualizations
- [Understanding Distance Metrics (Weaviate)](https://weaviate.io/blog/distance-metrics-in-vector-search) — practical comparison
- [3Blue1Brown: Dot Products](https://www.youtube.com/watch?v=LyGKycYT2v0) — beautiful visual intuition (YouTube)
- [Wikipedia: Cosine Similarity](https://en.wikipedia.org/wiki/Cosine_similarity) — mathematical deep dive
