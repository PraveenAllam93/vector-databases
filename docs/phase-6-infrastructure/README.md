# Phase 6: Infrastructure & CI/CD

## Reading Order

| # | Topic | File | Key Takeaway |
|---|-------|------|-------------|
| 1 | [Docker Production](01-docker-production.md) | Multi-stage builds (180MB runtime), non-root user, .dockerignore, layer caching, HEALTHCHECK, trivy scanning |
| 2 | [Kubernetes Deployment](02-kubernetes-deployment.md) | StatefulSet for Qdrant, Deployment for API, anti-affinity, HPA, PDB, Ingress with cert-manager |
| 3 | [GitHub Actions CI/CD](03-github-actions-cicd.md) | Lint → tests → security scan → build; OIDC to AWS; staging auto-deploy; prod canary + manual approval |
| 4 | [Cloud Deployment](04-cloud-deployment.md) | EKS+Terraform (VPC, node groups, IRSA, ECR, ElastiCache); GKE Autopilot alternative; cost optimization |

---

## Interview Quick Reference

**"Walk me through your deployment pipeline."**
→ Push triggers CI: lint (ruff/black) → type check (mypy) → unit tests → integration tests (Qdrant in Docker service) → secret scan (trufflehog) → build multi-stage Docker image → trivy CVE scan (block on HIGH/CRITICAL). On merge to main: build and push to ECR with SHA tag, deploy to staging cluster (`kubectl set image`), smoke tests. Production: manual `workflow_dispatch` with approval gate → canary to 1 pod → monitor error rate for 5min → full rollout.

**"How do you achieve zero-downtime deployments?"**
→ RollingUpdate strategy with `maxUnavailable: 0` and `maxSurge: 1` — K8s adds 1 new pod and only removes an old pod after the new one passes its readiness probe. `terminationGracePeriodSeconds: 60` allows in-flight requests to complete. PodDisruptionBudget keeps `minAvailable: 2` during node maintenance. Readiness probe gates traffic — pods only receive requests when genuinely ready.

**"How do you handle infrastructure as code?"**
→ Terraform for all cloud resources: VPC, EKS cluster, node groups, ECR, ElastiCache, Secrets Manager, IAM roles. State in S3 with DynamoDB locking. Separate workspaces for staging/prod. K8s manifests applied via `kubectl apply -f k8s/`. Secrets managed by External Secrets Operator syncing from AWS Secrets Manager — not committed to git.

**"How do you scale Qdrant vs API workers differently?"**
→ Qdrant on StatefulSet (3 dedicated r6i.4xlarge nodes with taints, no autoscaling — stateful, needs manual capacity planning). API on Deployment with HPA (scale 3→20 on CPU > 70%). Ingestion workers on Deployment with HPA on Kafka consumer lag (scale 2→10 when lag > 1000 messages). They scale independently, node groups have separate instance types and costs.

**"What happens when a Docker image has a critical vulnerability?"**
→ trivy runs in CI with `--exit-code 1 --severity HIGH,CRITICAL` — the build fails and the image is never pushed or deployed. ECR also scans on push. Dependabot sends automated PRs weekly for Python dependency updates. Base image is pinned (python:3.12-slim, not latest) so we control when to update.

**"How do you secure AWS credentials in GitHub Actions?"**
→ OIDC: no stored AWS credentials. GitHub Action assumes an IAM role via `aws-actions/configure-aws-credentials` with OIDC token. IAM role has a trust policy restricting it to the specific GitHub repo and branch. Least privilege: staging role can only push to ECR and update EKS staging; prod role is separate with additional approval requirements.

---

## Infrastructure Checklist

```
Docker:
  □ Multi-stage build (runtime image < 200MB)
  □ Non-root user (UID 1000)
  □ .dockerignore excludes .env, .git, tests
  □ HEALTHCHECK defined
  □ Dependencies pinned in requirements.txt
  □ trivy scan in CI (block on HIGH/CRITICAL)

Kubernetes:
  □ Namespace created with labels
  □ Qdrant on StatefulSet with PVC (encrypted-gp3)
  □ Anti-affinity: no 2 Qdrant pods on same node
  □ TopologySpread: pods spread across AZs
  □ PDB: minAvailable=2 for Qdrant and API
  □ HPA configured for API and ingestion workers
  □ RollingUpdate maxUnavailable=0
  □ readOnlyRootFilesystem + emptyDir for /tmp
  □ Resource requests AND limits set
  □ Liveness + readiness probes on all containers
  □ terminationGracePeriodSeconds >= 30

CI/CD:
  □ Tests run on every push (not just main)
  □ Integration tests use real Qdrant (Docker service)
  □ Secret scanning on every PR
  □ OIDC auth to cloud (no static credentials)
  □ Image tag = git SHA (immutable, traceable)
  □ Smoke tests after staging deploy
  □ Manual approval gate for production
  □ Slack notifications on failure

Cloud:
  □ Private subnets for all data services
  □ Security groups: Qdrant only from app services
  □ IRSA/Workload Identity (K8s SA → IAM role)
  □ ECR with immutable tags + scan on push
  □ ElastiCache Multi-AZ with encryption
  □ Secrets in Secrets Manager (not env vars)
  □ Terraform state in S3 with locking
  □ Dedicated node group for Qdrant (memory-optimized)
```
