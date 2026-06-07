# CI/CD Fundamentals

## Introduction
Continuous Integration and Continuous Delivery/Deployment (CI/CD) are the foundational practices of modern software delivery. They transform software releases from high-risk, infrequent events (quarterly releases, weekend deployment marathons) into low-risk, daily operations that any developer can perform. The practices emerged from Extreme Programming (Kent Beck, 1999) and were later codified by Jez Humble and Dave Farley in "Continuous Delivery" (2010).

CI/CD is not a tool (though Jenkins, GitHub Actions, GitLab CI, and ArgoCD are common choices). It's a set of principles: integrate code frequently (CI), ensure every change is releasable (CD), and make the release process a non-event. For solution architects, CI/CD is not just a developer concern—it's the delivery mechanism that connects code to customers, and its design directly impacts deployment frequency, change failure rate, and mean time to recovery (the DORA metrics).

## Definition

**Continuous Integration (CI)** is the practice of automatically building and testing every code change merged to the main branch, providing rapid feedback to developers.

**Continuous Delivery (CD)** extends CI by automatically deploying every passing change to a staging environment and ensuring it is releasable to production with the push of a button.

**Continuous Deployment** goes further: every change that passes all tests is automatically deployed to production without human intervention.

## Concept Explanation

### The CI/CD Pipeline

```
┌───────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE                      │
├───────────────────────────────────────────────────────┤
│                                                        │
│  [Source] → [Build] → [Test] → [Package] → [Deploy]  │
│     │         │         │          │           │       │
│     ▼         ▼         ▼          ▼           ▼       │
│   Git      Compile   Unit       Docker     Staging    │
│   Push     Lint      Integ      Push       Canary     │
│            SAST      E2E        Helm       Prod       │
│                      Security   Chart                  │
│                      Perf                              │
│                                                        │
└───────────────────────────────────────────────────────┘
```

### Pipeline as Code

```yaml
# GitHub Actions CI/CD pipeline
name: Build, Test, Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: order-service

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt
      
      - name: Lint
        run: |
          ruff check .
          mypy .
      
      - name: Security scan (SAST)
        run: bandit -r src/
      
      - name: Unit tests
        run: pytest tests/unit/ --junitxml=test-results.xml
      
      - name: Integration tests
        run: docker-compose -f docker-compose.test.yml up --abort-on-container-exit
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results.xml

  build-and-push:
    needs: lint-and-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.ECR_REPOSITORY }}:${{ github.sha }}
            ${{ env.ECR_REPOSITORY }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          helm upgrade --install order-service ./helm/order-service \
            --namespace staging \
            --set image.tag=${{ github.sha }} \
            --set environment=staging \
            --wait

  integration-tests-staging:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Smoke tests
        run: |
          curl -f https://staging-api.example.com/health
          curl -f https://staging-api.example.com/api/orders

  deploy-production:
    needs: integration-tests-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to production
        run: |
          helm upgrade --install order-service ./helm/order-service \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --set environment=production \
            --wait
      
      - name: Verify deployment
        run: |
          kubectl rollout status deployment/order-service -n production
          curl -f https://api.example.com/health
```

### Build Strategies

```dockerfile
# MULTI-STAGE BUILD: Separate build from runtime
# Stage 1: Build
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

# Stage 2: Runtime (minimal)
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Testing Pyramid

```
        ┌─────┐
        │ E2E │  (few tests, slow, validate critical user journeys)
       ┌┴─────┴┐
       │Integration│  (medium, verify service interactions work)
      ┌┴──────────┴┐
      │   Unit Tests  │  (many tests, fast, verify individual functions)
     └────────────────┘

Unit tests:     70% of tests, run in < 10 seconds
Integration:    20% of tests, run in < 2 minutes
E2E:            10% of tests, run in < 10 minutes
```

```python
# Unit test (fast, isolated)
def test_calculate_order_total():
    order = Order(items=[
        OrderItem(sku="SKU-1", quantity=2, unit_price=29.99),
        OrderItem(sku="SKU-2", quantity=1, unit_price=49.99)
    ])
    assert order.calculate_total() == 109.97

