# Affinity, Anti-Affinity & Taints/Tolerations

These scheduling controls determine WHERE pods run. Affinity attracts pods to specific nodes or other pods. Anti-affinity repels pods from nodes or other pods. Taints repel pods from nodes; Tolerations allow pods to "tolerate" those taints and still schedule there. Together, they enable advanced scheduling strategies for performance, fault tolerance, and resource isolation.

## Node Affinity (Attract pods to nodes)

```yaml
apiVersion: v1
kind: Pod
spec:
  affinity:
    nodeAffinity:
      # MUST match (hard requirement)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a", "us-east-1b"]
      
      # PREFER to match (soft preference)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values: ["compute-optimized"]
```

## Pod Affinity (Co-locate pods)

```yaml
# Run this pod on same node as redis-cache (for low latency)
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - topologyKey: kubernetes.io/hostname
      labelSelector:
        matchLabels:
          app: redis-cache
```

## Pod Anti-Affinity (Spread pods apart)

```yaml
# Spread replicas across different nodes (fault tolerance)
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - topologyKey: kubernetes.io/hostname
      labelSelector:
        matchLabels:
          app: order-service
    
    # Prefer to spread across zones
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        topologyKey: topology.kubernetes.io/zone
        labelSelector:
          matchLabels:
            app: order-service
```

## Taints & Tolerations

```bash
# Taint a node (prevent pods from scheduling)
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule
kubectl taint nodes db-node-1 dedicated=database:NoExecute
kubectl taint nodes critical-node-1 critical=true:PreferNoSchedule

# Remove taint
kubectl taint nodes gpu-node-1 nvidia.com/gpu:NoSchedule-
```

```yaml
# Pod tolerating the taint (only this pod can run on gpu-node)
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoExecute"
    tolerationSeconds: 3600  # Evict after 1 hour
```

## Taint Effects

| Effect | Behavior |
|--------|----------|
| NoSchedule | No new pods without toleration |
| PreferNoSchedule | Try not to schedule (soft) |
| NoExecute | Evict existing pods without toleration + block new ones |

## Common Patterns

```
GPU workloads:
  Taint GPU nodes: nvidia.com/gpu=true:NoSchedule
  GPU pods: toleration for that taint
  Result: Only GPU pods run on expensive GPU nodes

Dedicated database nodes:
  Taint: dedicated=database:NoSchedule
  DB pods: toleration + nodeSelector
  Result: Database gets dedicated hardware, no noisy neighbors

Spread across AZs:
  Pod anti-affinity with topologyKey: topology.kubernetes.io/zone
  Result: Replicas distributed across availability zones
```

## Imperative

```bash
# Node management (imperative)
kubectl taint nodes node-1 key=value:NoSchedule
kubectl label nodes node-1 disktype=ssd

# Pod scheduling is declarative (YAML only)
```

## Imperative vs Declarative

Taints are imperative (`kubectl taint`). Pod affinities and tolerations are declarative (in Pod spec YAML). Production clusters should define taints as part of node provisioning automation.
