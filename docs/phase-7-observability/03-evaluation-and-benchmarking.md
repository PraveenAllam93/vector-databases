# 3. Evaluation & Benchmarking — Measuring What Matters

---

## ELI5

You can measure that your search is fast (latency). But is it any GOOD? Does it return the right documents? Evaluation measures quality. Benchmarking measures performance. You need both.

A fast system that returns garbage is worse than a slightly slow system that returns the right answers.

---

## Evaluation Metrics

### Recall@K

"Of all the relevant documents, how many did we find in the top K results?"

```
Formula: Recall@K = |Relevant ∩ Retrieved_top_K| / |Relevant|

Example:
  Relevant documents: {doc1, doc2, doc3, doc4, doc5}
  Retrieved top-5:    {doc1, doc2, doc6, doc7, doc8}

  Recall@5 = |{doc1, doc2}| / |{doc1, doc2, doc3, doc4, doc5}|
           = 2 / 5
           = 0.40

Target: Recall@10 > 0.85 for production RAG
```

### Precision@K

"Of the top K results, how many were actually relevant?"

```
Formula: Precision@K = |Relevant ∩ Retrieved_top_K| / K

Same example:
  Precision@5 = |{doc1, doc2}| / 5 = 0.40
```

### Mean Reciprocal Rank (MRR)

"Was the most relevant document near the top?"

```
Formula: MRR = (1/|Q|) × Σ (1 / rank_of_first_relevant_result)

Example:
  Query 1: first relevant result at rank 1 → 1/1 = 1.0
  Query 2: first relevant result at rank 3 → 1/3 = 0.33
  Query 3: first relevant result at rank 2 → 1/2 = 0.5

  MRR = (1.0 + 0.33 + 0.5) / 3 = 0.61
```

### NDCG@K (Normalized Discounted Cumulative Gain)

"Rewards placing highly relevant docs higher in the ranking."

```
DCG@K = Σ (relevance_i / log2(i + 1)) for i in 1..K

Where relevance can be graded (0=not relevant, 1=somewhat, 2=highly)
NDCG = DCG / IDCG (ideal DCG — best possible ordering)

NDCG is most comprehensive; use it for re-ranking evaluation.
```

---

## Building an Evaluation Dataset

```python
"""
Evaluation dataset format: (query, list of relevant doc IDs)
"""

# Option 1: Human-labeled (gold standard)
evaluation_set = [
    {
        "query": "How do I reset my password?",
        "relevant_ids": ["doc_123", "doc_456", "doc_789"],
    },
    {
        "query": "What is your refund policy?",
        "relevant_ids": ["doc_321", "doc_654"],
    },
]

# Option 2: LLM-generated (fast, imperfect)
import json
from openai import OpenAI

def generate_eval_set_with_llm(documents: list[dict], n_queries: int) -> list[dict]:
    """Use LLM to generate evaluation queries from documents."""
    client = OpenAI()
    eval_set = []

    for doc in documents[:n_queries]:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"""Given this document, generate a realistic search query that a user might type to find this document.
                
Document: {doc['text'][:500]}

Return JSON: {{"query": "...", "relevant_id": "{doc['id']}"}}"""
            }]
        )
        result = json.loads(response.choices[0].message.content)
        eval_set.append(result)

    return eval_set

# Option 3: Synthetic from existing search logs (best for production)
# Parse your search logs to find queries where users clicked a result
# That (query, clicked_result) pair is a weak label
```

---

## Running Evaluation