# Integration test (verifies dependencies work together)
@pytest.mark.integration
def test_create_order_endpoint(client, test_db):
    response = client.post('/api/orders', json={
        'customer_id': 'CUST-001',
        'items': [{'sku': 'SKU-1', 'quantity': 1, 'unit_price': 29.99}]
    })
    assert response.status_code == 201
    assert test_db.query(Order).count() == 1

# E2E test (verifies entire user journey)
@pytest.mark.e2e
def test_order_flow():
    # Login → Search → Add to Cart → Checkout → Verify Order
    login("test_user@example.com", "password")
    search_results = search_product("laptop")
    add_to_cart(search_results[0]['id'])
    cart = get_cart()
    checkout(cart['items'])
    orders = get_orders()
    assert len(orders) == 1
    assert orders[0]['status'] == 'CONFIRMED'
```

### Environment Strategy

```yaml
environments:
  dev:
    purpose: Developer testing, rapid iteration
    deploy: Automatic on every push to feature branch
    data: Synthetic, anonymized
    scale: Minimal (1 replica)
    monitoring: Basic health checks

  staging:
    purpose: Pre-production validation, integration testing
    deploy: Automatic on merge to main
    data: Anonymized production snapshot (refreshed weekly)
    scale: Production-like (3+ replicas)
    monitoring: Full (metrics, logs, traces, alerts)

  production:
    purpose: Live customer traffic
    deploy: Manual approval or automated with canary
    data: Real customer data
    scale: Auto-scaled based on load
    monitoring: Full + PagerDuty alerts
```

### DORA Metrics

```python
class DORAMetrics:
    """
    Four key metrics that measure software delivery performance.
    """
    
    def deployment_frequency(self):
        """How often code is deployed to production"""
        # Elite: Multiple times per day
        # High: Once per day to once per week
        # Medium: Once per week to once per month
        # Low: Once per month to once per 6 months
        return self.deployments_last_30_days / 30
    
    def lead_time_for_changes(self):
        """Time from code commit to production deployment"""
        # Elite: < 1 hour
        # High: 1 day to 1 week
        # Medium: 1 week to 1 month
        # Low: > 1 month
        return self.avg(self.deploy_time - self.commit_time)
    
    def change_failure_rate(self):
        """Percentage of deployments causing failure"""
        # Elite: 0-15%
        # High: 15-30%
        # Medium: 30-45%
        # Low: > 45%
        return self.failed_deployments / self.total_deployments * 100
    
    def mean_time_to_recovery(self):
        """Time to recover from a failed deployment"""
        # Elite: < 1 hour
        # High: < 1 day
        # Medium: 1 day to 1 week
        # Low: > 1 week
        return self.avg(self.recovery_time - self.failure_time)
```

## Layman's Explanation

### The Assembly Line
Before assembly lines (before CI/CD): A craftsperson built an entire car from start to finish. If something went wrong, they discovered it at the very end. Cars took weeks to build. Defects were found when the customer tried to start the engine.

After assembly lines (CI/CD): The car moves through stations. At each station, quality is checked immediately. If a bolt is missing at station 3, the line stops and it's fixed immediately—not discovered when the customer drives off the lot. Cars are built in hours, and every car is tested before it leaves the factory.

- **CI (Continuous Integration)**: Every worker adds their part to the same car throughout the day. If two parts don't fit together, we know within minutes, not weeks.
- **CD (Continuous Delivery)**: Every completed car is parked in the showroom, fully inspected, ready to sell. The salesperson can hand over the keys anytime.
- **Continuous Deployment**: The car is automatically delivered to the customer's driveway. No human says "ship it"—the system ships automatically when all checks pass.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Pipeline as Code vs UI-Driven CI/CD**: Pipeline as Code (GitHub Actions, GitLab CI) means the pipeline definition lives in the repository alongside the code. This enables version-controlled, reviewed, and reproducible pipelines. UI-driven pipelines (classic Jenkins jobs) create configuration drift and are unreproducible in disasters.
- **Environment Promotion Strategy**: Same image (by SHA digest, not tag) must flow through dev → staging → production. Rebuilding at each stage introduces inconsistency—the staging image is not the production image. Build once, deploy everywhere.
- **Secrets in CI/CD**: Never bake secrets into container images. Inject them at deploy time via secret management (Vault, Secrets Manager, Sealed Secrets). CI systems need access to secrets (deploy credentials, signing keys); use OIDC federation (GitHub Actions → AWS IAM, GitLab → GCP, no long-lived credentials).
- **Deployment Strategy**: Rolling update (replace one-by-one), Blue/Green (switch all traffic at once), Canary (gradually increase traffic to new version). The choice depends on the cost of failure and the tolerance for brief inconsistencies during deployment.

### Business Impact
- **DORA Metrics → Business Performance**: Elite performers (multiple deploys/day, < 1 hour lead time, < 15% failure rate, < 1 hour MTTR) are 2x more likely to meet or exceed organizational performance goals (profitability, productivity, market share). CI/CD is not just a technical practice—it's a business differentiator.
- **Risk Reduction**: Small, frequent deployments are safer than large, infrequent ones. A deployment changing 10 lines is far easier to debug and rollback than one changing 10,000 lines. CI/CD enables small batch sizes.

## On-Premises Examples

### Jenkins (Traditional CI/CD)
```groovy
// Jenkinsfile (Pipeline as Code)
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps { git 'https://github.com/org/order-service.git' }
        }
        stage('Build') {
            steps { sh 'docker build -t order-service .' }
        }
        stage('Test') {
            steps { sh 'docker run order-service pytest' }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps { sh 'helm upgrade --install order-service ./chart' }
        }
    }
}
```

### GitLab CI
```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

