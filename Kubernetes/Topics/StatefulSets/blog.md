# StatefulSets

StatefulSets manage stateful applications—databases, message queues, clustered applications—that require stable identities, stable persistent storage, and ordered deployment/scaling. Unlike Deployments (where pods are interchangeable), StatefulSet pods have predictable names (`pod-0`, `pod-1`, `pod-2`), stable network identities, and dedicated PersistentVolumes that survive pod rescheduling.

Use StatefulSets for: PostgreSQL, MongoDB, Kafka, Zookeeper, Elasticsearch, Redis Cluster. Do NOT use for stateless apps (use Deployment) or for databases better managed by operators (use CloudNativePG, Strimzi Kafka).

## Imperative (kubectl)

```bash
# Create StatefulSet
kubectl create statefulset redis --image=redis:7-alpine \
  --replicas=3 --port=6379

# Scale
kubectl scale statefulset redis --replicas=5

# Note: imperative creates a headless service automatically
```

## Declarative (YAML)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless  # Required: headless service
  replicas: 3
  podManagementPolicy: OrderedReady  # or Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update all pods (set to 2 to only update pod-2+)
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: pgdata
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
  volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
---
# Headless service for stable network identities
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
```

## Stable Network Identity

```
Pod names:         DNS names:
postgres-0  →  postgres-0.postgres-headless.production.svc.cluster.local
postgres-1  →  postgres-1.postgres-headless.production.svc.cluster.local
postgres-2  →  postgres-2.postgres-headless.production.svc.cluster.local

Pod rescheduled to different node → SAME NAME, SAME DNS, SAME VOLUME
```

## Pod Management Policies

| Policy | Behavior |
|--------|----------|
| OrderedReady | Start/stop in order: 0→1→2 and 2→1→0 (safe for clustered DBs) |
| Parallel | Start/stop simultaneously (faster; only for independent pods) |

## Update Strategies

```yaml
# Rolling update with canary (partition)
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2  # Only update pod-2 and above (pod-2 only with 3 replicas)
# Test new version on one pod, then set partition: 0 for full rollout
```

## StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod identity | Random suffix (abc-xyz123) | Ordered index (app-0, app-1) |
| DNS name | None (only via Service) | Stable per-pod DNS |
| Storage | Shared via PVC template (shared volume) | Dedicated PVC per pod (volumeClaimTemplates) |
| Startup/shutdown | Parallel | Ordered (configurable) |
| Use case | Stateless web/API servers | Databases, caches, queues |

## Imperative vs Declarative

Imperative `kubectl create statefulset` is useful for quick test. Production requires YAML to define volumeClaimTemplates, headless service, podManagementPolicy, and update strategy.