```python
from qdrant_client import QdrantClient
import numpy as np
from dataclasses import dataclass

@dataclass
class EvaluationResult:
    recall_at_k: float
    precision_at_k: float
    mrr: float
    ndcg_at_k: float
    avg_latency_ms: float
    p99_latency_ms: float

class Evaluator:
    def __init__(self, client: QdrantClient, collection_name: str, embedder):
        self.client = client
        self.collection_name = collection_name
        self.embedder = embedder

    def evaluate(self, eval_set: list[dict], k: int = 10) -> EvaluationResult:
        recalls = []
        precisions = []
        reciprocal_ranks = []
        ndcgs = []
        latencies = []

        for item in eval_set:
            query = item["query"]
            relevant_ids = set(item["relevant_ids"])

            # Embed query
            start = time.time()
            query_vec = self.embedder.embed(query)

            # Search
            results = self.client.query_points(
                collection_name=self.collection_name,
                query=query_vec,
                limit=k,
            )
            latencies.append((time.time() - start) * 1000)

            retrieved_ids = [str(r.id) for r in results.points]

            # Recall@K
            hits = len(set(retrieved_ids) & relevant_ids)
            recall = hits / len(relevant_ids) if relevant_ids else 0
            recalls.append(recall)

            # Precision@K
            precision = hits / k
            precisions.append(precision)

            # MRR
            rr = 0.0
            for rank, doc_id in enumerate(retrieved_ids, start=1):
                if doc_id in relevant_ids:
                    rr = 1.0 / rank
                    break
            reciprocal_ranks.append(rr)

            # NDCG@K
            ndcg = self._ndcg(retrieved_ids, relevant_ids, k)
            ndcgs.append(ndcg)

        return EvaluationResult(
            recall_at_k=np.mean(recalls),
            precision_at_k=np.mean(precisions),
            mrr=np.mean(reciprocal_ranks),
            ndcg_at_k=np.mean(ndcgs),
            avg_latency_ms=np.mean(latencies),
            p99_latency_ms=np.percentile(latencies, 99),
        )

    def _ndcg(self, retrieved_ids: list, relevant_ids: set, k: int) -> float:
        """Compute NDCG@K (binary relevance)."""
        # Actual DCG
        dcg = sum(
            1.0 / np.log2(rank + 1)
            for rank, doc_id in enumerate(retrieved_ids[:k], start=1)
            if doc_id in relevant_ids
        )
        # Ideal DCG (best possible ordering)
        ideal_hits = min(len(relevant_ids), k)
        idcg = sum(1.0 / np.log2(rank + 1) for rank in range(1, ideal_hits + 1))
        return dcg / idcg if idcg > 0 else 0.0

# Run evaluation
evaluator = Evaluator(client, "documents", embedder)
results = evaluator.evaluate(eval_set, k=10)

print(f"Recall@10:    {results.recall_at_k:.3f}")
print(f"Precision@10: {results.precision_at_k:.3f}")
print(f"MRR:          {results.mrr:.3f}")
print(f"NDCG@10:      {results.ndcg_at_k:.3f}")
print(f"Avg latency:  {results.avg_latency_ms:.1f}ms")
print(f"p99 latency:  {results.p99_latency_ms:.1f}ms")
```

---

## Load Testing (Benchmarking)

```python
# locustfile.py — load test with Locust
from locust import HttpUser, task, between
import json
import random

SAMPLE_QUERIES = [
    "How do I configure authentication?",
    "What are the performance tuning options?",
    "Explain the backup and restore process",
    "How does sharding work?",
    "What is the difference between HNSW and IVF?",
]

class VectorSearchUser(HttpUser):
    # Wait 0.1-1 second between requests (simulate real user)
    wait_time = between(0.1, 1.0)

    def on_start(self):
        """Called once per user at start."""
        self.headers = {
            "Authorization": f"Bearer {self.get_test_token()}",
            "Content-Type": "application/json",
        }

    def get_test_token(self) -> str:
        # Create a JWT for load testing
        return create_jwt("load-test-user", "load-test-tenant", ["search"])

    @task(10)   # weight: 10x more search than ingest
    def search(self):
        query = random.choice(SAMPLE_QUERIES)
        self.client.post(
            "/search",
            json={"text": query, "top_k": 10},
            headers=self.headers,
            name="/search",
        )

    @task(1)    # weight: 1x ingest
    def ingest(self):
        self.client.post(
            "/ingest",
            json={"text": f"Test document content {random.randint(1, 1000000)}"},
            headers=self.headers,
            name="/ingest",
        )

    @task(2)
    def health_check(self):
        self.client.get("/health/ready", name="/health/ready")
```

```bash
# Run load test
locust -f locustfile.py \
  --host http://localhost:8000 \
  --users 100 \         # 100 concurrent users
  --spawn-rate 10 \     # ramp up 10 users/second
  --run-time 5m \       # run for 5 minutes
  --headless \          # no web UI
  --csv results/load_test

# Check results
cat results/load_test_stats.csv
```

---

## Recall vs Latency Trade-off Benchmarking