test:
  stage: test
  script:
    - pip install -r requirements.txt
    - pytest

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

deploy:
  stage: deploy
  script:
    - helm upgrade order-service ./chart --set image.tag=$CI_COMMIT_SHA
  environment: production
  when: manual  # Manual approval for production
```

## AWS Examples

### CodePipeline + CodeBuild + CodeDeploy
```hcl
resource "aws_codepipeline" "order_service" {
  name = "order-service-pipeline"

  artifact_store {
    location = aws_s3_bucket.artifacts.bucket
    type     = "S3"
  }

  stage {
    name = "Source"
    action {
      name     = "Source"
      category = "Source"
      provider = "GitHub"
      owner    = "ThirdParty"
      configuration = {
        Owner  = "org"
        Repo   = "order-service"
        Branch = "main"
      }
    }
  }

  stage {
    name = "Build"
    action {
      name     = "Build"
      category = "Build"
      provider = "CodeBuild"
      configuration = {
        ProjectName = aws_codebuild_project.order_service.name
      }
    }
  }

  stage {
    name = "Deploy"
    action {
      name     = "Deploy"
      category = "Deploy"
      provider = "CodeDeployToECS"
      configuration = {
        ApplicationName = aws_codedeploy_app.order_service.name
        DeploymentGroupName = aws_codedeploy_deployment_group.order_service.name
      }
    }
  }
}
```

## GCP Examples

### Cloud Build + Cloud Deploy
```yaml
# cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/order-service:$COMMIT_SHA', '.']
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/order-service:$COMMIT_SHA']
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  args: ['gcloud', 'deploy', 'releases', 'create', 'rel-$COMMIT_SHA',
         '--delivery-pipeline=order-service-pipeline',
         '--region=us-central1',
         '--images=order-service=gcr.io/$PROJECT_ID/order-service:$COMMIT_SHA']
```

## Summary

| CI/CD Capability | Tool Options |
|-----------------|--------------|
| Source Control | GitHub, GitLab, Bitbucket |
| CI Pipeline | GitHub Actions, GitLab CI, Jenkins, CircleCI |
| CD Pipeline | ArgoCD, Flux, Spinnaker, CodeDeploy |
| Container Registry | ECR, GCR/Artifact Registry, ACR, Docker Hub |
| Secrets in CI/CD | OIDC federation, Vault, Secrets Manager |

CI/CD is the delivery system that transforms code into customer value. A well-designed CI/CD pipeline provides fast feedback (minutes, not hours), reliable deployments (automated, repeatable, rollbackable), and safety (every change tested, every deployment verified). The architect's role is to design the pipeline architecture—what happens at each stage, how environments are structured, how secrets are managed—and to ensure the pipeline itself is resilient, observable, and auditable.
