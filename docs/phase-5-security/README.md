# Phase 5: Security — Production Hardening

## Reading Order

| # | Topic | File | Key Takeaway |
|---|-------|------|-------------|
| 1 | [AuthN & AuthZ](01-authn-and-authz.md) | API keys (hash, don't store), JWT (RS256 for microservices), OAuth2, RBAC enforcement at endpoint level |
| 2 | [Multi-Tenant Isolation](02-multi-tenant-isolation.md) | Defense in depth: JWT → shard key → payload filter → middleware → audit → tests; cross-tenant test must FAIL |
| 3 | [Encryption](03-encryption.md) | Disk encryption (AES-256 via cloud), field encryption for PII, TLS for external, mTLS for internal services |
| 4 | [Secrets Management](04-secrets-management.md) | Never hardcode; Vault with K8s auth for dynamic secrets; GitHub OIDC instead of static tokens |
| 5 | [Network Security](05-network-security.md) | VPC with private subnets, K8s NetworkPolicy default-deny-all, security headers, input validation |

---

## Interview Quick Reference

**"How do you authenticate API requests?"**
→ API keys (stored as SHA-256 hash, never plaintext) for machine-to-machine. JWT (RS256, short-lived 1hr) for user-facing apps — extract tenant_id from token claims, never from the request body. OAuth2 for enterprise SSO. mTLS inside the Kubernetes cluster.

**"How do you prevent data leaks in multi-tenancy?"**
→ Defense in depth: (1) extract tenant_id from JWT in middleware, (2) shard_key_selector routes query to tenant's shard only, (3) payload filter as belt-and-suspenders, (4) audit log every query, (5) automated tests that try cross-tenant access and MUST return zero results.

**"What encryption do you use?"**
→ At rest: AES-256 disk encryption via cloud (EBS, GCP PD) for all data. Application-level field encryption (Fernet) for PII/sensitive payload fields. In transit: TLS 1.2+ for external (terminate at ALB), mTLS for internal service-to-service (Istio handles transparently). Keys in Vault / AWS Secrets Manager, never hardcoded.

**"How do you manage secrets in production?"**
→ Vault with Kubernetes auth — pods authenticate using their SA token, receive short-lived Vault tokens, fetch secrets dynamically. No static credentials anywhere. Dynamic DB credentials (rotate every 1hr). GitHub Actions uses OIDC to assume IAM roles instead of stored AWS keys. Secret scanning (trufflehog) in CI blocks accidental commits.

**"What are your network security controls?"**
→ VPC: data services (Qdrant, Redis) in private subnets, no public IP. Security groups: Qdrant only accepts traffic from API pods and workers. K8s NetworkPolicy: default-deny-all + explicit allowlist. Ingress: TLS with cert-manager auto-renewal. mTLS for service mesh (Istio). Input validation on all user-supplied data.

**"Walk me through what happens when a JWT token is stolen."**
→ Short expiry (1hr) limits blast radius. Attacker can impersonate that user for up to 1hr. Mitigation: (1) detect anomalous usage (different IP, unusual QPS) and revoke; (2) use refresh token rotation — if refresh token is used twice, invalidate both (detect theft); (3) tenant isolation limits scope — stolen token only accesses that tenant's data; (4) audit logs capture all actions.

**"How do you handle RBAC for different tenant tiers?"**
→ Roles: viewer (search-only), contributor (search+ingest), admin (manage collections), super_admin. Permissions attached to roles, not users directly. Enforced via FastAPI dependencies: `Depends(require_permission(Permission.SEARCH))`. Tenant plan (free/pro/enterprise) maps to rate limits, not roles — they're separate concerns.

---

## Security Checklist for Production

```
Authentication:
  □ API keys stored as SHA-256 hash
  □ JWT expiry <= 1 hour
  □ JWT uses RS256 (asymmetric), not HS256
  □ Algorithm explicitly validated (reject "none")
  □ tenant_id extracted from JWT, never from request body

Authorization:
  □ RBAC enforced on every endpoint
  □ Qdrant API key set in qdrant config
  □ Qdrant JWT RBAC enabled for collection-scoped access

Multi-Tenancy:
  □ shard_key_selector used on every query
  □ tenant_id filter in every query (belt+suspenders)
  □ Cross-tenant access automated test in CI/CD
  □ Per-tenant rate limiting configured

Encryption:
  □ Disk encryption enabled (EBS/GCE with KMS)
  □ TLS 1.2+ for all external traffic
  □ mTLS for internal cluster traffic
  □ PII fields encrypted before storage
  □ Secrets in Vault / Secrets Manager

Network:
  □ Data services in private subnets (no public IPs)
  □ Security groups: Qdrant only accessible from app services
  □ K8s NetworkPolicy: default-deny-all applied
  □ Security headers on all responses
  □ Input validation on all user-supplied data

CI/CD:
  □ Secret scanning (trufflehog) on PRs
  □ No secrets in environment variables in pipelines (use OIDC)
  □ No debug endpoints in production builds
  □ Dependency vulnerability scanning (trivy / snyk)
```
