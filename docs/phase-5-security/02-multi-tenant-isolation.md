# 2. Multi-Tenant Isolation — No Data Leaks, Ever

---

## ELI5

Imagine a shared office building. Every company has their own locked room. They share the elevator (compute) and plumbing (infrastructure), but nobody can walk into another company's room. That's multi-tenancy.

The biggest fear: Tenant A accidentally sees Tenant B's data. This is a data leak — a catastrophic security incident that ends products and careers.

---

## Threat Model: What Can Go Wrong

```
Attack Surface:
  1. Developer bug: forgot tenant_id filter on a query
  2. API exploit: attacker sends someone else's tenant_id in request body
  3. SSRF: attacker tricks your server into querying another tenant's namespace
  4. Shard leakage: scatter-gather returns results from wrong shard
  5. Cache collision: Redis key includes query but not tenant_id
  6. Log leakage: debug logs print vectors from another tenant's documents
  7. Timing attack: query timing reveals if certain data exists in another tenant
```

---

## Isolation Models

### Model 1: Separate Clusters (Strongest, Most Expensive)

```
Tenant A → Dedicated Qdrant cluster (separate VMs/k8s namespace)
Tenant B → Dedicated Qdrant cluster
Tenant C → Dedicated Qdrant cluster

Pros:
  - Zero blast radius — Tenant A crash can't affect Tenant B
  - Dedicated resources — no noisy neighbors
  - Easiest to audit

Cons:
  - 1000 tenants = 1000 clusters (not viable unless enterprise-only)
  - High cost, complex operations
```

### Model 2: Separate Collections (Balanced)

```
Qdrant cluster:
  Collection "tenant_acme_documents"
  Collection "tenant_globex_documents"
  Collection "tenant_initech_documents"

Pros:
  - Strong isolation (different collection = different HNSW index)
  - Per-tenant configuration (different dimensions, different quantization)
  - Simple query: no filter needed, just query the right collection

Cons:
  - Collection overhead at scale (1000 tenants = 1000 collections)
  - Qdrant recommends < 500 collections per node for stability
  - No sharing of index/compute even for identical embedding models
```

```python
class CollectionPerTenantService:
    def __init__(self, client: QdrantClient):
        self.client = client

    def collection_name(self, tenant_id: str) -> str:
        # Sanitize to prevent injection
        safe_id = tenant_id.replace("/", "_").replace(".", "_")
        return f"tenant_{safe_id}_documents"

    def ensure_collection(self, tenant_id: str, vector_size: int):
        name = self.collection_name(tenant_id)
        existing = {c.name for c in self.client.get_collections().collections}
        if name not in existing:
            self.client.create_collection(
                collection_name=name,
                vectors_config=VectorParams(size=vector_size, distance=Distance.COSINE),
            )

    def search(self, tenant_id: str, query_vec: list[float], top_k: int = 10):
        # Impossible to leak: wrong collection = wrong tenant
        return self.client.query_points(
            collection_name=self.collection_name(tenant_id),
            query=query_vec,
            limit=top_k,
        )
```

### Model 3: Shared Collection with Custom Sharding (Recommended at Scale)

```
Single collection "documents"
Sharding: CUSTOM by tenant_id

Tenant A → shard key "acme"   → physical nodes [1, 4, 7]
Tenant B → shard key "globex" → physical nodes [2, 5, 8]

Pros:
  - Scales to thousands of tenants
  - Physical shard isolation (cross-tenant queries impossible at shard level)
  - Cost-efficient (small tenants share nodes)

Cons:
  - Must ALWAYS include shard_key_selector in every query
  - Bug (forgetting the shard key) = scatter-gather = sees all data
```

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, Distance, ShardingMethod, CreateShardKey
)

client = QdrantClient(host="localhost", port=6333)

# Create collection with custom sharding
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    sharding_method=ShardingMethod.CUSTOM,
)

