# DaemonSets

DaemonSets ensure that a copy of a Pod runs on every (or selected) node in the cluster. When nodes are added, the DaemonSet automatically schedules the pod on the new node. When nodes are removed, the pod is garbage collected. Typical use cases: log collectors (Fluentd/Fluent Bit), monitoring agents (Datadog, Node Exporter), storage daemons (CSI node plugins), network plugins (CNI), security agents (Falco).

## Imperative (kubectl)

```bash
# Create DaemonSet
kubectl create daemonset fluentd \
  --image=fluent/fluentd:v1.16 \
  --dry-run=client -o yaml > daemonset.yaml
# (No direct imperative creation—generate YAML then apply)

# View
kubectl get daemonset -A
kubectl describe daemonset fluentd
```

## Declarative (YAML)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true       # Use host network
      hostPID: true           # Access host processes
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule    # Run on control plane too
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.7.0
        ports:
        - containerPort: 9100
          hostPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

## Node Selector (Run on Specific Nodes)

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd       # Only nodes with label disktype=ssd
      # OR use affinity:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values: ["worker"]
```

## Common Use Cases

| DaemonSet | Purpose |
|-----------|---------|
| fluent-bit / fluentd | Collect container logs from every node |
| node-exporter | Expose node metrics to Prometheus |
| kube-proxy | Network proxy (part of control plane) |
| calico-node / weave-net | CNI networking |
| datadog-agent | Infrastructure monitoring |
| falco | Runtime security threat detection |
| aws-ebs-csi-node | EBS CSI driver node plugin |

## Update Strategies

| Strategy | Behavior |
|----------|----------|
| RollingUpdate (default) | Update pods one at a time across nodes |
| OnDelete | Only update when pod is manually deleted |

## Imperative vs Declarative

Always declarative for DaemonSets. The YAML involves tolerations, hostPath volumes, hostNetwork, and nodeSelectors that can't be expressed imperatively.
