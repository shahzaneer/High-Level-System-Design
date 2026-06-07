# Deployments

Deployments are the most common way to run stateless applications on Kubernetes. They manage ReplicaSets, which manage Pods. A Deployment declares the desired state (3 replicas of image X) and Kubernetes controllers continuously reconcile actual state to match. Deployments handle rolling updates, rollbacks, scaling, and self-healing.

## Imperative (kubectl)

```bash
# Create deployment
kubectl create deployment order-service \
  --image=order-service:v1.2.0 \
  --replicas=3 \
  --port=8080

# Scale manually
kubectl scale deployment order-service --replicas=5

# Update image
kubectl set image deployment/order-service app=order-service:v1.3.0

# Check rollout status
kubectl rollout status deployment/order-service

# Rollback
kubectl rollout undo deployment/order-service          # To previous version
kubectl rollout undo deployment/order-service --to-revision=2  # Specific revision

# View rollout history
kubectl rollout history deployment/order-service

# Restart (recreate all pods)
kubectl rollout restart deployment/order-service

# Pause/resume rollout (useful for canary-style manual control)
kubectl rollout pause deployment/order-service
kubectl rollout resume deployment/order-service

# Expose as service
kubectl expose deployment order-service --port=8080 --target-port=8080
```

## Declarative (YAML)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Create 1 extra pod during update
      maxUnavailable: 0    # Never go below desired count
  minReadySeconds: 10      # Wait after pod is ready before considering it available
  revisionHistoryLimit: 5  # Keep 5 old ReplicaSets for rollback
  template:
    metadata:
      labels:
        app: order-service
        version: v1
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: app
        image: order-service:v1.2.0
        ports:
        - containerPort: 8080
        resources:
          requests: { cpu: "250m", memory: "256Mi" }
          limits:   { cpu: "500m", memory: "512Mi" }
        readinessProbe:
          httpGet: { path: /ready, port: 8080 }
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: /health, port: 8080 }
          initialDelaySeconds: 30
          periodSeconds: 10
        env:
        - name: VERSION
          value: "v1.2.0"
```

```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods -l app=order-service
```

## Rolling Update Strategies

| Strategy | maxSurge | maxUnavailable | Behavior |
|----------|----------|---------------|----------|
| Fast (default) | 25% | 25% | Overlap old + new; some downtime possible |
| Safe | 1 | 0 | Never below desired; slightly slower |
| Replace all | 100% | 0 | Blue/green: all new ready before killing old |
| Recreate | N/A | N/A | Kill all old, then create new (downtime!) |

```yaml
# Recreate strategy (downtime, but simple for databases with single pod)
strategy:
  type: Recreate
```

## Rollback Mechanics

```bash
# Check what changed
kubectl rollout history deployment/order-service
# REVISION  CHANGE-CAUSE
# 1        kubectl apply --filename=deployment-v1.yaml
# 2        kubectl set image deployment/order-service app=order-service:v1.3.0

kubectl rollout undo deployment/order-service  # Back to revision 1

# Record change-cause for better history
kubectl annotate deployment/order-service \
  kubernetes.io/change-cause="Updated to v1.3.0 with payment fix"
```

## Common Patterns

```bash
# Restart pods (rolling restart)
kubectl rollout restart deployment/order-service

# Debug a specific revision without rolling back
kubectl rollout history deployment/order-service --revision=1
# Inspect the old ReplicaSet's pod template

# Scale to zero (stop service without deleting)
kubectl scale deployment order-service --replicas=0

# Zero-downtime deploy
kubectl set image deployment/order-service app=order-service:v1.3.0
kubectl rollout status deployment/order-service --timeout=5m
```

## Imperative vs Declarative

| Task | Imperative | Declarative |
|------|-----------|-------------|
| Quick test deploy | `kubectl create deploy` | Apply YAML |
| Production deploy | NO | YES (version controlled) |
| Emergency scale | `kubectl scale deploy X --replicas=10` | Edit YAML + apply |
| Rollback | `kubectl rollout undo` | Git revert + apply |

Always prefer declarative for production. Imperative is your Swiss Army knife for operations and debugging.