# Create a shard key for each tenant (call at tenant onboarding)
def onboard_tenant(tenant_id: str):
    client.create_shard_key(
        collection_name="documents",
        shard_key=tenant_id,
        shards_number=1,        # small tenants: 1 shard
        replication_factor=2,
    )

# Large tenant: more shards
def onboard_enterprise_tenant(tenant_id: str):
    client.create_shard_key(
        collection_name="documents",
        shard_key=tenant_id,
        shards_number=4,        # dedicated 4 shards
        replication_factor=2,
    )
```

---

## Defense in Depth Pattern

Never rely on a single isolation layer. Use all of them together.

```
Layer 1: JWT — tenant_id extracted from token (cannot be spoofed)
Layer 2: Shard key — query routed to tenant's shard only
Layer 3: Payload filter — belt-AND-suspenders filter on tenant_id field
Layer 4: Middleware — inject tenant context, impossible to forget
Layer 5: Audit log — record every cross-tenant attempt
Layer 6: Tests — automated cross-tenant access tests (must fail)
```

```python
from fastapi import FastAPI, Request, Depends
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

# Layer 4: Middleware auto-injects tenant context
class TenantMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        auth_header = request.headers.get("Authorization", "")
        if auth_header.startswith("Bearer "):
            token = auth_header[7:]
            try:
                claims = decode_jwt(token)
                # Store on request state — available in every endpoint
                request.state.tenant_id = claims["tenant_id"]
                request.state.user_id = claims["sub"]
                request.state.roles = claims.get("roles", [])
            except Exception:
                pass  # auth enforcement happens in individual endpoints
        response = await call_next(request)
        return response

app.add_middleware(TenantMiddleware)

# Layer 1+2+3: Search endpoint
@app.post("/search")
async def search(query: SearchQuery, request: Request):
    # Layer 1: from JWT via middleware — can't be forged
    tenant_id = request.state.tenant_id

    query_vec = embed(query.text)

    results = client.query_points(
        collection_name="documents",
        query=query_vec,
        shard_key_selector=tenant_id,           # Layer 2: shard routing
        query_filter=Filter(
            must=[
                FieldCondition(
                    key="tenant_id",
                    match=MatchValue(value=tenant_id),  # Layer 3: belt+suspenders
                )
            ]
        ),
        limit=query.top_k,
    )

    # Layer 5: audit log
    audit_logger.info({
        "event": "search",
        "tenant_id": tenant_id,
        "user_id": request.state.user_id,
        "result_count": len(results.points),
    })

    return results
```

---

## Automated Cross-Tenant Access Tests

Run these in CI/CD. They MUST fail (return 0 results or 403).

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_tenant_a_cannot_see_tenant_b_data():
    """Tenant A searching must never return Tenant B's documents."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        # Tenant B's token
        token_b = create_jwt("user_b", "tenant_b", ["search"])

        # Tenant A's token — this is who's actually searching
        token_a = create_jwt("user_a", "tenant_a", ["search"])

        # Ingest into Tenant B
        await client.post(
            "/ingest",
            json={"text": "Tenant B secret document XYZ-SECRET-42"},
            headers={"Authorization": f"Bearer {token_b}"},
        )

        # Search as Tenant A — MUST NOT find Tenant B's data
        resp = await client.post(
            "/search",
            json={"text": "XYZ-SECRET-42"},
            headers={"Authorization": f"Bearer {token_a}"},
        )
        results = resp.json()["results"]

        # This assertion MUST pass — if it fails, you have a data leak
        tenant_b_docs = [r for r in results if "XYZ-SECRET-42" in r.get("text", "")]
        assert len(tenant_b_docs) == 0, "DATA LEAK: Tenant A can see Tenant B's data!"

@pytest.mark.asyncio
async def test_cannot_override_tenant_id_in_request_body():
    """Client should not be able to inject a different tenant_id."""
    token_a = create_jwt("user_a", "tenant_a", ["search"])

    async with AsyncClient(app=app, base_url="http://test") as client:
        resp = await client.post(
            "/search",
            json={
                "text": "secret",
                "tenant_id": "tenant_b",   # attacker tries to override
            },
            headers={"Authorization": f"Bearer {token_a}"},
        )
        # Server must ignore the tenant_id in body and use the one from JWT
        assert resp.status_code == 200
        for result in resp.json()["results"]:
            assert result["tenant_id"] == "tenant_a"   # only tenant_a data
```

