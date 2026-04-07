# 4. Cloud Deployment — AWS & GCP Production Setup

---

## ELI5

You can run Kubernetes on your own servers, but that's like building your own data center. Cloud providers (AWS, GCP) give you managed Kubernetes (EKS, GKE) where they handle the control plane, node updates, and networking. You focus on your application.

---

## AWS Deployment (EKS)

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  AWS Account: us-east-1                                       │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  VPC: 10.0.0.0/16                                       │  │
│  │                                                         │  │
│  │  Public Subnets (ALB, NAT)                              │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐             │  │
│  │  │ 10.0.1.0 │  │ 10.0.2.0 │  │ 10.0.3.0 │  AZ 1/2/3  │  │
│  │  └──────────┘  └──────────┘  └──────────┘             │  │
│  │                                                         │  │
│  │  Private Subnets (EKS Nodes)                            │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐             │  │
│  │  │ 10.0.4.0 │  │ 10.0.5.0 │  │ 10.0.6.0 │  AZ 1/2/3  │  │
│  │  └──────────┘  └──────────┘  └──────────┘             │  │
│  │                                                         │  │
│  │  EKS Cluster: vector-search-prod                        │  │
│  │  ├── Node Group: api-workers    (c5.2xlarge, 3-20)     │  │
│  │  └── Node Group: qdrant-nodes   (r6i.4xlarge, 3-6)    │  │
│  │                                                         │  │
│  │  Managed Services:                                      │  │
│  │  ├── ElastiCache Redis (Multi-AZ)                      │  │
│  │  ├── ECR (container registry)                           │  │
│  │  ├── Secrets Manager (credentials)                      │  │
│  │  ├── KMS (encryption keys)                              │  │
│  │  └── CloudWatch (logs + metrics)                        │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Terraform — Complete AWS Setup

```hcl
# main.tf
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket = "your-terraform-state"
    key    = "vector-search/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}

# ─── VPC ─────────────────────────────────────────────────────────────────────
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = false   # one per AZ for HA

  # Tags required by EKS
  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }
}

# ─── EKS Cluster ─────────────────────────────────────────────────────────────
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "${var.project_name}-prod"
  cluster_version = "1.31"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # Managed node groups
  eks_managed_node_groups = {
    # API workers: burstable compute
    api_workers = {
      instance_types = ["c5.2xlarge"]
      min_size       = 3
      desired_size   = 3
      max_size       = 20
      disk_size      = 50
      labels = { role = "api" }
    }

    # Qdrant nodes: memory-optimized
    qdrant_nodes = {
      instance_types = ["r6i.4xlarge"]   # 128GB RAM
      min_size       = 3
      desired_size   = 3
      max_size       = 6
      disk_size      = 500
      labels = { role = "qdrant" }
      taints = [{
        key    = "dedicated"
        value  = "qdrant"
        effect = "NO_SCHEDULE"    # only Qdrant pods schedule here
      }]
    }
  }

  # OIDC provider for service accounts → IAM roles
  enable_irsa = true
}

# ─── ECR ─────────────────────────────────────────────────────────────────────
resource "aws_ecr_repository" "api" {
  name                 = "vector-search-api"
  image_tag_mutability = "IMMUTABLE"   # can't overwrite tags (security)

  image_scanning_configuration {
    scan_on_push = true   # auto-scan with ECR Basic scanning
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }
}

# Auto-delete old images (keep last 20 + all prod tags)
resource "aws_ecr_lifecycle_policy" "api" {
  repository = aws_ecr_repository.api.name
  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 20 images"
      selection = {
        tagStatus   = "untagged"
        countType   = "imageCountMoreThan"
        countNumber = 20
      }
      action = { type = "expire" }
    }]
  })
}

# ─── ElastiCache Redis ────────────────────────────────────────────────────────
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "${var.project_name}-redis"
  description                = "Redis cache for vector search"
  node_type                  = "cache.r6g.large"
  num_node_groups            = 1       # single shard
  replicas_per_node_group    = 2       # 1 primary + 2 replicas
  automatic_failover_enabled = true    # auto-promote replica on primary failure
  multi_az_enabled           = true
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token
  engine_version             = "7.1"
  subnet_group_name          = aws_elasticache_subnet_group.redis.name
  security_group_ids         = [aws_security_group.redis.id]
}

# ─── Secrets Manager ─────────────────────────────────────────────────────────
resource "aws_secretsmanager_secret" "qdrant_api_key" {
  name                    = "prod/qdrant/api-key"
  recovery_window_in_days = 30
}

# IAM role for API pods to read secrets (via IRSA)
resource "aws_iam_role" "api_pod" {
  name = "${var.project_name}-api-pod"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = module.eks.oidc_provider_arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${module.eks.oidc_provider}:sub" = "system:serviceaccount:vector-search:api-service"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "api_pod_secrets" {
  role = aws_iam_role.api_pod.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["secretsmanager:GetSecretValue"]
      Resource = [
        aws_secretsmanager_secret.qdrant_api_key.arn,
        # add other secrets here
      ]
    }]
  })
}
```

---

## GCP Deployment (GKE)

