# Chapter 17: CI/CD with Helm

Integrating Helm into CI/CD pipelines transforms manual deployment into repeatable, auditable, and version-controlled releases. This chapter covers every stage of the Helm CI/CD lifecycle across all major platforms.

---

## 17.1 Why Helm in CI/CD

Helm in CI/CD solves fundamental deployment challenges:

| Without Helm | With Helm in CI/CD |
|---|---|
| Manual `kubectl apply` via ad-hoc YAML | Templatized, parameterized deployments |
| Config values scattered across scripts | Single values file per environment |
| No rollback capability (re-apply old manifests) | `helm rollback` to any revision in seconds |
| "Works on my machine" drift | Identical Helm CLI produces identical manifests |
| Manual environment promotion | Automated promotion of chart+values through environments |
| No deployment history | Full release history stored as Secrets |
| Credentials in shell scripts | External secrets integration (ESO, SOPS, Vault) |

---

## 17.2 Core CI/CD Principles

Every Helm CI/CD pipeline follows the same fundamental sequence:

```
 ┌──────┐   ┌────────────┐   ┌──────────┐   ┌─────────┐   ┌──────────┐   ┌────────┐   ┌──────────┐
 │ LINT │→│ TEMPLATE   │→│ PACKAGE  │→│ PUSH    │→│ DEPLOY   │→│ TEST   │→│ VERIFY   │
 │      │   │ (dry-run)  │   │          │   │ (to OCI) │   │          │   │        │   │          │
 └──────┘   └────────────┘   └──────────┘   └─────────┘   └──────────┘   └────────┘   └──────────┘
     ✓            ✓               ✓             ✓              ✓             ✓             ✓
  Chart.yaml   Rendered      .tgz file     OCI registry    Kubernetes   helm test     Smoke tests
  syntax       manifests      created       populated       running      pods run      pass + logs
  check        are valid                                                  and exit      verified
                                                                          cleanly
```

**Parallel gates that run concurrently with the above:**
- Secret decryption (helm-secrets, SOPS)
- Vulnerability scanning (Trivy, Grype)
- Signature verification (helm verify, cosign)
- Policy enforcement (OPA, Kyverno)

---

## 17.3 `helm upgrade --install` — The CI/CD Workhorse

This single command is the most important Helm command in CI/CD. It is idempotent — it installs on first run and upgrades on subsequent runs.

```bash
helm upgrade --install myapp ./mychart \
  --namespace production \
  --create-namespace \
  --values values-prod.yaml \
  --atomic \
  --timeout 5m \
  --wait \
  --wait-for-jobs
```

| Flag | Purpose in CI/CD |
|---|---|
| `--install` | Idempotent: creates release if absent, upgrades if present |
| `--atomic` | Automatically rolls back on failure (do not deploy a broken release) |
| `--cleanup-on-fail` | Delete newly created resources on failed install |
| `--wait` | Block until all Pods, PVCs, Services, and Deployments are ready |
| `--wait-for-jobs` | Also wait for Jobs to complete before returning |
| `--timeout` | Maximum time to wait (fail fast in CI/CD; default 5m is often too short) |
| `--force` | Delete and recreate resources that cannot be updated (e.g., immutable fields) |
| `--recreate-pods` | Restart all Pods on upgrade (useful when image tag does not change) |
| `--dry-run` | Render manifests and validate without applying (CI lint gate) |
| `--dependency-update` | Run `helm dependency update` before installing |

**Warning:** `--force` deletes and recreates resources. This causes downtime. Use only when upgrading resources with immutable fields that prevent in-place updates.

### 17.3.1 Atomic Deployments

```bash
helm upgrade --install myapp ./mychart --atomic --cleanup-on-fail --timeout 10m
```

With `--atomic`:
1. Helm runs `helm install` or `helm upgrade`
2. If the release succeeds → Helm exits 0
3. If the release fails (pods crash, timeout exceeded) → **Helm automatically rolls back to the last successful revision**
4. With `--cleanup-on-fail`, newly created resources are also deleted on initial install failure

This is critical in CI/CD — you never want a broken release sitting in production.

---

## 17.4 Wait Strategies

