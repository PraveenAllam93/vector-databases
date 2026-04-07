# 1. Metrics & Monitoring — Knowing Your System is Healthy

---

## ELI5

Observability = your ability to understand what's happening inside your system from the outside. Like a doctor checking vital signs — heart rate, blood pressure, temperature — you check latency, error rate, throughput.

Without observability, production incidents are hunts in the dark. With it, you see the problem before users report it.

---

## The Three Pillars of Observability

```
Metrics   → Numbers over time (latency p99, error rate, QPS)
Traces    → A single request's journey through all services
Logs      → Detailed text records of what happened
```

---

## Key Metrics for Vector Search

```
Application metrics:
  search_latency_p50/p95/p99  → How fast are searches? (target: < 50ms p99)
  search_error_rate            → % of searches failing (alert if > 0.1%)
  search_qps                   → Queries per second (capacity planning)
  recall_at_k                  → Search quality (alert if drops > 5%)

  ingest_latency               → How long does ingestion take?
  ingest_queue_depth           → Kafka consumer lag (alert if > 10,000)
  embedding_latency            → Time to embed a document

Qdrant internals:
  qdrant_collections_total     → Number of collections
  qdrant_vectors_total         → Total vectors per collection
  qdrant_segments_count        → Segment count (many = compaction pending)
  qdrant_pending_optimizations → HNSW rebuild queue

Infrastructure:
  memory_usage_percent         → High = evictions, OOM risk
  cpu_usage_percent            → High = latency spikes
  disk_usage_percent           → Alert at 80% (Qdrant needs space for compaction)
  network_bytes_in/out         → Replication bandwidth
```

---

## Prometheus + FastAPI

```python
# src/metrics.py
from prometheus_client import (
    Counter, Histogram, Gauge, Summary,
    generate_latest, CONTENT_TYPE_LATEST
)
from fastapi import Response
import time

# ─── Define Metrics ───────────────────────────────────────────────────────────

SEARCH_LATENCY = Histogram(
    "search_latency_seconds",
    "End-to-end search latency",
    buckets=[0.005, 0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 1.0],
    labelnames=["tenant_id", "status"],
)

SEARCH_REQUESTS = Counter(
    "search_requests_total",
    "Total number of search requests",
    labelnames=["tenant_id", "status"],
)

INGEST_REQUESTS = Counter(
    "ingest_requests_total",
    "Total ingestion requests",
    labelnames=["tenant_id", "status"],
)

INGEST_LATENCY = Histogram(
    "ingest_latency_seconds",
    "End-to-end ingestion latency",
    buckets=[0.1, 0.5, 1.0, 5.0, 10.0, 30.0, 60.0],
    labelnames=["tenant_id"],
)

EMBEDDING_LATENCY = Histogram(
    "embedding_latency_seconds",
    "Embedding generation latency",
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.0],
    labelnames=["model"],
)

RECALL_AT_K = Gauge(
    "recall_at_k",
    "Search recall at K (0-1)",
    labelnames=["collection", "k"],
)

QDRANT_VECTORS = Gauge(
    "qdrant_vectors_total",
    "Total vectors per collection",
    labelnames=["collection"],
)

CACHE_HITS = Counter(
    "cache_hits_total",
    "Cache hit/miss counts",
    labelnames=["result"],    # "hit" or "miss"
)

# ─── Expose /metrics Endpoint ────────────────────────────────────────────────

from fastapi import FastAPI
app = FastAPI()

@app.get("/metrics")
async def metrics():
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST,
    )

# ─── Instrument Endpoints ─────────────────────────────────────────────────────

from functools import wraps

def track_search(func):
    """Decorator to track search latency and errors."""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        tenant_id = kwargs.get("tenant_id", "unknown")
        start = time.time()
        status = "success"
        try:
            result = await func(*args, **kwargs)
            return result
        except Exception as e:
            status = "error"
            raise
        finally:
            duration = time.time() - start
            SEARCH_LATENCY.labels(tenant_id=tenant_id, status=status).observe(duration)
            SEARCH_REQUESTS.labels(tenant_id=tenant_id, status=status).inc()
    return wrapper

@app.post("/search")
@track_search
async def search(request: SearchRequest, tenant_id: str = Depends(get_tenant_id)):
    # check cache
    cached = await redis.get(cache_key(tenant_id, request.text))
    if cached:
        CACHE_HITS.labels(result="hit").inc()
        return json.loads(cached)
    CACHE_HITS.labels(result="miss").inc()

    # search
    with SEARCH_LATENCY.labels(tenant_id=tenant_id, status="success").time():
        results = await vector_search(tenant_id, request.text)

    return results
```

