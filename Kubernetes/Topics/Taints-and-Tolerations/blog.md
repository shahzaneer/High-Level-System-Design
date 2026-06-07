# Taints & Tolerations

Taints and Tolerations control which pods can schedule on which nodes. Taints are applied to NODES and repel pods. Tolerations are applied to PODS and allow them to "tolerate" the taint. This is NOT affinity—it's an exclusion mechanism: "this node is reserved for special pods."

## Tainting Nodes (Imperative)

```bash
# Syntax: kubectl taint nodes <node> <key>=<value>:<effect>

# Prevent ALL pods from scheduling on this node
kubectl taint nodes node-1 dedicated=database:NoSchedule

# GPU node: only GPU pods can schedule
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule

# Critical workload only (evict existing pods that don't tolerate)
kubectl taint nodes critical-node critical=true:NoExecute

# Soft preference: try not to schedule
kubectl taint nodes spot-node spot=true:PreferNoSchedule

# Remove taint (add '-' at end)
kubectl taint nodes node-1 dedicated=database:NoSchedule-
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule-

# View all taints on all nodes
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
TAINTS:.spec.taints

# Describe specific node
kubectl describe node node-1 | grep Taints
```

## Tolerations (Declarative in Pod Spec)

```yaml
apiVersion: v1
kind: Pod
spec:
  tolerations:
  # Tolerate GPU taint (this pod CAN run on GPU nodes)
  - key: "nvidia.com/gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  
  # Tolerate any taint with key "dedicated" (any value)
  - key: "dedicated"
    operator: "Exists"
    effect: "NoSchedule"
  
  # Tolerate ALL taints (use carefully!)
  - operator: "Exists"

  # Tolerate NoExecute with time limit (evict after 3600s)
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 3600
```

## Taint Effects

| Effect | What Happens |
|--------|-------------|
| `NoSchedule` | No new pods without matching toleration can schedule here. EXISTING pods unaffected. |
| `PreferNoSchedule` | Kubernetes TRIES not to schedule. Not a hard rule—pod may still land here. |
| `NoExecute` | EVICTS existing pods that don't tolerate the taint. Blocks new ones too. |

## Common Use Cases

```bash
# 1. Dedicated GPU nodes (expensive hardware)
kubectl taint nodes gpu-1 nvidia.com/gpu=true:NoSchedule

# Only GPU-workload pods have this toleration:
# tolerations:
# - key: nvidia.com/gpu
#   operator: Equal
#   value: "true"
#   effect: NoSchedule

# 2. Isolate database workloads
kubectl taint nodes db-node-1 workload=database:NoSchedule
kubectl taint nodes db-node-2 workload=database:NoSchedule

# DB pods tolerate it; other pods cannot land on these nodes

# 3. Draining a node for maintenance
kubectl taint nodes node-1 maintenance=true:NoExecute
# All pods without toleration are evicted immediately

# 4. Spot/preemptible instances (automatically tainted by cloud provider)
# Taint: kubernetes.azure.com/scalesetpriority=spot:NoSchedule
```

## Imperative vs Declarative

- **Tainting**: Always imperative (`kubectl taint`). Taints are node-level operations.
- **Tolerating**: Always declarative (in Pod YAML). Tolerations are pod-level specifications.
