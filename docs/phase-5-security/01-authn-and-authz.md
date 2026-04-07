# 1. Authentication & Authorization — Who Are You? What Can You Do?

---

## ELI5

**Authentication (AuthN)** = Proving who you are. Like showing your ID at the door.
**Authorization (AuthZ)** = What you're allowed to do once inside. Like a bouncer checking your wristband.

A system can have strong authentication but weak authorization — "you got in, but now you can do anything." Both must be solid.

---

## The Three Ways to Authenticate an API

### 1. API Keys (Simplest)

```
Client → sends header: "X-API-Key: sk-abc123"
Server → looks up key in DB → finds tenant_id → grants access
```

**Pros:** Simple, fast, stateless
**Cons:** No expiry by default, hard to revoke without breaking clients, no user identity

```python
# FastAPI API key authentication
from fastapi import Security, HTTPException, status
from fastapi.security import APIKeyHeader
from database import get_api_key_record

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def require_api_key(api_key: str = Security(api_key_header)):
    if not api_key:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API key required",
        )

    record = await get_api_key_record(api_key)  # DB lookup
    if not record or record.is_revoked:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or revoked API key",
        )

    return record  # contains tenant_id, permissions, rate_limit

@app.get("/search")
async def search(query: str, auth=Depends(require_api_key)):
    tenant_id = auth.tenant_id
    # ... search filtered by tenant_id
```

**API Key Best Practices:**
```
Format:     prefix + random bytes — e.g., "sk-" + 32 bytes (base64url)
Storage:    Store only the HASH (SHA-256), never plaintext
Show once:  Display to user only at creation; can't recover after
Rotation:   Support overlapping old + new key during transition
Scope:      Keys can be scoped (read-only, ingest-only, admin)
Audit:      Log every API key usage with timestamp + IP
```

**Generating a secure API key:**
```python
import secrets
import hashlib

def generate_api_key() -> tuple[str, str]:
    """Returns (plaintext_for_user, hash_for_storage)."""
    raw = secrets.token_urlsafe(32)     # 256 bits of entropy
    key = f"sk-{raw}"                   # prefix for recognizability
    key_hash = hashlib.sha256(key.encode()).hexdigest()
    return key, key_hash

# Usage
plaintext, stored_hash = generate_api_key()
# Show plaintext to user ONCE, store stored_hash in DB
```

---

### 2. JWT (JSON Web Tokens)

A JWT is a signed token that carries claims (user_id, tenant_id, permissions) inside it — the server doesn't need to look anything up in a database.

```
Structure: header.payload.signature
Example:   eyJhbGci...  .  eyJ1c2VyX2lk...  .  abc123sig...
           (base64url)      (base64url)          (HMAC/RSA)
```

**Payload (decoded):**
```json
{
  "sub": "user_123",
  "tenant_id": "acme_corp",
  "roles": ["search", "ingest"],
  "iat": 1717200000,
  "exp": 1717203600
}
```

```python
# JWT in FastAPI
import jwt
from datetime import datetime, timedelta, timezone
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

SECRET_KEY = "your-256-bit-secret"  # in prod: from Vault/Secrets Manager
ALGORITHM = "HS256"
bearer_scheme = HTTPBearer()

def create_jwt(user_id: str, tenant_id: str, roles: list[str]) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": user_id,
        "tenant_id": tenant_id,
        "roles": roles,
        "iat": now,
        "exp": now + timedelta(hours=1),  # short-lived
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def decode_jwt(token: str) -> dict:
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

async def require_jwt(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
) -> dict:
    return decode_jwt(credentials.credentials)

@app.post("/search")
async def search(query: SearchQuery, claims: dict = Depends(require_jwt)):
    tenant_id = claims["tenant_id"]   # extract from token — NEVER from request body
    roles = claims["roles"]
    if "search" not in roles:
        raise HTTPException(status_code=403, detail="Search not allowed")
    # ...
```

**JWT Security Rules:**
```
NEVER:
  ✗ Trust claims sent by the client (put tenant_id in request body)
  ✗ Use "none" algorithm (alg: none bypass attack)
  ✗ Use symmetric key across multiple services (use RS256/ES256 instead)
  ✗ Store sensitive data in payload (it's base64, not encrypted)

ALWAYS:
  ✓ Validate exp (expiry) and iat (issued at) claims
  ✓ Use short-lived tokens (15min-1hr) + refresh tokens
  ✓ Use RS256 (asymmetric) for microservices: sign with private key, verify with public key
  ✓ Validate algorithm explicitly (don't accept what client claims)
```

