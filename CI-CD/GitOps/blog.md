# GitOps

## Introduction
GitOps is a paradigm for continuous delivery that uses Git as the single source of truth for declarative infrastructure and application configuration. Coined by Weaveworks in 2017, GitOps extends the DevOps principle of "everything as code" to operations: the desired state of the entire system is versioned in Git, and automated processes continuously reconcile the actual state of the cluster with the declared state in Git.

The key insight is that Git already provides the primitives operations needs: version control (every change is tracked), pull requests (changes are reviewed before applying), audit logs (who changed what and when), and rollback (revert to any previous state). GitOps applies these proven developer workflows to infrastructure and deployment management, eliminating the gap between "the code in the repo" and "what's actually running."

## Definition

**GitOps** is an operational framework that takes DevOps best practices used for application development (version control, collaboration, compliance, CI/CD) and applies them to infrastructure automation. Its core principles:

1. **Declarative Configuration**: The entire system state is described declaratively (Kubernetes YAML, Terraform, Helm charts)
2. **Git as Source of Truth**: All desired state is stored in Git; there is no other source of truth
3. **Automated Reconciliation**: Software agents (ArgoCD, Flux) continuously reconcile actual state with desired state in Git
4. **Closed-Loop Feedback**: If actual state diverges from Git (drift detection), the system either auto-corrects or alerts

## Concept Explanation

### Push vs Pull Deployment

```
PUSH MODEL (Traditional CI/CD):
  CI Pipeline → kubectl apply → Kubernetes
  (CI system has cluster credentials; pushes changes)

PULL MODEL (GitOps):
  Git Repo (desired state) ←→ ArgoCD/Flux (pulls) → Kubernetes
  (Agent inside cluster pulls from Git; CI system doesn't touch cluster)
```

```yaml
# ArgoCD Application: declares what should run and where
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-config
    targetRevision: main
    path: apps/order-service/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true      # Delete resources that are removed from Git
      selfHeal: true   # Revert manual changes to match Git
    syncOptions:
      - CreateNamespace=true
```

### Git Repository Structure

```
gitops-config/
├── clusters/
│   ├── production/
│   │   ├── cluster-config.yaml     # Ingress, cert-manager, monitoring
│   │   └── apps/
│   │       ├── order-service/
│   │       │   ├── deployment.yaml
│   │       │   ├── service.yaml
│   │       │   ├── ingress.yaml
│   │       │   └── kustomization.yaml
│   │       └── payment-service/
│   └── staging/
│       └── apps/
├── infrastructure/
│   ├── monitoring/
│   ├── logging/
│   └── ingress/
└── helm-charts/
    └── order-service/
        ├── Chart.yaml
        ├── values.yaml
        ├── values-production.yaml
        └── templates/
```

### ArgoCD Architecture

```
┌─────────────────────────────────────────────┐
│                  ARGOCD                      │
│                                              │
│  ┌──────────┐   ┌──────────────┐           │
│  │ API/UI   │   │ Application  │           │
│  │          │   │ Controller   │           │
│  └──────────┘   └──────┬───────┘           │
│                        │                    │
│          ┌─────────────┼─────────────┐      │
│          ▼             ▼             ▼      │
│    ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│    │ Git Repo │ │ K8s API  │ │ Health   │  │
│    │ (desired)│ │ (actual) │ │ Checks   │  │
│    └──────────┘ └──────────┘ └──────────┘  │
│                                              │
│  Reconciliation loop (every 3 minutes):     │
│    1. Fetch desired state from Git repos     │
│    2. Fetch actual state from K8s API       │
│    3. Diff: what's different?               │
│    4. Sync: apply changes to match desired  │
│    5. Health: verify deployment is healthy  │
└─────────────────────────────────────────────┘
```

### Flux CD Architecture

```yaml
# Flux: lightweight, CNCF-graduated GitOps tool
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops-config
spec:
  interval: 1m
  url: https://github.com/org/gitops-config
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: production-apps
spec:
  interval: 5m
  path: ./clusters/production/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-config
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: order-service
      namespace: production
```

