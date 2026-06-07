# Operators & Custom Resource Definitions (CRDs)

Operators extend Kubernetes to manage complex stateful applications automatically. They encode operational knowledge (how to deploy, scale, backup, upgrade, recover) into software running on the cluster. A CRD defines a new resource type; an Operator acts on those custom resources.

Canonical example: etcd Operator knows how to manage an etcd cluster—deploy, scale, backup, handle failures. A database operator knows how to provision databases, take backups, handle failover. Without operators, these operational tasks are manual runbooks.

## Custom Resource Definition (CRD)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.postgres.example.com
spec:
  group: postgres.example.com
  names:
    kind: PostgresCluster
    listKind: PostgresClusterList
    plural: postgresclusters
    singular: postgrescluster
    shortNames: ["pg"]
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 5
              storageSize:
                type: string
                pattern: '^[0-9]+Gi$'
              postgresVersion:
                type: string
                enum: ["14", "15", "16"]
            required: ["replicas", "storageSize"]
          status:
            type: object
            properties:
              phase:
                type: string
                enum: ["Creating", "Running", "Updating", "Failed"]
              primaryEndpoint:
                type: string
    subresources:
      status: {}
```

## Custom Resource (Instantiating the CRD)

```yaml
apiVersion: postgres.example.com/v1
kind: PostgresCluster
metadata:
  name: orders-db
  namespace: production
spec:
  replicas: 3
  storageSize: "100Gi"
  postgresVersion: "16"
```

## Operator Pattern

```
User creates Custom Resource → Operator watches for changes → Operator reconciles

Reconciliation loop:
  1. Observe: Read current state of the world
  2. Compare: Does actual state match desired state?
  3. Act: Create/update/delete resources to match desired state
  4. Repeat: Watch for next change
```

## Imperative (kubectl)

```bash
# Apply CRD
kubectl apply -f postgres-crd.yaml

# Create custom resource
kubectl apply -f orders-db.yaml

# List custom resources
kubectl get postgresclusters -n production
kubectl get pg -n production    # Short name

# Describe (shows status from operator)
kubectl describe postgrescluster orders-db -n production

# Delete (operator handles cleanup)
kubectl delete postgrescluster orders-db -n production
```

## Operator Frameworks

| Framework | Language | Best For |
|-----------|----------|----------|
| Operator SDK | Go, Ansible, Helm | Go-native operators, Ansible for simpler logic |
| Kopf | Python | Python developers, simpler operators |
| Kudo | YAML | Declarative operators, no coding |
| Metacontroller | Any (webhook) | Lightweight, lambda-style controllers |

## Popular Operators

```bash
# CloudNativePG (PostgreSQL)
kubectl apply -f https://cloudnative-pg.io/quickstart

# Strimzi (Kafka)
kubectl create -f 'https://strimzi.io/install/latest?namespace=default'

# Prometheus Operator
kubectl apply -f https://github.com/prometheus-operator/prometheus-operator

# cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager

# Elastic Cloud on Kubernetes (ECK)
kubectl apply -f https://download.elastic.co/downloads/eck/latest/crds.yaml
```

## Operator Lifecycle Manager (OLM)

```bash
# OLM manages operator installation, upgrades, and dependency resolution
# Installed by default on OpenShift; available for vanilla K8s

kubectl get operators        # Installed operators
kubectl get subscriptions    # Operator subscriptions
kubectl get csv             # ClusterServiceVersion (operator version info)
```

## When to Use Operators

| Scenario | Use Operator? |
|----------|--------------|
| Simple stateless app | No—use Deployment + Helm |
| Database with manual failover | YES—operator handles failover |
| Application with complex upgrade process | YES—operator encodes upgrade logic |
| Infrastructure component (cert-manager) | YES—standard pattern |
| Simple config generator | No—use Helm or Kustomize |

## Imperative vs Declarative

CRDs and Custom Resources are declarative (YAML manifests). Operators are software (running in pods) that react to these declarations. The CR YAML is the "desired state"; the operator is the controller reconciling reality to that desired state.
