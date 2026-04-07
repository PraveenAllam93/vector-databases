# 5. Network Security — Controlling What Can Talk to What

---

## ELI5

Your application has many services: API, Qdrant, Redis, Postgres, Kafka. By default, all of them can talk to each other freely. That's a problem — if an attacker compromises one service, they can pivot to all others.

Network security = building walls between services. Each service can only talk to the specific services it needs to. Nothing more.

---

## Defense in Depth for Network

```
Layer 1: Cloud VPC (Virtual Private Cloud)
  No public internet access to internal services (Qdrant, Redis, Postgres)
  Only the API Gateway has a public IP

Layer 2: Security Groups / Firewall Rules
  Qdrant: only accept traffic from API pods and ingestion workers
  Redis:  only accept traffic from API pods
  Postgres: only accept traffic from API pods and ingestion workers

Layer 3: Kubernetes Network Policies
  Pod-level enforcement inside the cluster

Layer 4: mTLS (Service Mesh)
  Encrypt + authenticate all service-to-service traffic

Layer 5: API Gateway
  Rate limiting, WAF, DDoS protection at the edge
```

---

## Cloud VPC Architecture

```
Internet
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                           │
│                                                             │
│  ┌─────────────────────────────────────┐                   │
│  │  Public Subnet: 10.0.1.0/24          │                   │
│  │  ┌─────────────┐  ┌──────────────┐  │                   │
│  │  │  ALB (HTTPS) │  │  NAT Gateway  │  │                   │
│  │  │  port 443    │  │              │  │                   │
│  │  └──────┬──────┘  └──────────────┘  │                   │
│  └─────────┼───────────────────────────┘                   │
│            │                                               │
│  ┌─────────▼───────────────────────────┐                   │
│  │  Private Subnet: 10.0.2.0/24         │                   │
│  │  ┌───────────┐  ┌────────────────┐  │                   │
│  │  │  API Pods  │  │  Ingestion     │  │                   │
│  │  │  (FastAPI) │  │  Workers       │  │                   │
│  │  └─────┬─────┘  └───────┬────────┘  │                   │
│  └────────┼────────────────┼───────────┘                   │
│           │                │                               │
│  ┌────────▼────────────────▼───────────┐                   │
│  │  Data Subnet: 10.0.3.0/24            │                   │
│  │  ┌─────────┐  ┌───────┐  ┌───────┐  │                   │
│  │  │  Qdrant  │  │ Redis │  │  PG   │  │                   │
│  │  │ :6333/34 │  │ :6379 │  │ :5432 │  │                   │
│  │  └─────────┘  └───────┘  └───────┘  │                   │
│  └─────────────────────────────────────┘                   │
└────────────────────────────────────────────────────────────┘

Security Groups:
  alb-sg:          inbound 443 from 0.0.0.0/0
  api-sg:          inbound 8000 from alb-sg only
  qdrant-sg:       inbound 6333/6334 from api-sg, worker-sg only
  redis-sg:        inbound 6379 from api-sg only
  postgres-sg:     inbound 5432 from api-sg, worker-sg only
```

```terraform
# Terraform: Security Group for Qdrant
resource "aws_security_group" "qdrant" {
  name   = "qdrant-sg"
  vpc_id = aws_vpc.main.id

  # Only API and ingestion workers can reach Qdrant
  ingress {
    from_port       = 6333
    to_port         = 6334
    protocol        = "tcp"
    security_groups = [
      aws_security_group.api.id,
      aws_security_group.worker.id,
    ]
  }

  # Internal Qdrant cluster communication
  ingress {
    from_port = 6335   # Qdrant peer port
    to_port   = 6335
    protocol  = "tcp"
    self      = true   # Qdrant nodes can talk to each other
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Kubernetes Network Policies

By default, all pods can communicate with all other pods. Network Policies add firewall rules at the pod level.

**Important: Network Policies require a CNI plugin that supports them (Calico, Cilium, Weave). The default K8s networking (kubenet) does NOT enforce them.**

### Default Deny All

Start by blocking everything, then allow only what's needed:

```yaml
# namespace-level: deny all ingress and egress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: vector-search
spec:
  podSelector: {}   # applies to all pods in namespace
  policyTypes:
    - Ingress
    - Egress
```

### Allow Specific Service Communication

```yaml
# Qdrant: only allow API pods and ingestion workers
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: qdrant-allow
  namespace: vector-search
spec:
  podSelector:
    matchLabels:
      app: qdrant
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-service
        - podSelector:
            matchLabels:
              app: ingestion-worker
      ports:
        - protocol: TCP
          port: 6333
        - protocol: TCP
          port: 6334

    # Qdrant peer-to-peer clustering
    - from:
        - podSelector:
            matchLabels:
              app: qdrant
      ports:
        - protocol: TCP
          port: 6335

---
# Redis: only API pods can connect
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-allow
  namespace: vector-search
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-service
      ports:
        - protocol: TCP
          port: 6379

---
# API pods: allow ingress from the ingress controller only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
  namespace: vector-search
spec:
  podSelector:
    matchLabels:
      app: api-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8000

---
# Allow DNS (kube-dns) for all pods — required for name resolution
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: vector-search
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## Kubernetes Ingress with TLS

