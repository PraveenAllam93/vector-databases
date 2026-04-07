# 4. Secrets Management — No Hardcoded Credentials, Ever

---

## ELI5

A secret is anything that grants access: API keys, passwords, database URLs, TLS certificates. If you put a secret in your code, everyone who can read your code can impersonate you. That includes GitHub, colleagues, CI/CD systems, and attackers who find old commits.

The rule: **secrets never touch source code.** Not even in `.env` files that are "gitignored."

---

## The Secrets Anti-Patterns (What NOT to Do)

```python
# NEVER DO THESE:

# Anti-pattern 1: Hardcoded in code
QDRANT_API_KEY = "sk-abc123def456"

# Anti-pattern 2: In .env file committed to git
# .env:
# QDRANT_API_KEY=sk-abc123def456

# Anti-pattern 3: In Docker Compose plaintext
environment:
  - QDRANT_API_KEY=sk-abc123def456

# Anti-pattern 4: In K8s YAML (even if not committed)
env:
  - name: QDRANT_API_KEY
    value: "sk-abc123def456"

# Anti-pattern 5: In CI/CD logs
# run: echo "API key is $QDRANT_API_KEY"  ← prints to logs
```

---

## Secrets Hierarchy

```
Development:    .env file (local only, gitignored) → simple but risky
Staging:        K8s Secrets (encrypted etcd)
Production:     HashiCorp Vault / AWS Secrets Manager / GCP Secret Manager
CI/CD:          GitHub Actions Secrets / CircleCI Contexts
```

---

## Option 1: Environment Variables (Minimum Bar)

```bash
# .env (NEVER commit this — add to .gitignore)
QDRANT_API_KEY=sk-your-key
QDRANT_URL=https://qdrant.internal:6333
OPENAI_API_KEY=sk-openai-...
REDIS_URL=redis://redis:6379
POSTGRES_URL=postgresql://user:pass@db:5432/mydb
```

```python
import os
from dotenv import load_dotenv

load_dotenv()  # load .env for local dev only

class Settings:
    qdrant_url: str = os.environ["QDRANT_URL"]
    qdrant_api_key: str = os.environ["QDRANT_API_KEY"]
    openai_api_key: str = os.environ["OPENAI_API_KEY"]
    redis_url: str = os.environ["REDIS_URL"]

# Use os.environ["KEY"] (raises error if missing) not os.getenv("KEY") (returns None silently)
```

---

## Option 2: Kubernetes Secrets (Staging/Prod)

```bash
# Create secret from literal values
kubectl create secret generic qdrant-secrets \
  --namespace=vector-search \
  --from-literal=api_key=sk-abc123 \
  --from-literal=tls_cert="$(cat server.crt)"

# Create from file
kubectl create secret generic qdrant-tls \
  --namespace=vector-search \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key
```

```yaml
# Reference in pod spec — as env var
env:
  - name: QDRANT_API_KEY
    valueFrom:
      secretKeyRef:
        name: qdrant-secrets
        key: api_key

# Reference as mounted file (for TLS certs)
volumeMounts:
  - name: qdrant-tls
    mountPath: /certs
    readOnly: true
volumes:
  - name: qdrant-tls
    secret:
      secretName: qdrant-tls
      defaultMode: 0400   # owner read-only
```

**Encrypt K8s Secrets at rest:**
```bash
# Enable encryption at rest for etcd
kubectl get pods -n kube-system | grep kube-apiserver

# Add --encryption-provider-config flag to kube-apiserver
# (managed clusters: EKS/GKE do this automatically with KMS integration)
```

---

## Option 3: HashiCorp Vault (Production Standard)

Vault is the industry-standard secrets management solution. It:
- Stores secrets encrypted
- Issues short-lived credentials (dynamic secrets)
- Has an audit log for every secret access
- Automatically rotates secrets
- Supports many backends: K8s auth, AWS IAM, LDAP, JWT

### Setup and Basic Usage