---

## Prometheus Configuration

```yaml
# config/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: "api"
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ["vector-search"]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        target_label: __metrics_path__
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __address__]
        target_label: __address__
        regex: (.+)(?::d+);(d+)

  - job_name: "qdrant"
    static_configs:
      - targets: ["qdrant.vector-search.svc.cluster.local:6333"]
    metrics_path: /metrics
```

---

## Alert Rules

```yaml
# config/alerts.yml
groups:
  - name: vector-search
    interval: 60s
    rules:

      # High search latency
      - alert: SearchLatencyHigh
        expr: histogram_quantile(0.99, rate(search_latency_seconds_bucket[5m])) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p99 search latency > 100ms"
          description: "p99 latency is {{ $value | humanizeDuration }}"

      - alert: SearchLatencyCritical
        expr: histogram_quantile(0.99, rate(search_latency_seconds_bucket[5m])) > 0.5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "p99 search latency > 500ms"

      # High error rate
      - alert: SearchErrorRateHigh
        expr: |
          rate(search_requests_total{status="error"}[5m])
          /
          rate(search_requests_total[5m]) > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Search error rate > 1%"
          description: "Error rate: {{ $value | humanizePercentage }}"

      # Low recall (search quality degraded)
      - alert: RecallDegraded
        expr: recall_at_k{k="10"} < 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Recall@10 dropped below 0.85"

      # Kafka consumer lag (ingestion falling behind)
      - alert: IngestQueueHighLag
        expr: kafka_consumer_lag_sum{topic="vector-ingestion"} > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Ingestion Kafka lag > 10,000 messages"

      # Qdrant disk usage
      - alert: QdrantDiskHigh
        expr: |
          (kubelet_volume_stats_used_bytes{persistentvolumeclaim=~"qdrant-storage-.*"}
          / kubelet_volume_stats_capacity_bytes{persistentvolumeclaim=~"qdrant-storage-.*"}) > 0.80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Qdrant disk usage > 80%"

      # Pod restarts
      - alert: PodRestarting
        expr: increase(kube_pod_container_status_restarts_total{namespace="vector-search"}[1h]) > 3
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} restarted {{ $value }} times in 1hr"
```

---

## Grafana Dashboards

Key panels to include:

```
Row 1: Overview
  ├── Search QPS (rate(search_requests_total[1m]))
  ├── Search p99 Latency (histogram_quantile(0.99, ...))
  ├── Error Rate (%)
  └── Cache Hit Rate (%)

Row 2: Qdrant
  ├── Vectors per Collection
  ├── Segments Count
  ├── Memory Usage per Node
  └── Disk Usage per Node

Row 3: Ingestion
  ├── Ingest QPS
  ├── Kafka Consumer Lag
  ├── Embedding Latency p95
  └── Ingestion Error Rate

Row 4: Infrastructure
  ├── CPU per pod
  ├── Memory per pod
  ├── Network I/O
  └── Pod restart count
```

```python
# Grafana dashboard JSON (excerpt) — export from UI after building manually
{
  "panels": [
    {
      "title": "Search p99 Latency",
      "type": "graph",
      "targets": [{
        "expr": "histogram_quantile(0.99, sum(rate(search_latency_seconds_bucket[5m])) by (le, tenant_id))",
        "legendFormat": "{{tenant_id}}",
      }],
      "thresholds": [
        {"value": 0.05, "colorMode": "ok"},
        {"value": 0.1, "colorMode": "warning"},
        {"value": 0.5, "colorMode": "critical"},
      ]
    }
  ]
}
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Three pillars | Metrics (numbers), traces (request journeys), logs (text) |
| Key metrics | p99 latency, error rate, QPS, recall@K, disk usage |
| Prometheus | Pull-based; scrape /metrics from each pod |
| Histograms | Use for latency (compute any percentile from one metric) |
| Alerting | Latency > 100ms warning, > 500ms critical; error rate > 1% alert |
| Recall@K | Track quality metric — silent degradation without it |
| Grafana | Visualize trends; dashboards for app, Qdrant, infra |

---

## Resources

- [Prometheus Python Client](https://github.com/prometheus/client_python)
- [Qdrant Metrics](https://qdrant.tech/documentation/guides/monitoring/)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/) — community dashboards
- [Google SRE Book: Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
