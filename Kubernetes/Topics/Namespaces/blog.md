# Namespaces

Namespaces partition cluster resources between multiple users, teams, or environments. They provide a scope for names (resource names must be unique within a namespace), resource isolation via ResourceQuotas, and access control via RBAC. Namespaces are NOT security boundaries for network isolation—use NetworkPolicies for that.

Default namespaces: `default` (catch-all), `kube-system` (control plane components), `kube-public` (publicly readable), `kube-node-lease` (node heartbeats).

## Imperative (kubectl)

```bash
# Create namespace
kubectl create namespace production
kubectl create namespace staging

# Run command in specific namespace
kubectl get pods -n production
kubectl describe pod order-service-abc -n production

# Switch default namespace context
kubectl config set-context --current --namespace=production

# Delete namespace (destroys ALL resources in it!)
kubectl delete namespace staging

# View all resources across all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A  # Short form
```

## Declarative (YAML)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    security-tier: high
---
# Resource with namespace
apiVersion: v1
kind: Pod
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
spec:
  containers:
  - name: app
    image: order-service:v1
```

## Namespace Strategy

```
Strategy 1: Per-Environment
  production/
  staging/
  development/
  
Strategy 2: Per-Team
  order-engineering/
  payment-engineering/
  platform-engineering/
  
Strategy 3: Per-Tenant (SaaS)
  tenant-acme/
  tenant-globex/
  tenant-initech/
  
Strategy 4: Hybrid
  production/ (per-environment)
    order-service
    payment-service
  order-team-dev/
  payment-team-dev/
```

## Resource Isolation

```yaml
# ResourceQuota: limit total resources per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    persistentvolumeclaims: "20"
    requests.storage: "500Gi"
---
# LimitRange: default/limits for individual pods
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
```

## Important Commands

```bash
# Namespace-scoped vs cluster-scoped resources
kubectl api-resources --namespaced=true   # Pods, Services, Deployments...
kubectl api-resources --namespaced=false  # Nodes, Namespaces, PVs, ClusterRoles...

# Clean up namespace (delete everything)
kubectl delete namespace test-ns  # Deletes ALL resources in the namespace

# Current namespace
kubectl config view --minify | grep namespace
```

## Imperative vs Declarative

Imperative (`kubectl create ns`) for quick namespace creation. Declarative for namespaces with ResourceQuotas, LimitRanges, and RBAC policies bundled together. Production namespaces should be created via GitOps with all policies attached.