### 17.4.1 `--wait` Behavior

`--wait` polls Kubernetes until all resources report ready:
- **Deployments/StatefulSets/DaemonSets:** All replicas are `Available`
- **Pods:** Status is `Ready` and containers are running
- **PVCs:** Status is `Bound`
- **Services:** At least one endpoint is ready (for LoadBalancer Services, this waits for external IP)

### 17.4.2 `--wait-for-jobs`

Without `--wait-for-jobs`, Helm returns as soon as Deployments are ready — before Job Pods complete. With it:

```bash
helm upgrade --install myapp ./mychart --wait --wait-for-jobs --timeout 15m
```

This is essential when deploying:
- Database migrations as Helm hooks (`helm.sh/hook: post-install,post-upgrade`)
- Data seeding Jobs
- Integration test Jobs that must complete before the pipeline advances

### 17.4.3 Readiness Probes in CI/CD Context

Without readiness probes, `--wait` considers a Pod ready as soon as its containers start — even if the application is still initializing. Always define readiness probes:

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

If probes are missing and `--wait` returns prematurely, the next pipeline stage (integration tests) may fail against an application that is not yet serving traffic.

---

## 17.5 Environment Management

### 17.5.1 Per-Environment Values Files

```
myapp/
├── Chart.yaml
├── templates/
├── values.yaml              # Common defaults
├── values-dev.yaml          # Development overrides
├── values-staging.yaml      # Staging overrides
├── values-prod.yaml         # Production overrides
├── secrets-dev.yaml         # Encrypted secrets (SOPS)
├── secrets-staging.yaml
└── secrets-prod.yaml
```

```bash
# Development
helm upgrade --install myapp ./mychart -f values.yaml -f values-dev.yaml -f secrets-dev.yaml

# Staging
helm upgrade --install myapp ./mychart -f values.yaml -f values-staging.yaml -f secrets-staging.yaml

# Production
helm upgrade --install myapp ./mychart -f values.yaml -f values-prod.yaml -f secrets-prod.yaml
```

Values merge in order — later files override earlier ones. `values.yaml` provides safe defaults; environment files override only what differs.

### 17.5.2 Environment Promotion

```
dev → staging → production

Chart: myapp-1.2.3 (same .tgz artifact)
Values: values-dev.yaml → values-staging.yaml → values-prod.yaml
```

The chart artifact is immutable — the same `myapp-1.2.3.tgz` is promoted through environments. Only the values file changes. This guarantees that code tested in staging is identical to what runs in production.

---

## 17.6 GitHub Actions — Complete Workflow