```hcl
# GCP / GKE equivalent setup
terraform {
  required_providers {
    google = { source = "hashicorp/google", version = "~> 5.0" }
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}

# ─── GKE Autopilot (simpler, GCP manages nodes) ───────────────────────────
resource "google_container_cluster" "primary" {
  name     = "vector-search-prod"
  location = "us-central1"

  # Autopilot: GCP manages nodes, autoscaling, and security
  enable_autopilot = true

  # Enable Workload Identity (like IRSA on AWS)
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Enable Private cluster (no public node IPs)
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  ip_allocation_policy {}   # VPC-native cluster
}

# Standard GKE (more control than Autopilot)
resource "google_container_cluster" "standard" {
  name     = "vector-search-prod"
  location = "us-central1"

  remove_default_node_pool = true
  initial_node_count       = 1

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
}

resource "google_container_node_pool" "qdrant" {
  name       = "qdrant-nodes"
  cluster    = google_container_cluster.standard.id
  node_count = 3

  node_config {
    machine_type = "n2-highmem-16"   # 128GB RAM
    disk_size_gb = 500
    disk_type    = "pd-ssd"

    # Enable Workload Identity on nodes
    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    labels = { role = "qdrant" }
    taint {
      key    = "dedicated"
      value  = "qdrant"
      effect = "NO_SCHEDULE"
    }
  }

  autoscaling {
    min_node_count = 3
    max_node_count = 6
  }
}

# ─── Cloud Memorystore (Redis) ────────────────────────────────────────────────
resource "google_redis_instance" "cache" {
  name           = "vector-search-cache"
  tier           = "STANDARD_HA"   # high availability
  memory_size_gb = 10
  region         = "us-central1"

  redis_version     = "REDIS_7_0"
  auth_enabled      = true
  transit_encryption_mode = "SERVER_AUTHENTICATION"
}

# ─── Secret Manager ────────────────────────────────────────────────────────
resource "google_secret_manager_secret" "qdrant_api_key" {
  secret_id = "qdrant-api-key"
  replication {
    auto {}
  }
}

# Workload Identity: allow K8s SA to access secrets
resource "google_service_account_iam_member" "api_workload_identity" {
  service_account_id = google_service_account.api.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[vector-search/api-service]"
}

resource "google_secret_manager_secret_iam_member" "api_read_secret" {
  secret_id = google_secret_manager_secret.qdrant_api_key.id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.api.email}"
}
```

---

## Deploying to the Cluster

### Full Deployment Script

```bash
#!/usr/bin/env bash
# deploy.sh — deploy to EKS
set -euo pipefail

CLUSTER_NAME="${CLUSTER_NAME:-vector-search-prod}"
REGION="${AWS_REGION:-us-east-1}"
IMAGE_TAG="${IMAGE_TAG:-latest}"
NAMESPACE="vector-search"
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

echo "Deploying $IMAGE_TAG to $CLUSTER_NAME"

# Authenticate to EKS
aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$REGION"

# Apply all manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmaps/ -n "$NAMESPACE"
kubectl apply -f k8s/qdrant/ -n "$NAMESPACE"
kubectl apply -f k8s/redis/ -n "$NAMESPACE"
kubectl apply -f k8s/api/ -n "$NAMESPACE"
kubectl apply -f k8s/ingestion/ -n "$NAMESPACE"
kubectl apply -f k8s/ingress/ -n "$NAMESPACE"

# Update image tag
kubectl set image deployment/api \
  "api=${ECR_REGISTRY}/vector-search:${IMAGE_TAG}" \
  -n "$NAMESPACE"

kubectl set image deployment/ingestion-worker \
  "worker=${ECR_REGISTRY}/vector-search:${IMAGE_TAG}" \
  -n "$NAMESPACE"

# Wait for rollout
kubectl rollout status deployment/api -n "$NAMESPACE" --timeout=300s
kubectl rollout status deployment/ingestion-worker -n "$NAMESPACE" --timeout=300s

echo "Deployment complete!"
kubectl get pods -n "$NAMESPACE"
```

---

## Node Affinity for Qdrant

Qdrant needs memory-optimized nodes. Use node selectors + tolerations:

```yaml
# In Qdrant StatefulSet spec.template.spec:
nodeSelector:
  role: qdrant

tolerations:
  - key: "dedicated"
    value: "qdrant"
    effect: "NoSchedule"

# This ensures Qdrant runs ONLY on r6i.4xlarge (128GB) nodes
# Other pods can't accidentally land on those expensive nodes
```

---

## Cost Optimization

```
Strategy:
  API workers:    Spot instances (up to 70% cheaper)
                  Use multiple instance types (fallback if spot unavailable)
                  Spot interruption handler: drain and reschedule gracefully

  Qdrant nodes:   On-demand or Reserved (1-year commitment = 40% discount)
                  Spot NOT recommended for stateful workloads

  Redis:         Smaller instance + disk offload (vs. all-in-memory)

  Scaling:       Scale to 0 for non-prod environments during off-hours
                 Use KEDA (Kubernetes Event-Driven Autoscaling) for Kafka-based scaling

Estimated monthly costs (us-east-1, 2024):
  3x r6i.4xlarge (Qdrant) = $3,000/mo  (On-demand, reserved: ~$1,800)
  3-10x c5.2xlarge (API)  = $500-$1,800/mo (Spot: $150-$540)
  ElastiCache r6g.large   = $250/mo
  ECR + storage           = $50/mo
  ALB                     = $30/mo
  Total estimate:          ~$4,000-$6,000/mo
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| EKS/GKE | Managed K8s control plane; you manage worker nodes |
| IRSA / Workload Identity | K8s service accounts → IAM roles; no static credentials |
| Dedicated node groups | Qdrant on memory-optimized nodes with taints |
| ECR/GCR | Immutable image tags; scan on push |
| ElastiCache/Memorystore | Managed Redis with Multi-AZ failover |
| Secrets Manager | Centralized secrets with auto-rotation |
| Terraform | Infrastructure as Code; reproducible environments |
| Cost optimization | Spot for API, reserved for Qdrant, scale to 0 for non-prod |