```bash
# Start Vault (dev mode for local testing)
docker run --rm -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=mytoken \
  hashicorp/vault:latest

# Store a secret
export VAULT_ADDR=http://localhost:8200
export VAULT_TOKEN=mytoken

vault kv put secret/qdrant api_key="sk-abc123" url="https://qdrant.internal"
vault kv get secret/qdrant
```

```python
import hvac

class VaultClient:
    def __init__(self):
        self.client = hvac.Client(
            url=os.environ["VAULT_ADDR"],
            token=os.environ["VAULT_TOKEN"],  # or use K8s service account auth
        )

    def get_secret(self, path: str) -> dict:
        response = self.client.secrets.kv.v2.read_secret_version(path=path)
        return response["data"]["data"]

# Usage
vault = VaultClient()
secrets = vault.get_secret("qdrant")
qdrant_api_key = secrets["api_key"]
qdrant_url = secrets["url"]
```

### Kubernetes Auth with Vault (No Static Tokens)

The best pattern: pods authenticate to Vault using their Kubernetes service account, not a static token.

```yaml
# Kubernetes auth: pod uses its SA token to get Vault token
# 1. Enable K8s auth in Vault
vault auth enable kubernetes

vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# 2. Create Vault policy
vault policy write qdrant-reader - <<EOF
path "secret/data/qdrant" {
  capabilities = ["read"]
}
EOF

# 3. Bind K8s service account to Vault policy
vault write auth/kubernetes/role/api-service \
    bound_service_account_names=api-service \
    bound_service_account_namespaces=vector-search \
    policies=qdrant-reader \
    ttl=1h
```

```python
# In the API pod — authenticate with service account token
import hvac

def get_vault_client() -> hvac.Client:
    client = hvac.Client(url=os.environ["VAULT_ADDR"])

    # Read K8s service account token (injected by K8s)
    with open("/var/run/secrets/kubernetes.io/serviceaccount/token") as f:
        sa_token = f.read()

    # Authenticate with Vault using K8s auth
    response = client.auth.kubernetes.login(
        role="api-service",
        jwt=sa_token,
    )
    client.token = response["auth"]["client_token"]
    return client
```

### Dynamic Secrets (Postgres Example)

Vault can generate short-lived database credentials on demand — no static DB passwords.

```bash
# Enable database secrets engine
vault secrets enable database

vault write database/config/mydb \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mydb" \
    allowed_roles="api-service" \
    username="vault-admin" \
    password="vault-admin-password"

vault write database/roles/api-service \
    db_name=mydb \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

```python
# Get fresh DB credentials at startup (auto-rotates)
def get_db_credentials() -> tuple[str, str]:
    vault = get_vault_client()
    lease = vault.secrets.database.generate_credentials("api-service")
    username = lease["data"]["username"]
    password = lease["data"]["password"]
    return username, password

username, password = get_db_credentials()
db_url = f"postgresql://{username}:{password}@postgres:5432/mydb"
```

---

## Option 4: AWS Secrets Manager (Cloud-Native)

For teams on AWS, Secrets Manager integrates natively with IAM and auto-rotates secrets.

```python
import boto3
import json
from functools import lru_cache

@lru_cache(maxsize=None)
def get_secret(secret_name: str) -> dict:
    """Fetch and cache secret from AWS Secrets Manager."""
    client = boto3.client("secretsmanager", region_name="us-east-1")
    response = client.get_secret_value(SecretId=secret_name)

    if "SecretString" in response:
        return json.loads(response["SecretString"])
    else:
        raise ValueError("Binary secrets not supported here")

# Usage
qdrant_secrets = get_secret("prod/qdrant/credentials")
api_key = qdrant_secrets["api_key"]
qdrant_url = qdrant_secrets["url"]
```

```terraform
# Terraform: create secret + auto-rotation
resource "aws_secretsmanager_secret" "qdrant_api_key" {
  name                    = "prod/qdrant/api-key"
  description             = "Qdrant API key for production"
  recovery_window_in_days = 30

  tags = {
    Environment = "production"
    Service     = "vector-search"
  }
}