```yaml
name: Helm CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CHART_PATH: ./charts/myapp
  OCI_REGISTRY: registry.example.com/charts
  KUBE_NAMESPACE_STAGING: staging
  KUBE_NAMESPACE_PROD: production

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.15.0

      - name: Helm Lint
        run: helm lint ${{ env.CHART_PATH }}

      - name: Helm Template (dry-run)
        run: |
          helm template myapp ${{ env.CHART_PATH }} \
            -f ${{ env.CHART_PATH }}/values.yaml > /dev/null

      - name: Scan Chart for Vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: ${{ env.CHART_PATH }}
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Helm Dependency Update
        run: helm dependency update ${{ env.CHART_PATH }}

      - name: Package Chart
        run: helm package ${{ env.CHART_PATH }} --destination /tmp/packages

      - name: Verify Package (if signed)
        run: helm verify /tmp/packages/myapp-*.tgz || true

      - name: Upload Chart Artifact
        uses: actions/upload-artifact@v4
        with:
          name: helm-chart
          path: /tmp/packages/myapp-*.tgz

  deploy-staging:
    needs: lint-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Setup Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.15.0

      - name: Download Chart Artifact
        uses: actions/download-artifact@v4
        with:
          name: helm-chart
          path: /tmp/packages

      - name: Setup kubeconfig
        uses: azure/k8s-set-context@v4
        with:
          kubeconfig: ${{ secrets.KUBECONFIG_STAGING }}

      - name: Decrypt Secrets (SOPS)
        run: |
          helm plugin install https://github.com/jkroepke/helm-secrets
          echo "${{ secrets.SOPS_AGE_KEY }}" > /tmp/age-key.txt
          export SOPS_AGE_KEY_FILE=/tmp/age-key.txt

      - name: Deploy to Staging
        run: |
          CHART_FILE=$(ls /tmp/packages/myapp-*.tgz | head -1)
          helm upgrade --install myapp "$CHART_FILE" \
            --namespace ${{ env.KUBE_NAMESPACE_STAGING }} \
            --create-namespace \
            -f values-staging.yaml \
            --atomic \
            --wait \
            --wait-for-jobs \
            --timeout 10m
        env:
          SOPS_AGE_KEY: ${{ secrets.SOPS_AGE_KEY }}

      - name: Run Smoke Tests
        run: |
          helm test myapp --namespace ${{ env.KUBE_NAMESPACE_STAGING }} --timeout 5m

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Setup Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.15.0

      - name: Download Chart Artifact
        uses: actions/download-artifact@v4
        with:
          name: helm-chart
          path: /tmp/packages

      - name: Setup kubeconfig
        uses: azure/k8s-set-context@v4
        with:
          kubeconfig: ${{ secrets.KUBECONFIG_PRODUCTION }}

      - name: Deploy to Production
        run: |
          CHART_FILE=$(ls /tmp/packages/myapp-*.tgz | head -1)
          helm upgrade --install myapp "$CHART_FILE" \
            --namespace ${{ env.KUBE_NAMESPACE_PROD }} \
            --create-namespace \
            -f values-prod.yaml \
            --atomic \
            --wait \
            --timeout 15m

  rollback:
    needs: deploy-production
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Rollback Production
        uses: azure/k8s-set-context@v4
        with:
          kubeconfig: ${{ secrets.KUBECONFIG_PRODUCTION }}
      - run: |
          helm rollback myapp --namespace ${{ env.KUBE_NAMESPACE_PROD }} --wait
```

---

## 17.7 GitLab CI — Complete Pipeline

```yaml
stages:
  - lint
  - test
  - build
  - deploy-staging
  - verify-staging
  - deploy-prod

variables:
  CHART_PATH: charts/myapp
  OCI_REGISTRY: ${CI_REGISTRY}/helm

.lint_template: &lint_steps
  - helm lint ${CHART_PATH}
  - helm template myapp ${CHART_PATH} -f values.yaml --debug > /dev/null
  - helm dependency update ${CHART_PATH}

lint:
  stage: lint
  image: alpine/helm:3.15.0
  script: *lint_steps

test:
  stage: test
  image: alpine/helm:3.15.0
  script:
    - helm plugin install https://github.com/helm-unittest/helm-unittest
    - helm unittest ${CHART_PATH}
  allow_failure: false

build:
  stage: build
  image: alpine/helm:3.15.0
  script:
    - helm registry login ${OCI_REGISTRY}
      --username ${CI_REGISTRY_USER}
      --password ${CI_REGISTRY_PASSWORD}
    - helm dependency update ${CHART_PATH}
    - helm package ${CHART_PATH}
    - CHART_FILE=$(ls myapp-*.tgz)
    - helm push ${CHART_FILE} oci://${OCI_REGISTRY}
  artifacts:
    paths:
      - myapp-*.tgz
  only:
    - main

deploy-staging:
  stage: deploy-staging
  image: alpine/helm:3.15.0
  environment:
    name: staging
  before_script:
    - kubectl config set-cluster staging --server=${STAGING_K8S_URL}
      --certificate-authority=${KUBE_CA_PEM}
    - kubectl config set-credentials gitlab --token=${STAGING_K8S_TOKEN}
    - kubectl config set-context staging --cluster=staging --user=gitlab
    - kubectl config use-context staging
  script:
    - helm upgrade --install myapp ${CHART_PATH}
      --namespace staging
      --create-namespace
      -f values-staging.yaml
      --atomic
      --wait
      --wait-for-jobs
      --timeout 10m
  only:
    - main

verify-staging:
  stage: verify-staging
  image: alpine/helm:3.15.0
  environment:
    name: staging
  script:
    - helm test myapp --namespace staging --timeout 5m
  needs: [deploy-staging]
  only:
    - main

deploy-prod:
  stage: deploy-prod
  image: alpine/helm:3.15.0
  environment:
    name: production
  before_script:
    - kubectl config set-cluster prod --server=${PROD_K8S_URL}
      --certificate-authority=${KUBE_CA_PEM}
    - kubectl config set-credentials gitlab --token=${PROD_K8S_TOKEN}
    - kubectl config set-context prod --cluster=prod --user=gitlab
    - kubectl config use-context prod
  script:
    - helm upgrade --install myapp ${CHART_PATH}
      --namespace production
      --create-namespace
      -f values-prod.yaml
      --atomic
      --wait
      --timeout 15m
      --description "GitLab CI pipeline ${CI_PIPELINE_ID}"
  only:
    - main
  when: manual
```

