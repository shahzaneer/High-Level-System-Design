# Helm

Helm is the package manager for Kubernetes. It bundles related Kubernetes resources into "charts" (templates + values), enabling versioned, reusable, and configurable deployments. Helm is to Kubernetes what apt/yum is to Linux.

Key concepts: Chart (package of templates), Repository (where charts are stored), Release (instance of a chart deployed to a cluster), Values (configuration that customizes the chart).

## Imperative (helm CLI)

```bash
# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo postgresql
helm search hub prometheus

# Install a chart
helm install my-release bitnami/postgresql \
  --namespace production \
  --set auth.password=SecretP@ss \
  --set persistence.size=100Gi

# Or with values file
helm install my-release bitnami/postgresql \
  -f values-production.yaml \
  -n production

# List releases
helm list -n production
helm list --all-namespaces

# Get release info
helm status my-release -n production
helm get values my-release -n production     # See what values were used
helm get manifest my-release -n production   # See generated YAML

# Upgrade release
helm upgrade my-release bitnami/postgresql \
  --set persistence.size=200Gi \
  -n production

# Rollback
helm rollback my-release 1 -n production
helm history my-release -n production       # See revision history

# Uninstall
helm uninstall my-release -n production
```

## Chart Structure

```
my-chart/
├── Chart.yaml           # Chart metadata (name, version, description)
├── values.yaml          # Default configuration values
├── values-production.yaml  # Environment-specific overrides
├── charts/              # Sub-charts (dependencies)
├── templates/           # Kubernetes YAML templates with Go templating
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl     # Template helpers (reusable snippets)
│   └── NOTES.txt        # Post-install instructions
├── .helmignore
└── Chart.lock           # Dependency lock file
```

## Chart.yaml

```yaml
apiVersion: v2
name: order-service
description: Order processing microservice
type: application
version: 1.2.0           # Chart version
appVersion: "1.2.0"      # Application version

dependencies:
- name: postgresql
  version: 15.x.x
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled

maintainers:
- name: Order Engineering
  email: order-eng@company.com
```

## Template Example (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        version: {{ .Chart.AppVersion }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- with .Values.env }}
        env:
          {{- toYaml . | nindent 10 }}
        {{- end }}
```

## values.yaml + Environment Override

```yaml
# values.yaml (defaults)
replicaCount: 2
image:
  repository: order-service
  tag: latest
service:
  type: ClusterIP
  port: 8080
resources:
  requests:
    cpu: 250m
    memory: 256Mi
env:
  LOG_LEVEL: info

---
# values-production.yaml (override)
replicaCount: 10
image:
  tag: v1.2.0    # Override tag
resources:
  requests:
    cpu: 500m
    memory: 512Mi
env:
  LOG_LEVEL: warn
```

```bash
# Install with environment-specific values
helm install order-service ./order-service-chart \
  -f values-production.yaml \
  -n production

# Template debugging (render without installing)
helm template order-service ./order-service-chart \
  -f values-production.yaml

# Lint chart
helm lint ./order-service-chart
```

## Common Commands

```bash
helm repo list                                 # List repos
helm search repo nginx                         # Search
helm install test ./chart --dry-run --debug    # Preview install
helm upgrade release ./chart --atomic          # Auto-rollback on failure
helm upgrade release ./chart --wait --timeout 5m
helm dependency update ./chart                 # Download sub-charts
helm package ./chart                           # Create .tgz package
```

## Imperative vs Declarative

Helm is declarative (values + templates) but operated imperatively via CLI. GitOps platforms (ArgoCD, Flux) can deploy Helm charts declaratively by referencing the chart and values in Git.
