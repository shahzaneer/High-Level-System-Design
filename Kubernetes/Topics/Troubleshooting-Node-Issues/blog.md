# Troubleshooting: Node & Cluster Issues

Node-level problems affect all pods scheduled to that node. Common issues: node NotReady, disk pressure, memory pressure, network unavailable. Cluster-level issues: API server unreachable, etcd issues, CNI problems.

## Node NotReady

```bash
kubectl get nodes
# NAME        STATUS     ROLES    AGE   VERSION
# node-1      Ready      <none>   30d   v1.30.0
# node-2      NotReady   <none>   30d   v1.30.0

kubectl describe node node-2 | grep -A10 "Conditions:"
# Conditions:
#   Ready            False   KubeletNotReady   container runtime not running

# Common causes:
# - Kubelet stopped or crashed
# - Container runtime (containerd) stopped
# - Node out of disk/memory
# - Network plugin (CNI) issue
# - Kernel panic / node rebooted
```

## Node Pressure Conditions

```bash
kubectl describe node <node> | grep -A5 "Conditions:"
```

| Condition | Meaning | Action |
|-----------|---------|--------|
| MemoryPressure | Node low on memory | Evict pods, add node, or increase memory |
| DiskPressure | Node low on disk space | Clean up images/logs, add storage |
| PIDPressure | Too many processes | Check for process leak |
| NetworkUnavailable | CNI not configured | Check network plugin |

```bash
# Check node resource usage
kubectl top nodes
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory

# SSH to node (if permitted)
ssh node-2
df -h              # Disk space
free -h            # Memory
systemctl status kubelet
journalctl -u kubelet -f
crictl ps           # Container runtime status
```

## etcd Issues (Control Plane)

```bash
# etcd is the cluster's brain—if it's broken, the cluster can't function
# Check etcd health (on control plane nodes)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check etcd metrics
etcdctl endpoint status --write-out=table

# Common issues:
# - Quorum lost (too many control plane nodes down)
# - Disk full (etcd database growing; enable compaction)
# - Network latency between control plane nodes
```

## API Server Issues

```bash
# Check if API server is reachable
kubectl get --raw /healthz
kubectl get --raw /readyz

# Check API server logs (if you have access)
kubectl logs -n kube-system kube-apiserver-<node>
```

## CNI (Networking) Issues

```bash
# Common: pods can't communicate, services don't work
kubectl get pods -n kube-system | grep -E 'calico|weave|flannel|cilium'

# Check CNI pod logs
kubectl logs -n kube-system <cni-pod>

# Test pod-to-pod communication
kubectl run test-pod --image=busybox --rm -it -- nslookup kubernetes.default
kubectl run test-pod --image=busybox --rm -it -- wget -O- http://<service-ip>:8080
```

## Disk Space Recovery

```bash
# Clean up unused images (on nodes)
crictl rmi --prune

# Clean up stopped containers
crictl rm $(crictl ps -a -q --state exited)

# Check which pods use most storage
kubectl get pvc --all-namespaces --sort-by=.spec.resources.requests.storage

# Node-level: delete old logs
journalctl --vacuum-size=500M
```

## Quick Diagnostics

```bash
# Cluster overview
kubectl cluster-info
kubectl get componentstatuses    # Deprecated but still informative

# Critical control plane pods
kubectl get pods -n kube-system | grep -E 'etcd|apiserver|controller|scheduler|coredns'

# Node status summary
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[?(@.type==\"Ready\")].status,\
CPU:.status.capacity.cpu,\
MEMORY:.status.capacity.memory,\
KUBELET:.status.nodeInfo.kubeletVersion
```
