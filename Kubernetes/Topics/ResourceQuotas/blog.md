# ResourceQuotas & LimitRanges

ResourceQuotas limit aggregate resource consumption per namespace. LimitRanges set default, minimum, and maximum resource constraints for individual Pods and containers. Together, they prevent resource hogging, enforce cost controls, and ensure fair sharing of cluster capacity.

Without these, a single namespace can consume all cluster resources, and a single Pod without resource limits can OOM-kill other workloads.

## Declarative (YAML)

```yaml
# ResourceQuota: namespace-level aggregate limits
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: order-engineering
spec:
  hard:
    # Compute
    requests.cpu: "50"
    requests.memory: "100Gi"
    limits.cpu: "100"
    limits.memory: "200Gi"
    
    # Object count limits
    pods: "50"
    services: "20"
    secrets: "50"
    configmaps: "30"
    persistentvolumeclaims: "10"
    
    # Storage
    requests.storage: "500Gi"
    
    # Extended resources (GPUs)
    requests.nvidia.com/gpu: "4"
---
# LimitRange: per-container defaults and constraints
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: order-engineering
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
    maxLimitRequestRatio:
      cpu: "4"  # Limits can be at most 4x requests
  
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"
    min:
      storage: "1Gi"
```

## Imperative (kubectl)

```bash
# Create ResourceQuota
kubectl create quota team-quota \
  --hard=cpu=50,memory=100Gi,pods=50,services=20 \
  -n order-engineering

# View quotas
kubectl describe quota -n order-engineering
kubectl get resourcequota -n order-engineering
```

## Resource Requests vs Limits

```yaml
resources:
  requests:           # Kubernetes GUARANTEES this amount
    cpu: "500m"       # 0.5 CPU cores
    memory: "512Mi"   # 512 MB RAM
  
  limits:             # Pod CANNOT exceed this
    cpu: "1000m"      # Max 1 CPU core (throttled if exceeds)
    memory: "1024Mi"  # Max 1 GB RAM (OOM killed if exceeds)
```

| Spec | Purpose | If exceeded |
|------|---------|------------|
| requests.cpu | CPU guaranteed | Throttles to request if node CPU saturated |
| limits.cpu | CPU cap | Throttled (never exceeds, just slower) |
| requests.memory | Memory guaranteed | OOM killed if node can't provide |
| limits.memory | Memory cap | OOM killed immediately |

## CPU Units

```
1000m = 1 CPU core (or 1 vCPU / hyperthread)
500m  = 0.5 CPU
250m  = 0.25 CPU
100m  = 0.1 CPU (minimum reasonable request)

"1" = 1000m (same thing)
```

## Quality of Service (QoS) Classes

| Class | Condition | Behavior |
|-------|-----------|----------|
| Guaranteed | requests == limits for CPU AND memory | Last to be evicted |
| Burstable | requests < limits, or missing limits | Evicted after Guaranteed |
| BestEffort | No requests/limits at all | First to be evicted |

## Best Practices

1. **Always set requests and limits**: No exceptions for production
2. **requests ~= typical usage**: Set requests at your p50-p75 utilization
3. **limits = burst ceiling**: Set limits at 2-3x requests for CPU, same for memory
4. **Don't set CPU limits on JVM apps**: JVM GC can spike CPU; throttling causes GC death spiral
5. **Set memory requests = limits**: Memory overcommit kills pods unpredictably
6. **Use VPA in recommendation mode first**: Let VPA learn before auto-applying

## Imperative vs Declarative

Imperative for quick quota checks (`kubectl describe quota`). Declarative for defining quotas—they must be versioned alongside namespace definitions in GitOps.
