# 2. Distributed Tracing & Logging — Following a Request Through the System

---

## ELI5

**Tracing** = following a single request as it bounces between services. Like a GPS tracker on a package showing every stop it made.

**Logging** = text records of what happened. Like a diary entry for each operation.

Together they answer: "Why did this specific request take 800ms?" — metrics tell you THAT it was slow, traces tell you WHERE it was slow, logs tell you WHY.

---

## OpenTelemetry — The Standard

OpenTelemetry (OTel) is the vendor-neutral standard for collecting traces, metrics, and logs.

```
Your App (OTel instrumented)
    │
    ▼
OTel Collector (central aggregator)
    ├── Traces → Jaeger / Tempo
    ├── Metrics → Prometheus
    └── Logs → Loki / CloudWatch
```

---

## Tracing with OpenTelemetry + FastAPI

```python
# src/telemetry.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.sdk.resources import Resource
import os

def setup_telemetry(app, service_name: str = "vector-search-api"):
    resource = Resource.create({
        "service.name": service_name,
        "service.version": os.environ.get("APP_VERSION", "unknown"),
        "deployment.environment": os.environ.get("ENV", "production"),
    })

    # Configure exporter (to OTel Collector or directly to Jaeger)
    exporter = OTLPSpanExporter(
        endpoint=os.environ.get("OTEL_EXPORTER_OTLP_ENDPOINT", "http://otel-collector:4317"),
    )

    provider = TracerProvider(resource=resource)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    # Auto-instrument: FastAPI, Redis, HTTPX — zero code changes needed
    FastAPIInstrumentor.instrument_app(app)
    RedisInstrumentor().instrument()
    HTTPXClientInstrumentor().instrument()

# In main.py
from src.telemetry import setup_telemetry

app = FastAPI()
setup_telemetry(app)
```

### Manual Spans for Custom Operations

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

async def search_with_tracing(tenant_id: str, query: str, top_k: int):
    # This span wraps the entire search operation
    with tracer.start_as_current_span("vector_search") as span:
        span.set_attribute("tenant.id", tenant_id)
        span.set_attribute("search.query_length", len(query))
        span.set_attribute("search.top_k", top_k)

        # Sub-span: embedding
        with tracer.start_as_current_span("embed_query") as embed_span:
            query_vec = embed(query)
            embed_span.set_attribute("vector.dimensions", len(query_vec))

        # Sub-span: Qdrant search
        with tracer.start_as_current_span("qdrant_search") as qdrant_span:
            results = client.query_points(
                collection_name="documents",
                query=query_vec,
                shard_key_selector=tenant_id,
                limit=top_k,
            )
            qdrant_span.set_attribute("search.result_count", len(results.points))

        # Sub-span: re-ranking
        if len(results.points) > 0:
            with tracer.start_as_current_span("rerank"):
                results = rerank(query, results.points)

        span.set_attribute("search.final_result_count", len(results))
        return results
```

**What a trace looks like:**
```
search_request (total: 42ms)
├── embed_query (8ms)
│   └── openai.embeddings.create (7ms)
├── redis_get (1ms)        ← cache miss
├── qdrant_search (28ms)
│   └── grpc_call (27ms)
├── rerank (4ms)
└── redis_set (1ms)        ← cache write
```

---

## Structured Logging

Use JSON logs — they're machine-readable and can be queried in CloudWatch/Loki.

```python
# src/logging_config.py
import logging
import json
import time
from typing import Any

class JSONFormatter(logging.Formatter):
    """Emit logs as JSON for easy querying."""

    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%S", time.gmtime(record.created)),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "service": "vector-search-api",
        }

        # Add trace context if available (correlate logs with traces)
        from opentelemetry import trace as otel_trace
        span = otel_trace.get_current_span()
        if span.is_recording():
            ctx = span.get_span_context()
            log_entry["trace_id"] = f"{ctx.trace_id:032x}"
            log_entry["span_id"] = f"{ctx.span_id:016x}"

        # Add extra fields
        for key, value in record.__dict__.items():
            if key not in ("message", "args", "msg", "levelname", "name",
                          "pathname", "filename", "funcName", "lineno",
                          "created", "thread", "process", "exc_info",
                          "exc_text", "stack_info", "msecs", "relativeCreated"):
                log_entry[key] = value

        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)

        return json.dumps(log_entry)

def setup_logging():
    handler = logging.StreamHandler()
    handler.setFormatter(JSONFormatter())

    root = logging.getLogger()
    root.handlers.clear()
    root.addHandler(handler)
    root.setLevel(logging.INFO)
```

### What to Log

```python
import logging

logger = logging.getLogger(__name__)

