# HPA & VPA (Auto-Scaling)

Kubernetes has two auto-scaling mechanisms for workloads: Horizontal Pod Autoscaler (HPA) adjusts the number of pod replicas based on metrics, and Vertical Pod Autoscaler (VPA) adjusts CPU/memory requests and limits. They are complementary: HPA scales out/in, VPA scales up/down.

## HPA - Horizontal Pod Autoscaler

### Imperative

```bash
# Create HPA (CPU-based)
kubectl autoscale deployment order-service \
  --cpu-percent=70 \
  --min=3 --max=20

# View HPA status
kubectl get hpa
kubectl describe hpa order-service
```

### Declarative (YAML)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  # CPU-based
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory-based
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom metric (Prometheus)
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: "500"
  
  # External metric (SQS queue depth)
  - type: External
    external:
      metric:
        name: sqs_queue_depth
        selector:
          matchLabels:
            queue: orders
      target:
        type: AverageValue
        averageValue: "100"

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10        # Scale down max 10% at a time
        periodSeconds: 60
      selectPolicy: Min
    scaleUp:
      stabilizationWindowSeconds: 0    # Scale up immediately
      policies:
      - type: Pods
        value: 4         # Add max 4 pods at a time
        periodSeconds: 60
```

### HPA Formula

```
desiredReplicas = ceil(currentReplicas * (currentMetric / targetMetric))

Example:
  currentReplicas = 3
  currentCPU = 85%
  targetCPU = 70%
  desired = ceil(3 * 85/70) = ceil(3.64) = 4
```

## VPA - Vertical Pod Autoscaler

```bash
# VPA is an add-on; install separately
# AWS EKS: not managed; install manually
# GKE: built-in, enable with flag
# AKS: managed add-on
```

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: Auto  # Off, Initial, Recreate, Auto
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "4"
        memory: "8Gi"
      controlledResources: ["cpu", "memory"]
```

### VPA Update Modes

| Mode | Behavior |
|------|----------|
| Off | Only recommends; no changes (safe start) |
| Initial | Sets requests at pod creation; no restart |
| Recreate | Restarts pod with new values (downtime) |
| Auto | Recreate mode automatically |

### HPA + VPA Together

```
HPA: Scales OUT (adds more pods) when CPU > 70%
VPA: Scales UP (increases pod resources) when requests are wrong

Warning: Don't use HPA on CPU with VPA on CPU—feedback loop!
  HPA thinks "CPU high, add pods"
  VPA thinks "CPU high, increase requests"
  Solution: Use VPA for memory only, or use HPA with VPA in Off mode
```

## Imperative vs Declarative

`kubectl autoscale` is good for quick CPU-based HPA. Production always uses declarative with multi-metric, custom metrics, and scaling behavior policies.