---

## 17.8 Jenkins — Jenkinsfile Example

```groovy
pipeline {
    agent any

    environment {
        CHART_PATH = 'charts/myapp'
        OCI_REGISTRY = 'registry.example.com/charts'
    }

    stages {
        stage('Lint & Validate') {
            steps {
                sh 'helm lint ${CHART_PATH}'
                sh 'helm template myapp ${CHART_PATH} -f values.yaml > /dev/null'
            }
        }

        stage('Unit Test') {
            steps {
                sh '''
                    helm plugin install https://github.com/helm-unittest/helm-unittest || true
                    helm unittest ${CHART_PATH}
                '''
            }
        }

        stage('Package & Push') {
            when { branch 'main' }
            steps {
                withCredentials([string(credentialsId: 'oci-password', variable: 'OCI_PASS')]) {
                    sh '''
                        helm registry login ${OCI_REGISTRY} --username ci-bot --password ${OCI_PASS}
                        helm dependency update ${CHART_PATH}
                        helm package ${CHART_PATH}
                        CHART_FILE=$(ls myapp-*.tgz)
                        helm push ${CHART_FILE} oci://${OCI_REGISTRY}
                    '''
                }
            }
        }

        stage('Deploy Staging') {
            when { branch 'main' }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-staging']) {
                    sh 'helm upgrade --install myapp ${CHART_PATH} -n staging --create-namespace -f values-staging.yaml --atomic --wait --timeout 10m'
                }
            }
        }

        stage('Smoke Test Staging') {
            when { branch 'main' }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-staging']) {
                    sh 'helm test myapp --namespace staging --timeout 5m'
                }
            }
        }

        stage('Deploy Production') {
            when { branch 'main' }
            input { message 'Deploy to production?' }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                    sh 'helm upgrade --install myapp ${CHART_PATH} -n production --create-namespace -f values-prod.yaml --atomic --wait --timeout 15m'
                }
            }
        }
    }

    post {
        failure {
            script {
                if (env.BRANCH_NAME == 'main') {
                    withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                        sh 'helm rollback myapp --namespace production --wait || true'
                    }
                }
            }
        }
    }
}
```

---

## 17.9 ArgoCD — Declarative Helm Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/helm-charts
    targetRevision: main
    path: charts/myapp
    helm:
      valueFiles:
        - values-prod.yaml
      parameters:
        - name: image.tag
          value: "v2.1.3"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Auto-delete resources removed from chart
      selfHeal: true     # Auto-revert manual kubectl changes
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m
```

**ArgoCD + Helm patterns:**
- Store values files in the same Git repo as the chart (monorepo) or in a separate config repo
- Use `helm.parameters` in the Application manifest for minor overrides (image tags, replica counts)
- Set `selfHeal: true` to prevent drift — if someone manually changes a Helm-managed resource, ArgoCD reverts it
- Multi-source apps allow combining Helm charts with raw Kubernetes manifests in a single Application

---

## 17.10 FluxCD — HelmRelease Resource

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: my-repo
  namespace: flux-system
spec:
  interval: 5m
  url: oci://registry.example.com/charts
  type: oci
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: myapp
      version: ">=1.0.0 <2.0.0"
      sourceRef:
        kind: HelmRepository
        name: my-repo
  values:
    replicaCount: 5
    image:
      tag: v2.1.3
  valuesFrom:
    - kind: ConfigMap
      name: myapp-config
      valuesKey: overrides.yaml
    - kind: Secret
      name: myapp-secrets
      valuesKey: secrets.yaml
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
  driftDetection:
    mode: enabled
  timeout: 10m
```

