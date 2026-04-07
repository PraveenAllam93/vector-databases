# 3. GitHub Actions CI/CD — Automated Testing & Deployment

---

## ELI5

CI (Continuous Integration) = Every time you push code, robots automatically run tests and checks to catch bugs.

CD (Continuous Deployment) = If all tests pass, the robots automatically deploy to production.

Without CI/CD, deployments are manual, error-prone, and slow. With it, you can ship dozens of times a day safely.

---

## Pipeline Overview

```
Developer pushes code
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  CI Pipeline (every push)                                │
│  ├── Lint & Format check (ruff, black)                   │
│  ├── Type check (mypy)                                   │
│  ├── Unit tests (pytest)                                 │
│  ├── Integration tests (Qdrant in Docker)               │
│  ├── Security scan (trivy, trufflehog, safety)           │
│  └── Build Docker image                                  │
└──────────────────────────┬──────────────────────────────┘
                           │ (all pass)
                           ▼
┌─────────────────────────────────────────────────────────┐
│  CD Pipeline (on merge to main)                          │
│  ├── Build & push image to ECR                          │
│  ├── Deploy to staging                                   │
│  ├── Run smoke tests on staging                         │
│  ├── Manual approval gate (for production)               │
│  └── Deploy to production (rolling update)              │
└─────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
.github/
└── workflows/
    ├── ci.yml          # Runs on every push/PR
    ├── cd-staging.yml  # Deploys to staging on merge to main
    └── cd-prod.yml     # Deploys to production (manual approval)
```

---

## CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: ["*"]
  pull_request:
    branches: [main]

env:
  PYTHON_VERSION: "3.12"
  IMAGE_NAME: vector-search-api

jobs:
  # ─── Lint & Format ───────────────────────────────────────────────────────
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip

      - name: Install dev dependencies
        run: pip install ruff black mypy

      - name: Ruff (linter)
        run: ruff check src/ tests/

      - name: Black (format check)
        run: black --check src/ tests/

      - name: Mypy (type check)
        run: mypy src/ --ignore-missing-imports

  # ─── Unit Tests ──────────────────────────────────────────────────────────
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip

      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Run unit tests
        run: pytest tests/unit/ -v --cov=src --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: coverage.xml

  # ─── Integration Tests ────────────────────────────────────────────────────
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      qdrant:
        image: qdrant/qdrant:v1.11.3
        ports:
          - 6333:6333
        options: >-
          --health-cmd "curl -f http://localhost:6333/healthz"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7.4-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 3

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip

      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Run integration tests
        env:
          QDRANT_URL: http://localhost:6333
          REDIS_URL: redis://localhost:6379/0
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: pytest tests/integration/ -v --timeout=120

  # ─── Security Scanning ───────────────────────────────────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # full history for secret scanning

      - name: Scan for secrets (trufflehog)
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --only-verified

      - name: Check dependency vulnerabilities (safety)
        run: |
          pip install safety
          safety check -r requirements.txt --full-report

      - name: Scan Dockerfile (hadolint)
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile

  # ─── Build Docker Image ───────────────────────────────────────────────────
  build:
    name: Build Image
    runs-on: ubuntu-latest
    needs: [lint, unit-tests, integration-tests, security]
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build (no push for PRs)
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          target: runtime
          push: false
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha      # GitHub Actions cache
          cache-to: type=gha,mode=max

      - name: Scan built image (trivy)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          exit-code: 1
          severity: HIGH,CRITICAL
          ignore-unfixed: true
```

---

## CD to Staging

```yaml
# .github/workflows/cd-staging.yml
name: Deploy to Staging

