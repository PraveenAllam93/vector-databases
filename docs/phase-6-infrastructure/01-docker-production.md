# 1. Docker Production — From Dev to Production-Ready Containers

---

## ELI5

Docker packages your application and all its dependencies into a box (container) that runs the same way everywhere — your laptop, staging server, or production cloud. No "it works on my machine."

In production, "works on my machine" is not good enough. You need: minimal attack surface, non-root user, deterministic builds, efficient layer caching.

---

## The Docker Build Anti-Patterns

```dockerfile
# BAD — common mistakes in prod Dockerfiles:

FROM python:latest       # ← unpinned; breaks silently when new version releases
WORKDIR /app
COPY . .                  # ← copies .env, .git, __pycache__, etc.
RUN pip install -r requirements.txt  # ← no cache optimization
CMD python main.py        # ← runs as root
                          # ← no health check
```

---

## Production Dockerfile (FastAPI)

```dockerfile
# ─── Stage 1: Dependencies ───────────────────────────────────────────────────
FROM python:3.12-slim AS builder

# Install build tools (needed for some packages, discarded after build)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy only requirements first — enables layer caching
# If requirements.txt doesn't change, this layer is reused
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# ─── Stage 2: Runtime ────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

# Security: create non-root user
RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid appuser --shell /bin/bash --create-home appuser

WORKDIR /app

# Copy installed packages from builder (no build tools in runtime image)
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser src/ ./src/

# Switch to non-root user
USER appuser

# PATH for user-installed packages
ENV PATH=/home/appuser/.local/bin:$PATH

# Tell Docker what port this container listens on (documentation only)
EXPOSE 8000

# Health check — Docker will restart unhealthy containers
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD python -c "import httpx; httpx.get('http://localhost:8000/health/live').raise_for_status()"

# Use exec form (not shell form) — proper signal handling
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Why multi-stage?**
```
builder image: ~900 MB (Python + gcc + all build tools)
runtime image: ~180 MB (Python slim + only installed packages)

Smaller image = faster pulls, smaller attack surface, less storage cost
```

---

## .dockerignore

```
# .dockerignore — like .gitignore but for Docker build context
.git
.github
**/__pycache__
**/*.pyc
**/*.pyo
.env
.env.*
*.log
*.tmp
docs/
tests/
.pytest_cache/
.coverage
htmlcov/
node_modules/
*.md
Dockerfile*
docker-compose*
```

---

## requirements.txt — Pinned Dependencies

```txt
# Pin EXACT versions for reproducible builds
fastapi==0.115.0
uvicorn[standard]==0.30.6
qdrant-client==1.11.1
openai==1.47.0
sentence-transformers==3.1.1
redis==5.1.1
pydantic==2.9.2
httpx==0.27.2
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.12
prometheus-client==0.21.0
opentelemetry-api==1.27.0
opentelemetry-sdk==1.27.0
opentelemetry-instrumentation-fastapi==0.48b0
```

Generate pinned requirements from your dev environment:
```bash
pip freeze > requirements.txt
```

Or use pip-compile for deterministic resolution:
```bash
pip install pip-tools
pip-compile requirements.in   # produces requirements.txt with pinned transitive deps
```

---

## Docker Compose — Full Production Stack

```yaml
# docker-compose.yml — full stack for local development / staging
version: "3.9"