**Production Note:** Flux's drift detection re-applies the Helm release automatically when it detects changes to the cluster that diverge from the desired state defined in Git. Combined with `remediateLastFailure`, Flux auto-rolls back on failed upgrades.

---

## 17.11 Canary Deployments with Helm

### 17.11.1 Helm + Service Mesh (Istio)

Deploy a canary as a separate Helm release with reduced traffic weight:

```bash
# Stable release (100% traffic)
helm upgrade --install myapp-stable ./mychart -f values-stable.yaml --namespace prod

# Canary release (10% traffic)
helm upgrade --install myapp-canary ./mychart -f values-canary.yaml --namespace prod
```

Istio `VirtualService` splits traffic:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp.prod.svc.cluster.local
  http:
    - route:
        - destination:
            host: myapp-stable
          weight: 90
        - destination:
            host: myapp-canary
          weight: 10
```

### 17.11.2 Helm + Argo Rollouts

Argo Rollouts replaces Deployments with `Rollout` resources that support canary and blue/green strategies natively:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
  template:
    # ... pod spec (generated by Helm template)
```

Helm deploys the Rollout; Argo Rollouts manages the canary progression. Helm's `--wait` waits for the initial rollout; the canary progression is managed by the Rollout controller.

---

## 17.12 Blue/Green Deployments with Helm

```bash
# Deploy blue (current production)
helm upgrade --install myapp-blue ./mychart \
  -f values-prod.yaml --set color=blue --namespace prod

# Deploy green (new version)
helm upgrade --install myapp-green ./mychart \
  -f values-prod.yaml --set color=green --set image.tag=v3.0.0 --namespace prod

# Wait for green to be fully ready
helm test myapp-green --namespace prod

# Switch the Service to green
kubectl patch service myapp -n prod -p '{"spec":{"selector":{"color":"green"}}}'

# After verification, clean up blue
helm uninstall myapp-blue --namespace prod
```

**Automated cutover via Helm hook (post-upgrade):**

```yaml
# templates/service-switcher-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-cutover
  annotations:
    helm.sh/hook: post-upgrade
    helm.sh/hook-weight: "10"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: cutover
          image: bitnami/kubectl
          command:
            - kubectl
            - patch
            - service
            - myapp
            - -n
            - {{ .Release.Namespace }}
            - -p
            - '{"spec":{"selector":{"color":"{{ .Values.color }}"}}}'
```

---

## 17.13 Rollback in CI/CD

### 17.13.1 Automated Rollback via `--atomic`

```bash
helm upgrade --install myapp ./mychart --atomic --timeout 10m
```

If the upgrade fails (pods crash, timeout exceeded), Helm automatically executes `helm rollback` to the last successful revision. No manual intervention needed.

### 17.13.2 Manual Rollback

```bash
# List release history
helm history myapp --namespace prod

# Rollback to a specific revision
helm rollback myapp 5 --namespace prod --wait --timeout 5m

# Rollback to the previous revision
helm rollback myapp 0 --namespace prod --wait
```

### 17.13.3 Git-Revert Based Rollback

When using GitOps (ArgoCD/Flux), rollback is typically a `git revert` of the commit that introduced the problematic change. The GitOps controller detects the reverted state and reconciles the cluster.

---

## 17.14 Secrets in CI/CD

**Never-commit checklist for CI/CD:**
- ✗ Secrets in `values.yaml`
- ✗ GPG private keys
- ✗ kubeconfig files
- ✗ SOPS master keys (age/pgp private keys)
- ✗ Registry passwords
- ✗ API tokens

**Instead, use CI/CD secret variables:**

| CI/CD Platform | Secret Mechanism |
|---|---|
| GitHub Actions | Repository secrets + environment secrets |
| GitLab CI | CI/CD Variables (Masked, Protected) |
| Jenkins | Credentials Plugin + `withCredentials` |
| Azure DevOps | Variable Groups + Azure Key Vault integration |
| Bitbucket Pipelines | Repository variables + secured variables |

