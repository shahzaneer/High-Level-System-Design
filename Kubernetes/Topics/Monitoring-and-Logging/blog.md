# Monitoring & Logging in Kubernetes

Observability in Kubernetes requires a dedicated stack. The cluster itself emits metrics (kube-state-metrics, node-exporter), logs (container stdout/stderr), and events. You must deploy and operate: metrics server, Prometheus, Grafana, log aggregator, and optionally distributed tracing.

## Metrics Stack

### Metrics Server (Required for HPA)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top nodes
kubectl top pods -n production
kubectl top pods --containers -n production
```

### Prometheus + Grafana

```bash
# Prometheus stack (kube-prometheus-stack Helm chart)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=Admin123

# Components deployed:
# - Prometheus (metrics collection)
# - Alertmanager (alerting)
# - Grafana (dashboards)
# - node-exporter (node metrics)
# - kube-state-metrics (K8s object metrics)
# - Prometheus Operator (manages Prometheus instances)
```

### Key Metrics

```promql
# Node CPU usage
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Pod memory usage
sum(container_memory_working_set_bytes{namespace="production"}) by (pod)

# Deployment replicas (available vs desired)
kube_deployment_status_replicas_available{deployment="order-service"}
kube_deployment_spec_replicas{deployment="order-service"}

# Service error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# PVC usage vs capacity
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes
```

## Logging Stack

### kubectl Logs (Direct)

```bash
# Basic log access
kubectl logs deployment/order-service -n production
kubectl logs -f order-service-abc123 -n production     # Follow
kubectl logs order-service-abc123 --previous            # Previous crashed container
kubectl logs order-service-abc123 -c sidecar            # Specific container
kubectl logs order-service-abc123 --tail=100            # Last 100 lines
kubectl logs order-service-abc123 --since=5m            # Last 5 minutes

# Multi-container pod
kubectl logs order-service-abc123 --all-containers=true
```

### Centralized Logging (Loki + Grafana)

```yaml
# Promtail DaemonSet: collects logs from every node
# Fluent Bit alternative: lighter weight, more performant

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:3.0
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath: { path: /var/log }
      - name: varlibdockercontainers
        hostPath: { path: /var/lib/docker/containers }
```

### EFK Stack (Elasticsearch + Fluentd + Kibana)

```bash
# ECK Operator (Elastic Cloud on Kubernetes)
kubectl apply -f https://download.elastic.co/downloads/eck/latest/crds.yaml

# Then deploy Elasticsearch, Kibana, and Fluentd/Fluent Bit
```

## Events Monitoring

```bash
kubectl get events --sort-by=.lastTimestamp -n production
kubectl get events --field-selector type=Warning -A
kubectl get events -w -n production    # Watch mode

# Common problematic events:
# - FailedScheduling, Failed, Unhealthy, BackOff, FailedMount
# - OOMKilling, Evicted, NodeNotReady
```

## Audit Logs

```yaml
# Kubernetes audit policy (configured in API server)
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  # Log all requests at metadata level
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
  # Log full request/response for sensitive resources
```

```bash
# View audit logs (on control plane nodes)
cat /var/log/kubernetes/audit/audit.log | jq .
```

## Essential Dashboard Setup

```
Must-have Grafana dashboards:
1. Kubernetes Cluster Overview (nodes, CPU, memory, pods)
2. Node Exporter (per-node CPU, memory, disk, network)
3. K8s Pod Overview (per-pod CPU, memory, network, restarts)
4. Application RED Dashboard (Rate, Errors, Duration per service)
5. Persistent Volume Dashboard (capacity, usage, throughput)
```

## Imperative vs Declarative

Observability infrastructure is deployed declaratively (Helm charts, YAML). Day-to-day operations are imperative (`kubectl logs`, `kubectl top`, `kubectl describe`).
