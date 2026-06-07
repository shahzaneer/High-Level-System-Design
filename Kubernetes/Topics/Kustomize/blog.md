# Kustomize

Kustomize is a Kubernetes-native configuration management tool that customizes YAML without templates. Unlike Helm (which uses Go templating), Kustomize works with plain YAML files and applies overlays—patches, changes, additions—on top of a base configuration. It's built into kubectl (`kubectl apply -k`).

The model: base/ contains the common configuration; overlays/ contain environment-specific customizations. Each overlay patches the base without modifying the original files.

## Directory Structure

```
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
│
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replicas-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── replicas-patch.yaml
        └── resource-patch.yaml
```

## base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

namespace: default

commonLabels:
  app: order-service
  managed-by: kustomize

images:
- name: order-service
  newTag: v1.2.0

configMapGenerator:
- name: app-config
  files:
  - config.properties
  literals:
  - LOG_LEVEL=info
```

## overlays/production/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base              # Reference base

namespace: production

patches:
- path: replicas-patch.yaml
- path: resource-patch.yaml

images:
- name: order-service
  newTag: v1.2.0          # Override base image tag

configMapGenerator:
- name: app-config
  behavior: merge          # Merge with base configmap
  literals:
  - LOG_LEVEL=warn
  - API_TIMEOUT=15

secretGenerator:
- name: db-credentials
  literals:
  - username=admin
  - password=SuperSecret123!

# Strategic merge patch for HPA
patchesStrategicMerge:
- hpa-patch.yaml

# JSON 6902 patch
patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: order-service
  path: json-patch.yaml
```

## Patch Files

```yaml
# replicas-patch.yaml (strategic merge)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 5
---
# resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
---
# json-patch.yaml (JSON 6902 patch)
- op: add
  path: /spec/template/spec/affinity
  value:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: kubernetes.io/hostname
        labelSelector:
          matchLabels:
            app: order-service
```

## Imperative (kubectl + kustomize)

```bash
# Build and view rendered output (without applying)
kubectl kustomize overlays/production

# Apply directly
kubectl apply -k overlays/staging
kubectl apply -k overlays/production

# Delete
kubectl delete -k overlays/production

# Diff (show changes before applying)
kubectl diff -k overlays/production
```

## Helm vs Kustomize

| Aspect | Helm | Kustomize |
|--------|------|-----------|
| Templating | Go templates ({{ .Values.x }}) | Patch overlays on plain YAML |
| Packaging | Charts (.tgz), repositories | Git directories |
| Learning curve | Steeper (Go templates) | Gentler (plain YAML patches) |
| Best for | Distributing software to others | Managing your own deployments |
| Versioning | Chart version + app version | Git version (tags, branches) |
| Built into kubectl | No (separate CLI) | Yes (`kubectl apply -k`) |

## Generators

```yaml
# ConfigMap from file
configMapGenerator:
- name: nginx-config
  files:
  - nginx.conf
  - mime.types

# ConfigMap from literals
configMapGenerator:
- name: app-settings
  literals:
  - LOG_LEVEL=debug
  - TIMEOUT=30

# Secret generator
secretGenerator:
- name: app-secrets
  literals:
  - api-key=sk-live-xxxxxx

# Generators add content hash suffix to prevent stale config:
# nginx-config-abc123, app-settings-def456
```

## Transformers

```yaml
# Common transformers
namePrefix: prod-
nameSuffix: -v2
commonLabels:
  environment: production
commonAnnotations:
  contact: team@company.com
images:
- name: nginx
  newName: nginx
  newTag: alpine
```

## Imperative vs Declarative

Kustomize is declarative. The base + overlay model IS the declarative state. Use `kubectl apply -k` for deployments; `kubectl kustomize` to preview.