**RS256 (asymmetric) for microservices:**
```python
# Auth service (has private key)
import jwt
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa

def sign_token(payload: dict) -> str:
    with open("private.pem", "rb") as f:
        private_key = serialization.load_pem_private_key(f.read(), password=None)
    return jwt.encode(payload, private_key, algorithm="RS256")

# Other services (only have public key — can verify but not forge)
def verify_token(token: str) -> dict:
    with open("public.pem", "rb") as f:
        public_key = serialization.load_pem_public_key(f.read())
    return jwt.decode(token, public_key, algorithms=["RS256"])
```

---

### 3. OAuth2 (Delegated Authorization)

OAuth2 is a framework for "User authorizes App to act on their behalf."

```
Flow (Authorization Code):
  1. App redirects user to Identity Provider (Google, Okta, Auth0)
  2. User logs in and grants permissions
  3. IDP issues authorization code to App
  4. App exchanges code for access_token + refresh_token
  5. App uses access_token to call your API
  6. Your API validates token with IDP (introspection) or using IDP's public key (JWT)
```

```python
# OAuth2 with FastAPI + python-jose (token validation)
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    # Validate against your IDP (Auth0, Okta, Keycloak)
    payload = verify_with_idp_public_key(token)
    return {
        "user_id": payload["sub"],
        "tenant_id": payload.get("https://yourapp.com/tenant_id"),
        "roles": payload.get("https://yourapp.com/roles", []),
    }
```

**When to use what:**
```
API Keys:  Machine-to-machine, developer tools, simple integrations
JWT:       Stateless auth, microservices, mobile apps
OAuth2:    User login via third-party IDP, enterprise SSO, B2B
mTLS:      Service-to-service within Kubernetes (most secure)
```

---

## RBAC — Role-Based Access Control

Don't give every authenticated user the same permissions. Define roles, assign permissions to roles, assign roles to users/tenants.

### Role Hierarchy

```
Roles:
  viewer      → search (read-only)
  contributor → search + ingest
  admin       → search + ingest + manage collections + delete
  super_admin → all + manage users

Permissions:
  search          → POST /search, GET /collections
  ingest          → POST /ingest, PUT /documents
  manage          → POST /collections, DELETE /collections
  manage_users    → POST /users, DELETE /users, PATCH /users/roles
```

### RBAC Implementation

```python
from enum import Enum
from functools import wraps
from fastapi import HTTPException, status

class Permission(str, Enum):
    SEARCH = "search"
    INGEST = "ingest"
    MANAGE_COLLECTIONS = "manage_collections"
    DELETE = "delete"
    MANAGE_USERS = "manage_users"

ROLE_PERMISSIONS: dict[str, set[Permission]] = {
    "viewer":      {Permission.SEARCH},
    "contributor": {Permission.SEARCH, Permission.INGEST},
    "admin":       {Permission.SEARCH, Permission.INGEST,
                    Permission.MANAGE_COLLECTIONS, Permission.DELETE},
    "super_admin": set(Permission),  # all permissions
}

def require_permission(permission: Permission):
    """Dependency factory — checks a specific permission."""
    async def checker(claims: dict = Depends(require_jwt)):
        user_roles = claims.get("roles", [])
        allowed = any(
            permission in ROLE_PERMISSIONS.get(role, set())
            for role in user_roles
        )
        if not allowed:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Permission '{permission}' required",
            )
        return claims
    return checker

# Usage on endpoints
@app.post("/search")
async def search(
    query: SearchQuery,
    claims: dict = Depends(require_permission(Permission.SEARCH)),
):
    ...

@app.delete("/collections/{name}")
async def delete_collection(
    name: str,
    claims: dict = Depends(require_permission(Permission.DELETE)),
):
    ...
```

### Qdrant Built-in API Keys

Qdrant itself supports key-based access control:

```yaml
# qdrant config.yaml
service:
  api_key: "your-master-api-key"           # required for all requests
  read_only_api_key: "your-read-only-key"  # only for read operations
  jwt_rbac: true                           # enable JWT-based RBAC
```

```python
# In Qdrant JWT payload, you can scope to specific collections
{
  "sub": "service-account",
  "exp": 1717203600,
  "access": [
    {
      "collection": "documents",
      "access": "r"   # r = read-only, rw = read-write, m = manage
    },
    {
      "collection": "logs",
      "access": "rw"
    }
  ]
}
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| API Keys | Simple, machine-to-machine; store only the hash |
| JWT | Stateless, carries claims; use RS256 for microservices |
| OAuth2 | User-facing, delegated auth via IDP |
| RBAC | Roles → permissions; enforce at endpoint level |
| Never trust client | Always extract tenant_id from token, not request body |
| Short-lived tokens | Access tokens: 15min–1hr; rotate with refresh tokens |

---

## Resources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [JWT.io Debugger](https://jwt.io/) — decode and inspect JWTs
- [FastAPI Security Docs](https://fastapi.tiangolo.com/tutorial/security/) — official guide
- [Qdrant JWT RBAC](https://qdrant.tech/documentation/guides/security/) — Qdrant-specific auth