# Log with structured extra fields
async def handle_search(tenant_id: str, query: str, top_k: int):
    logger.info("search_started", extra={
        "tenant_id": tenant_id,
        "query_length": len(query),
        "top_k": top_k,
    })

    start = time.time()
    try:
        results = await search(tenant_id, query, top_k)
        duration = time.time() - start

        logger.info("search_completed", extra={
            "tenant_id": tenant_id,
            "duration_ms": round(duration * 1000, 2),
            "result_count": len(results),
        })

        return results

    except Exception as e:
        duration = time.time() - start
        logger.error("search_failed", extra={
            "tenant_id": tenant_id,
            "duration_ms": round(duration * 1000, 2),
            "error": str(e),
            "error_type": type(e).__name__,
        }, exc_info=True)
        raise

# Slow query logging
SLOW_QUERY_THRESHOLD_MS = 100

async def search(tenant_id: str, query: str, top_k: int):
    start = time.time()
    results = client.query_points(...)
    duration_ms = (time.time() - start) * 1000

    if duration_ms > SLOW_QUERY_THRESHOLD_MS:
        logger.warning("slow_query", extra={
            "tenant_id": tenant_id,
            "duration_ms": duration_ms,
            "top_k": top_k,
        })

    return results
```

### What NOT to Log

```python
# NEVER log:
logger.info(f"User query: {user_query}")    # PII risk if query contains names/emails
logger.debug(f"Vector: {query_vector}")     # vectors are huge (768+ floats)
logger.info(f"API key: {api_key}")          # secrets
logger.info(f"JWT: {token}")                # secrets

# DO log:
logger.info("query_received", extra={
    "query_length": len(user_query),        # length is fine, not content
    "top_k": top_k,
    "has_filters": bool(filters),
})
```

---

## Log Aggregation

### Grafana Loki (K8s native)

```yaml
# Loki stack for K8s log aggregation
# Promtail (DaemonSet) collects logs from all pods and sends to Loki
# Grafana queries Loki for log search

# docker-compose for local setup
services:
  loki:
    image: grafana/loki:3.0.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/loki.yml
    volumes:
      - ./config/loki.yml:/etc/loki/loki.yml:ro

  promtail:
    image: grafana/promtail:3.0.0
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./config/promtail.yml:/etc/promtail/promtail.yml:ro
    command: -config.file=/etc/promtail/promtail.yml
```

### Querying Logs in Grafana (LogQL)

```
# All errors from api service
{app="api"} |= "ERROR"

# Search failures with context
{app="api"} | json | event="search_failed" | duration_ms > 100

# Slow queries for a specific tenant
{app="api"} | json | event="slow_query" | tenant_id="acme" | line_format "{{.duration_ms}}ms"

# Error rate (Loki metric query)
sum(rate({app="api"} | json | level="ERROR" [5m])) by (tenant_id)
```

---

## Health Endpoints

```python
from fastapi import FastAPI
from enum import Enum

class HealthStatus(str, Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"

@app.get("/health/live")
async def liveness():
    """K8s liveness probe: am I running? (if unhealthy, restart me)"""
    return {"status": HealthStatus.HEALTHY}

@app.get("/health/ready")
async def readiness():
    """K8s readiness probe: can I serve traffic? (if not, stop sending me requests)"""
    checks = {}

    # Check Qdrant
    try:
        info = client.get_collections()
        checks["qdrant"] = HealthStatus.HEALTHY
    except Exception as e:
        checks["qdrant"] = HealthStatus.UNHEALTHY
        logger.error("qdrant_health_check_failed", extra={"error": str(e)})

    # Check Redis
    try:
        await redis.ping()
        checks["redis"] = HealthStatus.HEALTHY
    except Exception as e:
        checks["redis"] = HealthStatus.DEGRADED  # degraded: cache miss, not fatal
        logger.warning("redis_health_check_failed", extra={"error": str(e)})

    # Overall status: unhealthy if any critical service is down
    if checks.get("qdrant") == HealthStatus.UNHEALTHY:
        from fastapi import HTTPException
        raise HTTPException(status_code=503, detail=checks)

    overall = (
        HealthStatus.HEALTHY
        if all(v == HealthStatus.HEALTHY for v in checks.values())
        else HealthStatus.DEGRADED
    )

    return {"status": overall, "checks": checks}

@app.get("/health/startup")
async def startup():
    """K8s startup probe: am I done starting up?"""
    # Used to distinguish "still booting" from "crashed"
    # Same logic as readiness but longer timeout window
    return await readiness()
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| OpenTelemetry | Vendor-neutral standard; auto-instruments FastAPI, Redis, HTTPX |
| Distributed traces | Follow a request across services; identify bottlenecks |
| Manual spans | Wrap custom operations (embedding, Qdrant search, reranking) |
| Structured logs | JSON format; machine-readable; query with LogQL |
| Trace correlation | Include trace_id in logs to correlate metrics→traces→logs |
| What not to log | Never log PII, vectors, API keys, JWTs |
| Health endpoints | /live (restart?), /ready (send traffic?), /startup (still booting?) |

---

## Resources

- [OpenTelemetry Python](https://opentelemetry-python.readthedocs.io/)
- [Grafana Loki](https://grafana.com/docs/loki/latest/)
- [Jaeger](https://www.jaegertracing.io/) — open-source distributed tracing
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) — central aggregator
