# Phase 7: Observability & Evaluation

## Reading Order

| # | Topic | File | Key Takeaway |
|---|-------|------|-------------|
| 1 | [Metrics & Monitoring](01-metrics-and-monitoring.md) | Prometheus counters/histograms/gauges for latency, errors, QPS, recall; Grafana dashboards; alert on p99 > 100ms and error rate > 1% |
| 2 | [Tracing & Logging](02-tracing-and-logging.md) | OpenTelemetry auto-instruments FastAPI/Redis; manual spans for embed/search/rerank; JSON structured logs with trace_id correlation; /health/live + /health/ready |
| 3 | [Evaluation & Benchmarking](03-evaluation-and-benchmarking.md) | Recall@K, Precision@K, MRR, NDCG formulas + implementations; efSearch recall-vs-latency benchmark; continuous eval with weekly alerts |

---

## Interview Quick Reference

**"How do you monitor your vector search system?"**
→ Three pillars: Metrics (Prometheus), Traces (OpenTelemetry → Jaeger/Tempo), Logs (structured JSON → Loki). Key metrics: search_latency_p99 (alert > 100ms), error_rate (alert > 1%), recall@10 (alert < 0.85), Kafka consumer lag (alert > 10k), Qdrant disk usage (alert > 80%). Grafana dashboards with per-tenant breakdown.

**"How do you measure search quality?"**
→ Offline: Recall@K (coverage), Precision@K (relevance density), MRR (was best doc near top?), NDCG@K (graded relevance). Evaluation set: gold labels from human annotation or LLM-generated synthetic queries. Run weekly evaluation job → push results to Prometheus → alert on > 5% regression. Online: implicit signals from user behavior (clicks, feedback buttons).

**"How do you know if a model update improved search quality?"**
→ Shadow evaluation: deploy new model in parallel → run evaluation set against both → compare Recall@K and NDCG. Use A/B testing: route 10% of traffic to new model → measure user satisfaction. Only switch if quality improves AND latency budget holds. Roll back automatically if recall drops > 5%.

**"How do you debug a slow search request?"**
→ Find the trace in Jaeger (via trace_id in logs). Each span shows: embed_query (OpenAI API call?), redis_get (cache miss?), qdrant_search (too many candidates?), rerank (cross-encoder bottleneck?). Check span attributes: was top_k unusually large? Was the shard_key_selector missing (triggering scatter-gather)? Compare to baseline: is this one tenant, or all tenants?

**"How do you tune efSearch for production?"**
→ Build an evaluation set with ground truth labels. Run benchmark: for each efSearch value (16, 32, 64, 128, 256), measure Recall@10 and p99 latency. Pick the sweet spot: typically efSearch=64 gives 0.94 recall at 5-8ms p99. Higher recall has diminishing returns; lower efSearch degrades recall significantly. Re-benchmark after quantization changes.

**"What's in your Grafana dashboard?"**
→ Row 1: Overview — QPS, p99 latency, error rate, cache hit rate. Row 2: Qdrant internals — vectors per collection, segment count, memory per node, disk per node. Row 3: Ingestion — ingest QPS, Kafka consumer lag, embedding latency. Row 4: Infrastructure — CPU/memory per pod, network I/O, pod restarts. Separate dashboards per tenant for isolation/debugging.

---

## Observability Checklist

```
Metrics:
  □ /metrics endpoint exposed on all services
  □ Prometheus scraping all pods via annotations
  □ Key metrics defined: search_latency, error_rate, recall_at_k
  □ Histograms for latency (not summaries — histograms support aggregation)
  □ Labels: tenant_id, status (success/error)

Alerts:
  □ p99 latency > 100ms → warning, > 500ms → critical
  □ Error rate > 1% → warning
  □ Recall@10 < 0.85 → warning
  □ Kafka consumer lag > 10,000 → warning
  □ Disk usage > 80% → warning
  □ Pod restart > 3 times/hr → warning

Tracing:
  □ OpenTelemetry configured with OTLP exporter
  □ Auto-instrumentation: FastAPI, Redis, HTTPX
  □ Manual spans: embed_query, qdrant_search, rerank
  □ trace_id injected into all log records
  □ Sampling rate configured (100% dev, 10% prod)

Logging:
  □ JSON structured logging (not plaintext)
  □ Log levels: DEBUG (dev), INFO (staging/prod)
  □ Slow query logging (> 100ms)
  □ Never log PII, vectors, secrets, JWTs
  □ Log aggregation: Loki or CloudWatch

Evaluation:
  □ Evaluation dataset created (>= 100 queries)
  □ Baseline Recall@10, NDCG@10, MRR recorded
  □ Weekly evaluation job scheduled
  □ Recall@K gauge in Prometheus
  □ efSearch benchmarked for recall vs latency
  □ Load test run before every major release

Health Endpoints:
  □ /health/live (liveness probe)
  □ /health/ready (readiness probe, checks Qdrant + Redis)
  □ /health/startup (startup probe — avoids false liveness failures during boot)
```