Integrate with external secrets managers:

```bash
# AWS Secrets Manager in CI/CD
DB_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id prod/db-password \
  --query SecretString --output text)

# HashiCorp Vault in CI/CD
DB_PASSWORD=$(vault kv get -field=password secret/prod/database)
```

---

## 17.15 Multi-Cluster Deployments

```yaml
# GitHub Actions: deploy to multiple clusters
jobs:
  deploy-multi-cluster:
    strategy:
      matrix:
        cluster: [us-east-1, eu-west-1, ap-southeast-1]
    steps:
      - name: Configure kubeconfig
        run: |
          aws eks update-kubeconfig --region ${{ matrix.cluster }} \
            --name myapp-${{ matrix.cluster }} \
            --alias ${{ matrix.cluster }}

      - name: Deploy
        run: |
          helm upgrade --install myapp ./mychart \
            --kube-context ${{ matrix.cluster }} \
            -f values-${{ matrix.cluster }}.yaml \
            --atomic --wait --timeout 15m
```

Use `--kube-context` to target different clusters from a single CI/CD runner. Each cluster can have unique values overrides (different replica counts, resource limits, ingress hostnames).

---

## 17.16 GitOps vs CI-Push

| Dimension | GitOps (ArgoCD / Flux) | CI-Push (Helm in Pipeline) |
|---|---|---|
| **State store** | Git is the single source of truth | Pipeline action defines desired state |
| **Deployment trigger** | Git commit → controller detects drift → reconciles | Pipeline runs → pushes to cluster |
| **Drift detection** | Automatic (selfHeal in ArgoCD, driftDetection in Flux) | Manual (re-run pipeline) |
| **Rollback** | `git revert` → controller reconciles | `helm rollback` in pipeline or manual |
| **Auditability** | Full Git history of every change | Pipeline logs + Helm release history |
| **Secret handling** | Sealed Secrets, ESO (secrets live in cluster) | CI/CD variables, injected at deploy time |
| **Multi-cluster** | One controller per cluster or hub-spoke | Matrix-deploy in pipeline |
| **Complexity** | Higher (requires GitOps controller) | Lower (just Helm + pipeline) |
| **When to use** | Mature platform teams, many clusters, strict compliance | Smaller teams, fewer clusters, rapid iteration |

**Production Note:** The two approaches are not mutually exclusive. Many organizations use GitOps for steady-state management (ArgoCD watches Git) and CI-push for emergency hotfixes (pipeline pushes directly with Helm).

---

## 17.17 Complete Production Pipeline

```
                          PULL REQUEST (feature branch)
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              helm lint      helm template    trivy scan
              (syntax)      (dry-run)         (vulnerabilities)
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
                            ALL GATES PASSED?
                               │          │
                              YES         NO → PR blocked
                               │
                               ▼
                          MERGE TO MAIN
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
        helm package    cosign sign      grype scan rendered
        .tgz created    OCI artifact     manifest
              │                │                 │
              └────────────────┼─────────────────┘
                               ▼
                    PUSH TO OCI REGISTRY
                    (myapp-1.2.3 tagged)
                               │
              ┌────────────────┼────────────────┐
              ▼                                 ▼
        DEPLOY STAGING                   VERIFY SIGNATURE
  helm upgrade --install                    cosign verify
  --atomic --wait                       (trusted key check)
              │
              ▼
        HELM TEST (integration)
        + Smoke test endpoints
              │
              ▼
        MANUAL APPROVAL
              │
              ▼
        DEPLOY PRODUCTION
  helm upgrade --install
  --atomic --wait --timeout 15m
  (canary → 10% → monitor 5m → 100%)
              │
      ┌───────┴───────┐
      ▼               ▼
  SUCCESS         FAILURE
      │               │
      ▼               ▼
  Notify Slack   helm rollback --wait
  Tag release    + Notify PagerDuty
```

**Key properties of this pipeline:**
1. Every PR is linted, templated, and scanned before merge is allowed
2. The chart is packaged once and promoted unchanged through environments
3. Signatures are verified before deployment
4. Atomic deployments with automatic rollback eliminate manual recovery
5. Canary deployment minimizes production blast radius
6. Tests run after every stage deploy — the pipeline does not advance on failure