### Progressive Delivery with Argo Rollouts

```yaml
# Argo Rollouts: canary deployment for Kubernetes
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 10     # 10% to new version
      - pause: {duration: 60s}  # Wait 60 seconds
      - setWeight: 30
      - pause: {duration: 60s}
      - setWeight: 50
      - pause: {}  # Manual approval required
      - setWeight: 100
  selector:
    matchLabels:
      app: order-service
  template:
    # Pod spec (same as Deployment)
```

### Image Updater (Automated Image Updates)

```yaml
# ArgoCD Image Updater: automatically updates image tags
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  annotations:
    argocd-image-updater.argoproj.io/image-list: order-service=123456789.dkr.ecr.us-east-1.amazonaws.com/order-service
    argocd-image-updater.argoproj.io/order-service.update-strategy: semver
    argocd-image-updater.argoproj.io/order-service.allow-tags: "regexp:^[0-9]+\\.[0-9]+\\.[0-9]+$"
    argocd-image-updater.argoproj.io/write-back-method: git
```

### Secrets in GitOps

```yaml
# PROBLEM: Don't store plaintext secrets in Git!
# SOLUTION: Encrypted secrets or external references

# Option 1: Sealed Secrets (Bitnami)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    password: AgBy3i4...encrypted...base64...

# Option 2: External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: prod/database/password

# Option 3: SOPS (Mozilla)
# Encrypted YAML files committed to Git, decrypted at deploy time
```

### Multi-Cluster Management

```yaml
# ArgoCD ApplicationSet: deploy across multiple clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: order-service
spec:
  generators:
  - list:
      elements:
      - cluster: production-us
        url: https://prod-us-cluster.example.com
      - cluster: production-eu
        url: https://prod-eu-cluster.example.com
      - cluster: staging
        url: https://staging-cluster.example.com
  template:
    metadata:
      name: 'order-service-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/gitops-config
        path: apps/order-service/overlays/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: production
```

### Drift Detection

```python
class DriftDetector:
    """
    GitOps agents continuously detect drift between Git (desired) 
    and Kubernetes (actual). Options:
    """
    
    def detect(self, desired_state, actual_state):
        drift = self.diff(desired_state, actual_state)
        
        if drift:
            if self.policy == 'AUTO_CORRECT':
                # ArgoCD with selfHeal: true
                # Automatically revert actual to match desired
                self.sync(desired_state)
                self.alert("Drift detected and auto-corrected")
            
            elif self.policy == 'ALERT_ONLY':
                # ArgoCD with selfHeal: false
                # Alert but don't auto-correct
                self.alert(f"Drift detected: {drift.summary()}")
            
            elif self.policy == 'NEVER_SYNC':
                # Flux with selfHeal: false
                # Detected but neither corrected nor alerted
                # Seen in UI as "OutOfSync"
                pass
    
    def common_causes_of_drift(self):
        return [
            "Someone ran kubectl apply manually on the cluster",
            "Someone edited a resource in the cloud console",
            "An operator or controller modified the resource",
            "A HPA (Horizontal Pod Autoscaler) changed replica count"
        ]
```

## Layman's Explanation

### The Thermostat Analogy
GitOps is like a smart thermostat for your infrastructure:

- **Desired State (Git)**: You set the thermostat to 72°F (committed in Git: "order-service should have 3 replicas")
- **Actual State (Kubernetes)**: The current temperature is 68°F (actual: only 2 replicas running because one crashed)
- **Reconciliation (ArgoCD/Flux)**: The thermostat detects the gap and turns on the heat (starts a new pod to reach 3 replicas)
- **Self-Healing**: If someone opens a window (manual kubectl edit changes replica count to 5), the thermostat detects the change and brings it back to 72°F (reverts to 3 replicas)
- **Git Log = Temperature History**: Every time someone changes the thermostat setting, it's recorded (git log). If someone sets it to 90°F by mistake, you can revert to the previous setting (git revert).