on:
  push:
    branches: [main]   # auto-deploy to staging on merge to main

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging    # GitHub Environment with secrets
    permissions:
      id-token: write       # OIDC for AWS
      contents: read

    steps:
      - uses: actions/checkout@v4

      # Auth to AWS via OIDC (no static credentials!)
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActions-Staging
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push to ECR
        uses: docker/build-push-action@v6
        with:
          context: .
          target: runtime
          push: true
          tags: |
            ${{ steps.ecr-login.outputs.registry }}/vector-search:staging-${{ github.sha }}
            ${{ steps.ecr-login.outputs.registry }}/vector-search:staging-latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to staging cluster
        env:
          IMAGE_TAG: staging-${{ github.sha }}
          CLUSTER_NAME: vector-search-staging
        run: |
          aws eks update-kubeconfig --name $CLUSTER_NAME --region us-east-1
          
          # Update image tag
          kubectl set image deployment/api \
            api=${{ steps.ecr-login.outputs.registry }}/vector-search:$IMAGE_TAG \
            -n vector-search
          
          # Wait for rollout
          kubectl rollout status deployment/api -n vector-search --timeout=300s

      - name: Run smoke tests
        run: |
          STAGING_URL="https://api.staging.yourcompany.com"
          
          # Health check
          curl -f "$STAGING_URL/health/live" || exit 1
          
          # Basic search test
          curl -f -X POST "$STAGING_URL/search" \
            -H "Authorization: Bearer ${{ secrets.STAGING_TEST_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"text": "test query", "top_k": 5}' || exit 1
          
          echo "Smoke tests passed!"

      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          channel-id: "deployments"
          slack-message: ":red_circle: Staging deployment failed! ${{ github.run_url }}"
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## CD to Production

```yaml
# .github/workflows/cd-prod.yml
name: Deploy to Production

on:
  workflow_dispatch:    # manual trigger
    inputs:
      image_tag:
        description: "Image tag to deploy (e.g., staging-abc1234)"
        required: true
      confirm:
        description: "Type DEPLOY to confirm production deployment"
        required: true

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Validate confirmation
        if: ${{ github.event.inputs.confirm != 'DEPLOY' }}
        run: |
          echo "Confirmation failed. Type DEPLOY to confirm."
          exit 1

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: validate
    environment: production     # requires approval from environment reviewers
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActions-Prod
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Re-tag staging image as prod
        run: |
          REGISTRY=${{ steps.ecr-login.outputs.registry }}
          SOURCE_TAG=${{ github.event.inputs.image_tag }}
          PROD_TAG=prod-${{ github.sha }}
          
          # Pull staging image, re-tag as prod (no new build)
          docker pull $REGISTRY/vector-search:$SOURCE_TAG
          docker tag $REGISTRY/vector-search:$SOURCE_TAG $REGISTRY/vector-search:$PROD_TAG
          docker push $REGISTRY/vector-search:$PROD_TAG

      - name: Deploy to production (canary first)
        env:
          IMAGE_TAG: prod-${{ github.sha }}
          CLUSTER_NAME: vector-search-prod
        run: |
          aws eks update-kubeconfig --name $CLUSTER_NAME --region us-east-1
          
          # Step 1: Deploy to 1 of 10 pods (10% canary)
          kubectl patch deployment api \
            -n vector-search \
            --type='json' \
            -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"'$REGISTRY/vector-search:$IMAGE_TAG'"}]'
          
          echo "Canary pod deployed. Monitoring for 5 minutes..."
          sleep 300
          
          # Step 2: Check error rate (query Prometheus)
          ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query" \
            --data-urlencode 'query=rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])' \
            | jq -r '.data.result[0].value[1]')
          
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high: $ERROR_RATE. Rolling back!"
            kubectl rollout undo deployment/api -n vector-search
            exit 1
          fi
          
          # Step 3: Full rollout
          kubectl rollout status deployment/api -n vector-search --timeout=600s
          
          echo "Production deployment successful!"

      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: "prod-${{ github.sha }}"
          release_name: "Production ${{ github.sha }}"
          body: "Deployed ${{ github.event.inputs.image_tag }} to production"

      - name: Notify success
        uses: slackapi/slack-github-action@v1.27.0
        with:
          channel-id: "deployments"
          slack-message: ":white_check_mark: Production deployment successful! ${{ github.sha }}"
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## Dependency Updates (Automated)

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: pip
    directory: "/"
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
    labels: ["dependencies", "python"]

  - package-ecosystem: docker
    directory: "/"
    schedule:
      interval: weekly
    labels: ["dependencies", "docker"]

  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: monthly
    labels: ["dependencies", "github-actions"]
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| CI on every push | Lint → type check → unit tests → integration tests → security scan → build |
| OIDC auth | No stored AWS keys; GitHub assumes IAM role via OIDC |
| Staging auto-deploy | Every merge to main → staging; fast feedback loop |
| Production manual | `workflow_dispatch` + environment approval gate |
| Canary deployment | Deploy to 1 pod, monitor error rate, then full rollout |
| Smoke tests | Basic health + search test after every staging deployment |
| Dependabot | Automated dependency update PRs weekly |
| Trivy in CI | Block deploys with HIGH/CRITICAL CVEs |
