# Labels & Selectors

Labels are key-value pairs attached to Kubernetes objects for identification, grouping, and selection. They are the glue that connects resources: a Service finds its Pods via label selectors, a Deployment finds its Pods via label selectors, and network policies target Pods via labels.

Labels are NOT annotations. Labels are used for selection and grouping (queryable, indexed). Annotations store non-identifying metadata (build info, contact info, tooling hints).

## Imperative (kubectl)

```bash
# Add label to resource
kubectl label pod order-service-abc123 environment=production
kubectl label node ip-10-0-1-5 disktype=ssd

# Add label to all pods
kubectl label pods --all tier=backend

# Overwrite existing label
kubectl label pod order-service-abc123 version=v2 --overwrite

# Remove label
kubectl label pod order-service-abc123 version-

# Show labels
kubectl get pods --show-labels
kubectl get pods -l app=order-service
kubectl get pods -l 'environment in (production,staging)'
kubectl get pods -l 'app=order-service,version!=v1'
kubectl get nodes -l disktype=ssd
```

## Declarative (YAML)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-service
  labels:
    app: order-service        # Application name
    version: v1.2.0           # Version for canary/blue-green
    tier: backend             # Architecture tier
    environment: production   # Environment
    team: order-engineering   # Owning team
    cost-center: retail       # FinOps tag
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    build: "abc123"
---
# Service using label selector
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    version: v1          # Only route to v1 pods
  ports:
  - port: 8080
---
# Deployment using matchLabels
apiVersion: apps/v1
kind: Deployment
spec:
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1
```

## Selector Types

| Type | Syntax | Example |
|------|--------|---------|
| Equality | key = value, key != value | `app=nginx` |
| Set-based | key in (val1, val2), key notin (val) | `env in (prod,staging)` |
| Exists | key (just check key exists) | `app` |
| Not exists | !key | `!temporary` |

## kubectl Label Commands

```bash
# Filtering pods with labels
kubectl get pods -l 'app=order-service,environment=production'
kubectl get pods -l 'version in (v1,v2)'
kubectl get pods -l 'app,environment'  # Has both app and environment labels

# Column output with custom labels
kubectl get pods -L app,version,environment
# NAME          READY  STATUS   APP             VERSION  ENVIRONMENT
# order-abc123  1/1    Running  order-service   v1.2.0   production
```

## Best Practices

1. **Label everything**: app, version, environment, tier, team at minimum
2. **Use meaningful keys**: `app.kubernetes.io/name`, `app.kubernetes.io/version`, `app.kubernetes.io/part-of`
3. **Recommended labels** (Kubernetes standard):
   ```yaml
   labels:
     app.kubernetes.io/name: order-service
     app.kubernetes.io/instance: order-service-prod
     app.kubernetes.io/version: "1.2.0"
     app.kubernetes.io/component: api
     app.kubernetes.io/part-of: ecommerce-platform
     app.kubernetes.io/managed-by: helm
   ```
4. **Immutable after creation**: Some labels can't change. Plan label schema before deploying.

## Imperative vs Declarative

Imperative label commands (`kubectl label`) are essential for operations: emergency labeling, debugging, adding missing labels. Declarative ensures labels are defined from the start in version-controlled manifests.