```python
"""
Benchmark: how does efSearch affect recall AND latency?
Run this before choosing efSearch for production.
"""
import time
import numpy as np
from qdrant_client import QdrantClient
from qdrant_client.models import SearchParams

client = QdrantClient(host="localhost", port=6333)
COLLECTION = "benchmark"
N_QUERIES = 200
GROUND_TRUTH_K = 100   # get 100 results for ground truth

def benchmark_ef_search(ef_values: list[int], eval_queries: list, top_k: int = 10):
    results = {}

    # First get ground truth with very high ef (brute-force-like)
    ground_truth = {}
    for i, q_vec in enumerate(eval_queries):
        hits = client.query_points(
            collection_name=COLLECTION,
            query=q_vec,
            limit=GROUND_TRUTH_K,
            search_params=SearchParams(hnsw_ef=512),
        )
        ground_truth[i] = {str(h.id) for h in hits.points[:top_k]}

    for ef in ef_values:
        latencies = []
        recalls = []

        for i, q_vec in enumerate(eval_queries):
            start = time.time()
            hits = client.query_points(
                collection_name=COLLECTION,
                query=q_vec,
                limit=top_k,
                search_params=SearchParams(hnsw_ef=ef),
            )
            latencies.append((time.time() - start) * 1000)

            retrieved = {str(h.id) for h in hits.points}
            recall = len(retrieved & ground_truth[i]) / len(ground_truth[i])
            recalls.append(recall)

        results[ef] = {
            "recall": np.mean(recalls),
            "latency_p50": np.percentile(latencies, 50),
            "latency_p99": np.percentile(latencies, 99),
        }

    return results

# Run benchmark
ef_values = [16, 32, 64, 128, 256]
benchmark = benchmark_ef_search(ef_values, eval_query_vectors, top_k=10)

print(f"{'efSearch':>10} {'Recall@10':>12} {'p50 (ms)':>12} {'p99 (ms)':>12}")
print("-" * 50)
for ef, metrics in benchmark.items():
    print(f"{ef:>10} {metrics['recall']:>12.3f} {metrics['latency_p50']:>12.1f} {metrics['latency_p99']:>12.1f}")

# Output:
#  efSearch    Recall@10     p50 (ms)     p99 (ms)
# --------------------------------------------------
#        16        0.789          1.2          2.1
#        32        0.876          1.8          3.2
#        64        0.943          3.1          5.8
#       128        0.981          5.9         10.2
#       256        0.995         11.3         19.7
# Pick efSearch=64: 0.943 recall, 5.8ms p99 — sweet spot
```

---

## Continuous Evaluation in Production

```python
# Automated eval job — run weekly via cron or GitHub Actions
async def run_weekly_evaluation():
    evaluator = Evaluator(client, "production_docs", embedder)
    results = evaluator.evaluate(eval_set, k=10)

    # Push to Prometheus gauge
    RECALL_AT_K.labels(collection="production_docs", k="10").set(results.recall_at_k)

    # Alert if quality dropped
    if results.recall_at_k < 0.85:
        await send_alert(
            level="warning",
            title="Search quality degraded",
            body=f"Recall@10 dropped to {results.recall_at_k:.3f} (threshold: 0.85)"
        )

    # Log to audit trail
    logger.info("weekly_evaluation_complete", extra={
        "recall_at_10": results.recall_at_k,
        "ndcg_at_10": results.ndcg_at_k,
        "mrr": results.mrr,
        "avg_latency_ms": results.avg_latency_ms,
        "p99_latency_ms": results.p99_latency_ms,
    })

    return results
```

---

## Summary

| Metric | Formula | Target |
|--------|---------|--------|
| Recall@K | \|Relevant ∩ Retrieved\| / \|Relevant\| | > 0.85 |
| Precision@K | \|Relevant ∩ Retrieved\| / K | Depends on use case |
| MRR | Mean of 1/rank_of_first_hit | > 0.7 |
| NDCG@K | Graded relevance, position-weighted | > 0.8 |
| p99 Latency | 99th percentile response time | < 50ms (standard), < 10ms (low-latency) |

| Concept | Key Point |
|---------|-----------|
| Gold labels | Human-labeled ground truth; most reliable |
| LLM-generated | Fast eval set generation; biased toward LLM knowledge |
| Search logs | Implicit labels from click data; great for production monitoring |
| efSearch benchmark | Always benchmark recall vs latency before choosing efSearch value |
| Continuous eval | Run weekly; alert on recall drop > 5% |

---

## Resources

- [BEIR Benchmark](https://github.com/beir-cellar/beir) — standard retrieval evaluation benchmark
- [Ragas](https://docs.ragas.io/) — RAG evaluation framework
- [Locust](https://locust.io/) — load testing
- [MTEB](https://huggingface.co/spaces/mteb/leaderboard) — embedding model evaluation leaderboard