resource "aws_secretsmanager_secret_version" "qdrant_api_key" {
  secret_id = aws_secretsmanager_secret.qdrant_api_key.id
  secret_string = jsonencode({
    api_key = var.qdrant_api_key
    url     = var.qdrant_url
  })
}

# IAM policy: only allow vector-search pods to read this secret
resource "aws_iam_policy" "read_qdrant_secret" {
  name = "read-qdrant-secret"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["secretsmanager:GetSecretValue"]
      Resource = [aws_secretsmanager_secret.qdrant_api_key.arn]
    }]
  })
}
```

---

## GitHub Actions Secrets (CI/CD)

```yaml
# GitHub repo → Settings → Secrets and variables → Actions → New secret

# .github/workflows/deploy.yml
name: Deploy

env:
  QDRANT_URL: ${{ secrets.QDRANT_URL }}
  QDRANT_API_KEY: ${{ secrets.QDRANT_API_KEY }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to production
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
        run: |
          kubectl apply -f k8s/

      # NEVER do this:
      # - run: echo "API key: $QDRANT_API_KEY"  ← leaks to logs
      #
      # GitHub auto-masks secrets in logs, but still don't print them
```

**Using OIDC instead of long-lived tokens (preferred):**
```yaml
# GitHub Actions → AWS without storing AWS credentials
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    steps:
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsDeployRole
          aws-region: us-east-1
          # No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY needed!

      - name: Deploy
        run: aws eks update-kubeconfig --name my-cluster
```

---

## Secret Rotation

```
Rotation Strategy:
  API Keys:         Rotate every 90 days (or after any leak suspicion)
  JWT signing keys: Rotate every 30 days (support overlapping keys during transition)
  DB passwords:     Rotate every 30 days (Vault handles automatically)
  TLS certs:        Rotate before expiry (cert-manager auto-renews)
  Encryption keys:  Rotate annually + re-encrypt data (expensive, plan ahead)

Zero-downtime rotation:
  1. Issue new key
  2. Update all services to accept BOTH old and new key
  3. Update all services to SIGN/SEND new key
  4. Monitor: confirm old key has zero usage
  5. Revoke old key
```

```python
# Support multiple valid JWT signing keys (during rotation)
class MultiKeyJWTVerifier:
    def __init__(self, active_keys: list[str]):
        """active_keys: list of valid signing keys, newest first."""
        self.keys = active_keys

    def verify(self, token: str) -> dict:
        for key in self.keys:
            try:
                return jwt.decode(token, key, algorithms=["HS256"])
            except jwt.InvalidSignatureError:
                continue   # try next key
        raise jwt.InvalidTokenError("Token invalid with all known keys")
```

---

## Secret Scanning in CI

Add secret scanning to prevent accidental commits:

```yaml
# .github/workflows/security.yml
- name: Scan for secrets (trufflehog)
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: main
    head: HEAD
    extra_args: --only-verified
```

```bash
# Pre-commit hook: block secret commits locally
pip install detect-secrets
detect-secrets scan > .secrets.baseline

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Never hardcode | Secrets in code = compromised code = compromised secrets |
| K8s Secrets | base64 only; enable etcd encryption; use External Secrets Operator |
| Vault | Dynamic secrets + K8s auth = no static credentials |
| AWS SM | Cloud-native; IAM-scoped access; auto-rotation |
| GitHub Actions | Use repository/environment secrets; prefer OIDC over static tokens |
| Rotation | Every 90 days for keys; support overlapping keys during rotation |
| Scanning | trufflehog + detect-secrets prevent accidental commits |

---

## Resources

- [HashiCorp Vault](https://developer.hashicorp.com/vault/docs) — comprehensive secrets management
- [External Secrets Operator](https://external-secrets.io/) — sync Vault/AWS SM to K8s Secrets
- [TruffleHog](https://github.com/trufflesecurity/trufflehog) — secret scanning
- [detect-secrets](https://github.com/Yelp/detect-secrets) — pre-commit hook for secrets
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/) — AWS-native option