---

## 17.18 Azure DevOps — Minimal Example

```yaml
trigger:
  - main

pool:
  vmImage: ubuntu-latest

variables:
  - group: helm-secrets

stages:
  - stage: Lint
    jobs:
      - job: Lint
        steps:
          - task: HelmInstaller@1
            inputs:
              helmVersionToInstall: '3.15.0'
          - script: helm lint charts/myapp
  - stage: Deploy
    dependsOn: Lint
    jobs:
      - deployment: DeployStaging
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:
                - task: HelmDeploy@0
                  inputs:
                    command: upgrade
                    chartType: FilePath
                    chartPath: charts/myapp
                    releaseName: myapp
                    namespace: staging
                    valueFile: values-staging.yaml
                    install: true
                    waitForExecution: true
                    arguments: --atomic --wait --timeout 10m
```

---

## 17.19 Bitbucket Pipelines

```yaml
image: alpine/helm:3.15.0

pipelines:
  default:
    - step:
        name: Lint & Test
        script:
          - helm lint charts/myapp
          - helm template myapp charts/myapp -f values.yaml > /dev/null
  branches:
    main:
      - step:
          name: Build & Push
          script:
            - helm registry login ${OCI_REGISTRY} --username ${REGISTRY_USER} --password ${REGISTRY_PASS}
            - helm dependency update charts/myapp
            - helm package charts/myapp
            - helm push myapp-*.tgz oci://${OCI_REGISTRY}
      - step:
          name: Deploy Staging
          deployment: staging
          script:
            - echo "${KUBECONFIG_STAGING}" > /tmp/kubeconfig
            - export KUBECONFIG=/tmp/kubeconfig
            - helm upgrade --install myapp charts/myapp -n staging --create-namespace -f values-staging.yaml --atomic --wait --timeout 10m
      - step:
          name: Deploy Production
          deployment: production
          trigger: manual
          script:
            - echo "${KUBECONFIG_PROD}" > /tmp/kubeconfig
            - export KUBECONFIG=/tmp/kubeconfig
            - helm upgrade --install myapp charts/myapp -n production --create-namespace -f values-prod.yaml --atomic --wait --timeout 15m
```

---

## 17.20 CI/CD Best Practices Checklist

```
☐ 1.  ALWAYS RUN helm lint before packaging
☐ 2.  ALWAYS DRY-RUN (helm template) to validate rendering before deploy
☐ 3.  USE --atomic for automatic rollback on failed deployments
☐ 4.  USE --wait --wait-for-jobs with explicit --timeout
☐ 5.  USE --install for idempotent upgrade/install in pipelines
☐ 6.  PACKAGE ONCE, deploy the same .tgz artifact to all environments
☐ 7.  PROMOTE the same chart version through environments (not rebuild)
☐ 8.  SCAN CHARTS for vulnerabilities (Trivy/Grype) in CI before deploy
☐ 9.  VERIFY CHART SIGNATURES (helm verify or cosign) before deploy
☐ 10. NEVER COMMIT SECRETS — use CI/CD secret variables + SOPS/ESO
☐ 11. STORE CHARTS in OCI registries (immutable, signed, versioned)
☐ 12. PIN EXACT CHART VERSIONS — never use ranges or :latest
☐ 13. RUN helm test and smoke tests after every deployment
☐ 14. SEPARATE RBAC — use different ServiceAccounts per environment
☐ 15. USE ENVIRONMENT-SPECIFIC values files (never share across envs)
☐ 16. HAVE AN AUTOMATED ROLLBACK STRATEGY (--atomic or pipeline rollback step)
☐ 17. LOG RELEASE NOTES (--description) with CI/CD build metadata
☐ 18. USE MANUAL APPROVAL GATES for production deployments
☐ 19. NOTIFY TEAMS (Slack, PagerDuty) on deploy success and failure
☐ 20. CONFIGURE READINESS PROBES so --wait reflects actual readiness
☐ 21. ROTATE kubeconfig credentials regularly
☐ 22. USE SHORT-LIVED CI/CD TOKENS, not long-lived ServiceAccount tokens
```