services:
  # ─── Vector Database ───────────────────────────────────────────────────────
  qdrant:
    image: qdrant/qdrant:v1.11.3   # pinned version
    ports:
      - "6333:6333"    # REST API + Web UI (remove in production!)
      - "6334:6334"    # gRPC
    volumes:
      - qdrant_storage:/qdrant/storage
      - ./config/qdrant.yaml:/qdrant/config/production.yaml:ro
    environment:
      - QDRANT__SERVICE__API_KEY=${QDRANT_API_KEY}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    networks:
      - backend

  # ─── Cache ─────────────────────────────────────────────────────────────────
  redis:
    image: redis:7.4-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 1gb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"    # remove in production!
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 15s
      timeout: 5s
      retries: 3
    restart: unless-stopped
    networks:
      - backend

  # ─── Message Queue ──────────────────────────────────────────────────────────
  zookeeper:
    image: confluentinc/cp-zookeeper:7.7.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data
    networks:
      - backend

  kafka:
    image: confluentinc/cp-kafka:7.7.0
    depends_on: [zookeeper]
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - kafka_data:/var/lib/kafka/data
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
    networks:
      - backend

  # ─── Search API ─────────────────────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    ports:
      - "8000:8000"
    environment:
      - QDRANT_URL=http://qdrant:6333
      - QDRANT_API_KEY=${QDRANT_API_KEY}
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - JWT_PRIVATE_KEY_PATH=/secrets/jwt-private.pem
    volumes:
      - ./secrets:/secrets:ro
    depends_on:
      qdrant:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    networks:
      - backend
      - frontend

  # ─── Ingestion Workers ──────────────────────────────────────────────────────
  ingestion-worker:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    command: ["python", "-m", "src.workers.ingestion"]
    environment:
      - QDRANT_URL=http://qdrant:6333
      - QDRANT_API_KEY=${QDRANT_API_KEY}
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      qdrant:
        condition: service_healthy
      kafka:
        condition: service_healthy
    deploy:
      replicas: 3    # scale workers independently from API
    restart: unless-stopped
    networks:
      - backend

  # ─── Observability ──────────────────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:v2.54.1
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=30d
    networks:
      - backend

  grafana:
    image: grafana/grafana:11.2.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./config/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
    depends_on: [prometheus]
    networks:
      - backend

volumes:
  qdrant_storage:
  redis_data:
  kafka_data:
  zookeeper_data:
  prometheus_data:
  grafana_data:

networks:
  backend:
    driver: bridge
    internal: true        # no direct internet access from backend services
  frontend:
    driver: bridge        # api is on both; clients can reach it
```

---

## Docker Build & Run Commands

```bash
# Build production image
docker build --target runtime -t vector-search-api:1.0.0 .

# Build with build args (e.g., for different environments)
docker build \
  --build-arg PYTHON_VERSION=3.12 \
  --target runtime \
  -t vector-search-api:1.0.0 \
  .

# Run full stack
docker compose up -d

# Scale workers
docker compose up -d --scale ingestion-worker=5

# Check health
docker compose ps
docker compose logs api --tail 50

# Production: build, tag, push to registry
docker build --target runtime -t vector-search-api:${GIT_SHA} .
docker tag vector-search-api:${GIT_SHA} 123456789.dkr.ecr.us-east-1.amazonaws.com/vector-search:${GIT_SHA}
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/vector-search:${GIT_SHA}
```

---

## Container Security Scanning

```bash
# Trivy: scan image for vulnerabilities
trivy image vector-search-api:1.0.0

# Output example:
# 2024-01-15T10:00:00Z INFO Detected OS: debian 12.0
# HIGH: openssl - CVE-2023-XXXX (upgrade to 3.0.11+)

# In CI/CD: fail build if HIGH or CRITICAL vulnerabilities found
trivy image --exit-code 1 --severity HIGH,CRITICAL vector-search-api:1.0.0
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Multi-stage build | Builder (large) → Runtime (small, ~180MB); discards build tools |
| Non-root user | Security: containers run as UID 1000, not root |
| .dockerignore | Excludes .env, .git, __pycache__ from build context |
| Layer caching | Copy requirements.txt before src/ to maximize cache hits |
| HEALTHCHECK | Docker restarts unhealthy containers automatically |
| Pinned versions | Exact versions in requirements.txt and image tags |
| Separate services | API, workers, Qdrant all independent; scale workers separately |
| Security scanning | trivy in CI; fail build on HIGH/CRITICAL CVEs |