```yaml
# Ingress with TLS termination and rate limiting
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vector-search-ingress
  namespace: vector-search
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    # Rate limiting (nginx ingress)
    nginx.ingress.kubernetes.io/limit-rps: "100"          # 100 req/s per IP
    nginx.ingress.kubernetes.io/limit-connections: "10"   # 10 concurrent per IP
    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains";
      more_set_headers "X-XSS-Protection: 1; mode=block";
spec:
  tls:
    - hosts:
        - api.yourcompany.com
      secretName: api-tls    # auto-created by cert-manager
  rules:
    - host: api.yourcompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8000
```

---

## API Gateway Security

```
AWS API Gateway / Kong / Traefik at the edge:
  ✓ DDoS protection
  ✓ Rate limiting per IP / per API key
  ✓ WAF rules (SQL injection, XSS detection)
  ✓ Request validation (reject malformed requests before they reach your app)
  ✓ IP allowlisting (for internal services)
  ✓ TLS termination
```

```python
# FastAPI middleware: security headers + IP-based rate limiting
from fastapi import FastAPI, Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import time
from collections import defaultdict

app = FastAPI()

# IP Rate Limiting Middleware
class IPRateLimitMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, max_requests: int = 100, window_seconds: int = 60):
        super().__init__(app)
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self._requests: dict[str, list[float]] = defaultdict(list)

    async def dispatch(self, request: Request, call_next):
        client_ip = request.client.host

        now = time.time()
        window_start = now - self.window_seconds

        # Remove old timestamps outside the window
        self._requests[client_ip] = [
            ts for ts in self._requests[client_ip]
            if ts > window_start
        ]

        if len(self._requests[client_ip]) >= self.max_requests:
            return Response(
                content='{"detail": "Rate limit exceeded"}',
                status_code=429,
                media_type="application/json",
                headers={"Retry-After": str(self.window_seconds)},
            )

        self._requests[client_ip].append(now)
        return await call_next(request)

# Security Headers Middleware
class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Content-Security-Policy"] = "default-src 'none'"
        return response

app.add_middleware(IPRateLimitMiddleware, max_requests=100, window_seconds=60)
app.add_middleware(SecurityHeadersMiddleware)
```

---

## Input Validation (Prevent Injection)

Vector DB doesn't have SQL injection, but still has injection risks in:
- Collection names (path traversal)
- Payload filter fields (crafted filter JSON)
- Metadata fields (stored and returned to other users)

```python
import re
from fastapi import HTTPException

def validate_collection_name(name: str) -> str:
    """Collection names: alphanumeric + underscores only."""
    if not re.match(r'^[a-zA-Z0-9_-]{1,64}$', name):
        raise HTTPException(
            status_code=400,
            detail="Invalid collection name: must be alphanumeric, 1-64 chars",
        )
    return name

def validate_tenant_id(tenant_id: str) -> str:
    """Tenant IDs: alphanumeric + hyphens only."""
    if not re.match(r'^[a-zA-Z0-9-]{1,64}$', tenant_id):
        raise HTTPException(status_code=400, detail="Invalid tenant ID")
    return tenant_id

# Pydantic models with validation
from pydantic import BaseModel, field_validator

class SearchRequest(BaseModel):
    text: str
    top_k: int = 10
    filters: dict = {}

    @field_validator("text")
    def text_not_empty(cls, v):
        v = v.strip()
        if not v:
            raise ValueError("Query text cannot be empty")
        if len(v) > 10000:
            raise ValueError("Query text too long (max 10000 chars)")
        return v

    @field_validator("top_k")
    def top_k_reasonable(cls, v):
        if not 1 <= v <= 100:
            raise ValueError("top_k must be between 1 and 100")
        return v
```

---

## OWASP API Security Top 10 Checklist

```
1. Broken Object Level Auth    → Always filter by tenant_id, never trust IDs in path
2. Broken Authentication       → Use short-lived JWTs, validate all claims
3. Broken Object Property Auth → Only return fields the caller is allowed to see
4. Unrestricted Resource Cons  → Rate limit per tenant, validate top_k bounds
5. Broken Function Level Auth  → RBAC on every endpoint, not just sensitive ones
6. Unrestricted Access to Biz  → Use pagination, limit response sizes
7. SSRF                        → Validate all URLs, use allowlists for external calls
8. Security Misconfig           → Default deny network policies, no debug endpoints in prod
9. Improper Inventory Mgmt     → Document all endpoints, retire deprecated ones
10. Unsafe Consumption of APIs  → Validate all external API responses before trusting
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| VPC | No public access to data services; only API has public IP |
| Security Groups | Qdrant/Redis/PG only accessible from specific services |
| K8s NetworkPolicy | Default deny all; allowlist specific pod-to-pod routes |
| Ingress TLS | cert-manager auto-renews Let's Encrypt certs |
| Security headers | X-Frame-Options, HSTS, X-Content-Type-Options on all responses |
| Input validation | Sanitize collection names, tenant IDs, all user inputs |
| OWASP API Top 10 | Review checklist before every production deployment |

---

## Resources

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Calico Network Policy](https://docs.tigera.io/calico/latest/network-policy/) — most feature-rich CNI
- [cert-manager](https://cert-manager.io/) — automated TLS cert management
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [AWS VPC Security Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)