### Why This Is Better Than Manual Operations
Traditional ops: "SSH into server, edit config file, restart service." Nobody knows who changed what, when, or why. Recovery = "does anyone remember the old config?"

GitOps: "Create pull request to change replicas from 3 to 5. Review. Merge. ArgoCD deploys automatically." Full audit trail. One-click rollback. Blame is a feature, not a problem.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **ArgoCD vs Flux**: ArgoCD has a rich UI and is easier for teams new to GitOps. Flux is more CLI-focused, lighter weight, and CNCF-graduated. Both are excellent. The choice often comes down to team preference and existing tools (Flux if using Weaveworks, ArgoCD if using Codefresh/Intuit ecosystem).
- **Monorepo vs Multi-Repo for GitOps**: Single GitOps repo for all clusters (simpler, single source of truth) vs per-app repos (better for independent teams, but harder to get a holistic view). Monorepo is simpler; poly-repo requires ApplicationSets (ArgoCD) or additional tooling.
- **Secret Management Strategy**: Sealed Secrets (encrypt and commit to Git—simple but manual rotation), External Secrets Operator (references cloud secret stores—more setup but better security posture), SOPS (encrypt YAML with KMS—balanced approach).
- **Cluster Bootstrapping**: How do you deploy ArgoCD/Flux itself? The "app of apps" pattern: a bootstrap Application that deploys all other Applications. This creates a self-managing cluster—once bootstrapped, the cluster manages itself via GitOps.

### Business Impact
- **Disaster Recovery**: A destroyed cluster can be completely rebuilt from Git in under an hour. Every configuration, every application, every setting is in Git. Compare this to traditional DR where rebuilding from documentation takes days.
- **Audit & Compliance**: Git history provides a complete, immutable audit trail of every infrastructure change. SOC 2 auditors love GitOps—every change is reviewed (PR), approved, timestamped, and attributable to a specific person.
- **Developer Self-Service**: Developers deploy to production by merging a PR. No ticket to ops. No waiting for a deployment window. This is the ultimate expression of "you build it, you run it."

## AWS + GCP + Azure Examples

### AWS: ArgoCD on EKS
```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose via AWS Load Balancer Controller
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### GCP: Config Sync (Managed GitOps)
```bash
gcloud container clusters update production-cluster \
  --enable-config-sync

# Config Sync watches a Git repository and syncs to the cluster
# Managed by GCP—no ArgoCD/Flux to operate
```

### Azure: Flux Extension for AKS
```bash
az k8s-extension create \
  --name flux \
  --extension-type microsoft.flux \
  --cluster-name production-cluster \
  --resource-group myRG \
  --cluster-type managedClusters

az k8s-configuration flux create \
  --name gitops-config \
  --cluster-name production-cluster \
  --resource-group myRG \
  --namespace flux-system \
  --url https://github.com/org/gitops-config \
  --branch main
```

## Summary

| GitOps Tool | Best For | UI | Complexity |
|------------|----------|-----|------------|
| ArgoCD | Teams wanting rich UI, multi-cluster | Excellent | Medium |
| Flux CD | CLI-focused, CNCF-native | Basic | Medium-Low |
| Config Sync | GCP-managed, hands-off | GCP Console | Low (managed) |
| Azure Flux | Azure-managed, native integration | Azure Portal | Low (managed) |

GitOps applies the software development lifecycle to operations: version control, code review, automated testing, and continuous delivery for infrastructure. The declarative model ensures that what's in Git IS what's running—if they disagree, reconciliation brings them in sync. For a solution architect, adopting GitOps means designing the repository structure, choosing the reconciliation tool, defining the promotion flow (dev → staging → production), and establishing the secret management strategy. GitOps transforms operations from a source of risk (manual changes, configuration drift) into a source of confidence (every change auditable, every state reproducible).
