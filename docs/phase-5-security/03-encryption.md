# 3. Encryption — Protecting Data at Rest and in Transit

---

## ELI5

**At Rest** = data sitting on a hard drive. If someone steals the disk, they get nothing.
**In Transit** = data moving over a network. If someone intercepts the connection, they get nothing.

Encryption doesn't prevent access breaches — it limits what an attacker can do WITH the data after a breach.

---

## Encryption at Rest

### Layer 1: Disk Encryption (Infrastructure Level)

The OS/cloud encrypts the entire volume. Qdrant doesn't do anything special — the disk is encrypted transparently.

```
AWS:  EBS volumes with AES-256 (enable by default in AWS console or Terraform)
GCP:  Persistent Disks with AES-256 (enabled by default for all GCP projects)
K8s:  Use StorageClass with encrypted: "true"
```

```yaml
# Kubernetes StorageClass with EBS encryption
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/mrk-abc123"  # your CMK
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain  # don't auto-delete on PVC deletion
```

```terraform
# Terraform: EBS-encrypted Qdrant volume
resource "aws_ebs_volume" "qdrant_data" {
  availability_zone = "us-east-1a"
  size              = 500
  type              = "gp3"
  encrypted         = true
  kms_key_id        = aws_kms_key.qdrant.arn

  tags = {
    Name = "qdrant-data"
  }
}

resource "aws_kms_key" "qdrant" {
  description             = "Qdrant data encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}
```

### Layer 2: Application-Level Encryption (For Sensitive Payloads)

If you need to encrypt specific payload fields (PII, medical data), encrypt before storing in Qdrant:

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.primitives import hashes
import base64
import os

class FieldEncryption:
    """AES-128 (Fernet) encryption for sensitive payload fields."""

    def __init__(self, key: bytes):
        self.fernet = Fernet(key)

    @classmethod
    def from_env(cls) -> "FieldEncryption":
        """Load key from environment variable (set by Secrets Manager)."""
        raw_key = os.environ["FIELD_ENCRYPTION_KEY"]  # 32 bytes, base64url
        return cls(raw_key.encode())

    def encrypt(self, plaintext: str) -> str:
        return self.fernet.encrypt(plaintext.encode()).decode()

    def decrypt(self, ciphertext: str) -> str:
        return self.fernet.decrypt(ciphertext.encode()).decode()

# Usage: encrypt PII before upserting
encryptor = FieldEncryption.from_env()

point = PointStruct(
    id=generate_id(doc_id),
    vector=embed(text),
    payload={
        "text": text,                                    # searchable
        "email": encryptor.encrypt(user_email),         # encrypted PII
        "phone": encryptor.encrypt(user_phone),         # encrypted PII
        "tenant_id": tenant_id,
    }
)
client.upsert(collection_name="documents", points=[point])

# Decrypt on retrieval
result = client.retrieve(collection_name="documents", ids=[point_id])
email = encryptor.decrypt(result[0].payload["email"])
```

**Important**: You CANNOT filter on encrypted fields (you'd need to decrypt all points to compare). Design carefully — encrypt only what truly needs it.

---

## Encryption in Transit

### TLS for External Traffic

All client-to-API traffic must use TLS 1.2+.

```yaml
# Qdrant with TLS enabled
service:
  tls:
    enable: true
    cert: /certs/server.crt
    key: /certs/server.key

# In docker-compose
services:
  qdrant:
    image: qdrant/qdrant
    volumes:
      - ./certs:/certs:ro
      - qdrant_storage:/qdrant/storage
    environment:
      - QDRANT__SERVICE__TLS_ENABLE=true
      - QDRANT__SERVICE__TLS_CERT=/certs/server.crt
      - QDRANT__SERVICE__TLS_KEY=/certs/server.key
```

**Using TLS in Qdrant Python client:**
```python
from qdrant_client import QdrantClient

# With self-signed cert
client = QdrantClient(
    host="qdrant.internal",
    port=6333,
    https=True,
    prefix="",
    verify="/path/to/ca.crt",  # CA cert for verification
)

# With valid cert (Let's Encrypt)
client = QdrantClient(
    url="https://qdrant.yourcompany.com",
    api_key="your-api-key",
)
```

**Generating TLS certificates (dev/staging):**
```bash
# Self-signed cert for dev
openssl req -x509 -newkey rsa:4096 \
  -keyout server.key \
  -out server.crt \
  -days 365 \
  -nodes \
  -subj "/CN=qdrant.internal" \
  -addext "subjectAltName=DNS:qdrant.internal,DNS:localhost,IP:127.0.0.1"

# Production: use cert-manager in Kubernetes (auto-renews Let's Encrypt certs)
```

### mTLS — Mutual TLS (Service-to-Service)

Standard TLS: client verifies server identity.
mTLS: BOTH sides verify each other. Server rejects connections from unknown clients.

```
Why mTLS for internal services?
  - Even if attacker gets inside the cluster, they can't call Qdrant directly
  - No API key needed; the certificate IS the identity
  - Zero-trust networking: nothing is trusted just because it's inside the cluster
