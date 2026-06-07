# Pods

Pods are the smallest deployable units in Kubernetes—a group of one or more containers sharing network namespace, IPC, and storage volumes. Every container in a Pod shares `localhost` networking (they can reach each other on `localhost`), the same IP address, and can share volumes. Pods are ephemeral by design: they are created, scheduled to nodes, and when they die (node failure, eviction), they are NOT resurrected—higher-level controllers (Deployments, StatefulSets) handle that.

## Imperative (kubectl)

```bash
# Run a single-container Pod (simplest possible)
kubectl run nginx --image=nginx:alpine --port=80

# Run with environment variables
kubectl run order-service --image=order-service:v1 \
  --env="DATABASE_URL=postgresql://db:5432/orders" \
  --port=8080

# Run with resource limits
kubectl run batch-job --image=python:3.12 --restart=Never \
  --requests="cpu=250m,memory=256Mi" \
  --limits="cpu=500m,memory=512Mi" \
  --command -- python process.py

# Get Pod details
kubectl get pods
kubectl get pods -o wide              # Include node, IP
kubectl describe pod order-service    # Full details + events
kubectl logs order-service            # Stdout logs
kubectl logs order-service -c sidecar # Specific container logs
kubectl exec -it order-service -- /bin/sh  # Shell into container

# Delete Pod
kubectl delete pod order-service
kubectl delete pod --all              # Delete all pods in namespace
```

## Declarative (YAML)

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-service
  labels:
    app: order-service
    version: v1
spec:
  containers:
  - name: app
    image: order-service:v1.2.0
    ports:
    - containerPort: 8080
      protocol: TCP
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
  
  # Sidecar container (shares network + volumes with main container)
  - name: log-shipper
    image: fluent/fluent-bit:latest
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app
  
  volumes:
  - name: app-logs
    emptyDir: {}

  restartPolicy: Never  # Always (default), OnFailure, Never
```

```bash
kubectl apply -f pod.yaml
kubectl get pod order-service -o yaml  # See full state
kubectl delete -f pod.yaml
```

## Multi-Container Pods (Sidecar Pattern)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-proxy
spec:
  containers:
  - name: app
    image: myapp:v1
    ports:
    - containerPort: 8080
  
  # Envoy sidecar proxy: handles mTLS, traffic routing
  - name: envoy
    image: envoyproxy/envoy:v1.28
    ports:
    - containerPort: 15001
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
  
  volumes:
  - name: envoy-config
    configMap:
      name: envoy-config
```

## Init Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nslookup postgres; do sleep 2; done']
  
  - name: migrate-db
    image: myapp:v1
    command: ['python', 'manage.py', 'migrate']
  
  containers:
  - name: app
    image: myapp:v1
```

## When to Use Imperative vs Declarative

| Scenario | Approach |
|----------|----------|
| Quick test/debug | Imperative (`kubectl run`) |
| Production deployment | Declarative (`kubectl apply -f`) |
| CI/CD pipeline | Declarative (version-controlled YAML) |
| One-off investigation | Imperative |
| GitOps workflow | Declarative (YAML in Git) |

Golden rule: imperative for development/debugging; declarative for everything that matters.