---

## Noisy Neighbor Prevention

```
Problem: Tenant A sends 10,000 QPS and saturates the cluster.
         Tenant B (who is on plan) gets 5-second latency.

Solutions:

1. Per-tenant rate limiting
   Token bucket: 100 tokens, refill 100/second = max 100 QPS burst

2. Per-tenant request queues
   Separate Kafka topics per tenant → backlog doesn't bleed across

3. Resource quotas
   Limit ingestion batch size per tenant
   Limit concurrent searches per tenant

4. Tier-based dedicated resources
   Free/Pro → shared cluster with rate limits
   Enterprise → dedicated node pool
```

```python
import asyncio
from collections import defaultdict
import time

class TenantRateLimiter:
    """Token bucket rate limiter per tenant."""

    def __init__(self, rate: float, burst: float):
        self.rate = rate        # tokens per second
        self.burst = burst      # max burst size
        self._buckets: dict[str, float] = defaultdict(lambda: burst)
        self._last_refill: dict[str, float] = defaultdict(time.time)

    def is_allowed(self, tenant_id: str) -> bool:
        now = time.time()
        elapsed = now - self._last_refill[tenant_id]
        self._last_refill[tenant_id] = now

        # Refill tokens
        self._buckets[tenant_id] = min(
            self.burst,
            self._buckets[tenant_id] + elapsed * self.rate,
        )

        # Consume one token
        if self._buckets[tenant_id] >= 1.0:
            self._buckets[tenant_id] -= 1.0
            return True
        return False

# Plan-based rate limits
PLAN_LIMITS = {
    "free":       TenantRateLimiter(rate=10,   burst=20),
    "pro":        TenantRateLimiter(rate=100,  burst=200),
    "enterprise": TenantRateLimiter(rate=1000, burst=2000),
}

def get_rate_limiter(tenant_plan: str) -> TenantRateLimiter:
    return PLAN_LIMITS.get(tenant_plan, PLAN_LIMITS["free"])
```

---

## Namespace Isolation in Kubernetes

Each tenant tier maps to a Kubernetes namespace with ResourceQuota:

```yaml
# Namespace per enterprise tenant
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acme
  labels:
    tenant: acme
    tier: enterprise

---
# Resource quota: limit blast radius
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-acme-quota
  namespace: tenant-acme
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "32Gi"
    limits.cpu: "16"
    limits.memory: "64Gi"
    pods: "20"
    services: "5"

---
# Network policy: isolate tenant namespace from others
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-tenant
  namespace: tenant-acme
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: tenant-acme          # only from same namespace
        - namespaceSelector:
            matchLabels:
              name: shared-infrastructure  # allow API gateway
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: tenant-acme
        - namespaceSelector:
            matchLabels:
              name: shared-infrastructure
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Separate clusters | Strongest isolation, too expensive at 1000+ tenants |
| Separate collections | Good isolation, operational overhead at scale |
| Custom shard keys | Best for scale; always include shard_key_selector |
| Defense in depth | JWT + shard key + payload filter + middleware |
| Auto tests | Cross-tenant access tests MUST fail in CI |
| Rate limiting | Token bucket per tenant; plan-based limits |
| Noisy neighbor | Tier-based pools, per-tenant queues, quotas |

---

## Resources

- [Qdrant Multi-Tenancy Guide](https://qdrant.tech/documentation/tutorials/multiple-partitions/)
- [OWASP Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