```

```
PKI Setup:
  1. Create a Root CA (your organization's CA)
  2. Issue server cert to Qdrant (signed by Root CA)
  3. Issue client cert to each service (API, ingestion workers)
  4. All parties have Root CA's public key → can verify each other
```

```bash
# Create Root CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -out ca.crt -subj "/CN=Internal CA"

# Create Qdrant server cert
openssl genrsa -out qdrant.key 2048
openssl req -new -key qdrant.key -out qdrant.csr -subj "/CN=qdrant"
openssl x509 -req -in qdrant.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out qdrant.crt -days 365 -sha256

# Create client cert for API service
openssl genrsa -out api-service.key 2048
openssl req -new -key api-service.key -out api-service.csr -subj "/CN=api-service"
openssl x509 -req -in api-service.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out api-service.crt -days 365 -sha256
```

```python
# Python client with mTLS
import ssl
import httpx

# Create SSL context with mTLS
ssl_ctx = ssl.create_default_context(ssl.Purpose.SERVER_AUTH, cafile="ca.crt")
ssl_ctx.load_cert_chain(certfile="api-service.crt", keyfile="api-service.key")
ssl_ctx.verify_mode = ssl.CERT_REQUIRED  # require server cert verification

client = QdrantClient(
    url="https://qdrant.internal:6333",
    # httpx transport with mTLS
    # (for custom SSL contexts in qdrant_client, use http_client parameter)
)
```

### mTLS with Istio (Kubernetes Service Mesh)

Instead of managing certs manually, use Istio/Linkerd to handle mTLS transparently:

```yaml
# Istio: enforce mTLS across the entire namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: vector-search
spec:
  mtls:
    mode: STRICT   # reject all non-mTLS connections

---
# Allow specific services to communicate
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: qdrant-access
  namespace: vector-search
spec:
  selector:
    matchLabels:
      app: qdrant
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/vector-search/sa/api-service"
              - "cluster.local/ns/vector-search/sa/ingestion-worker"
```

With this setup, only `api-service` and `ingestion-worker` service accounts can reach Qdrant. Everything else is blocked — even inside the cluster.

---

## TLS in FastAPI

```python
import uvicorn

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8443,
        ssl_certfile="./certs/server.crt",
        ssl_keyfile="./certs/server.key",
        # For mTLS (verify client certs):
        ssl_ca_certs="./certs/ca.crt",
        ssl_cert_reqs=ssl.CERT_REQUIRED,  # require client cert
    )
```

**Production pattern:** Don't terminate TLS in FastAPI. Terminate at the load balancer / Ingress, then use mTLS for internal service-to-service.

```
Internet → [TLS terminated at ALB] → [mTLS] → API pods → [mTLS] → Qdrant
```

---

## Encryption Key Management

### Never Hardcode Keys

```python
# BAD — hardcoded key
FIELD_ENCRYPTION_KEY = "hardcoded-key-123"  # attacker finds in git blame

# GOOD — from environment
FIELD_ENCRYPTION_KEY = os.environ["FIELD_ENCRYPTION_KEY"]

# BEST — from secrets manager (auto-rotates, audit trail)
import boto3

def get_encryption_key() -> bytes:
    client = boto3.client("secretsmanager", region_name="us-east-1")
    response = client.get_secret_value(SecretId="prod/qdrant/encryption-key")
    return response["SecretString"].encode()
```

### Kubernetes Secrets

```yaml
# Store TLS certs and API keys as K8s Secrets
apiVersion: v1
kind: Secret
metadata:
  name: qdrant-tls
  namespace: vector-search
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>

---
# Reference in Qdrant StatefulSet
containers:
  - name: qdrant
    volumeMounts:
      - name: tls-certs
        mountPath: /certs
        readOnly: true
volumes:
  - name: tls-certs
    secret:
      secretName: qdrant-tls
```

**Important: K8s Secrets are base64 encoded, NOT encrypted by default.**

For production, enable encryption at rest for etcd (where K8s stores secrets):

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}  # fallback
```

Or use **External Secrets Operator** with AWS Secrets Manager / HashiCorp Vault:

```yaml
# ExternalSecret: fetch from AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: qdrant-api-key
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-store
    kind: ClusterSecretStore
  target:
    name: qdrant-api-key   # K8s Secret name to create
  data:
    - secretKey: api_key
      remoteRef:
        key: prod/qdrant/api-key
        property: api_key
```

---

## Data Classification

Know what you're protecting and apply proportional controls:

```
Classification:   Public        Internal       Confidential    Restricted
                  (marketing)   (doc text)     (PII)           (medical/legal)

At Rest:          Optional      Disk encrypt   Disk + field    Disk + field
                                               encrypt         encrypt + HSM

In Transit:       Optional      TLS            TLS required    mTLS required

Access:           Anyone        API key        RBAC + audit    RBAC + MFA + audit

Retention:        Forever       Policy-based   Minimize        Strict + deletion

Examples:         Docs text     Embeddings,    Email, names    SSNs, health data
                                chunk metadata
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Disk encryption | AES-256 at infrastructure level; enable by default in cloud |
| Field encryption | For PII in payloads; can't filter on encrypted fields |
| TLS | Mandatory for all external traffic; terminate at LB |
| mTLS | Service-to-service inside cluster; certificate = identity |
| Istio/Linkerd | Handles mTLS transparently without code changes |
| Key management | Never hardcode; use Vault/Secrets Manager |
| K8s Secrets | base64 only; use encryption config or External Secrets Operator |

---

## Resources

- [Qdrant TLS Configuration](https://qdrant.tech/documentation/guides/security/)
- [Istio mTLS](https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/)
- [cert-manager](https://cert-manager.io/docs/) — automated TLS cert management in K8s
- [External Secrets Operator](https://external-secrets.io/latest/) — K8s ↔ Vault/AWS SM
- [OWASP Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)
