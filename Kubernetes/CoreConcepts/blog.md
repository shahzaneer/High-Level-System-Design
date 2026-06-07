# Kubernetes: Core Concepts

## Introduction
Kubernetes (K8s) is the de facto standard for container orchestration—the system that decides where containers run, ensures they keep running, scales them up and down, routes traffic to them, and manages their configuration. Born from Google's internal Borg system (which managed billions of containers), Kubernetes was open-sourced in 2014 and donated to the CNCF in 2015. It has since become one of the most active open-source projects in history, supported by every major cloud provider (EKS, GKE, AKS) and available on-premises (OpenShift, Rancher, kubeadm).

Kubernetes is not just a tool—it's a platform for building platforms. It provides a declarative API (describe desired state; K8s reconciles to achieve it), a consistent deployment interface across clouds, and an extensible architecture (CRDs, operators, service mesh) that has spawned an entire ecosystem. Understanding its core abstractions—Pods, Deployments, Services, Ingress, ConfigMaps, and Secrets—is essential for any architect working with cloud-native applications.

## Definition

**Kubernetes** is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. It groups containers into Pods, runs Pods on a cluster of Nodes, and continuously reconciles the actual state of the cluster with the desired state declared by the user.

**Key abstractions**:
- **Pod**: The smallest deployable unit—one or more tightly-coupled containers sharing network and storage
- **Node**: A worker machine (VM or physical) that runs Pods
- **Cluster**: A set of Nodes managed by a control plane
- **Deployment**: Declarative management of replicated, stateless Pods
- **Service**: A stable network endpoint for a set of Pods (load-balanced)
- **Ingress**: HTTP/HTTPS routing from outside the cluster to Services inside
- **ConfigMap / Secret**: Configuration and sensitive data injected into Pods
- **Namespace**: Virtual cluster for resource isolation and access control

## Concept Explanation

### Cluster Architecture

```
┌───────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                           │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐           │
│  │ API      │  │ Scheduler│  │ Controller    │           │
│  │ Server   │  │          │  │ Manager       │           │
│  └────┬─────┘  └────┬─────┘  └───────┬───────┘           │
│       │             │               │                     │
│  ┌────┴─────────────┴───────────────┴───────┐            │
│  │              etcd (State Store)          │            │
│  └──────────────────────────────────────────┘            │
└───────────────────────┬───────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Worker Node │ │  Worker Node │ │  Worker Node │
│  ┌────────┐  │ │  ┌────────┐  │ │  ┌────────┐  │
│  │ kubelet│  │ │  │ kubelet│  │ │  │ kubelet│  │
│  └────────┘  │ │  └────────┘  │ │  └────────┘  │
│  ┌────────┐  │ │  ┌────────┐  │ │  ┌────────┐  │
│  │kube-   │  │ │  │kube-   │  │ │  │kube-   │  │
│  │proxy   │  │ │  │proxy   │  │ │  │proxy   │  │
│  └────────┘  │ │  └────────┘  │ │  └────────┘  │
│  ┌──┐┌──┐┌──┐│ │  ┌──┐┌──┐┌──┐│ │  ┌──┐┌──┐┌──┐│
│  │P1││P2││P3││ │  │P4││P5││P6││ │  │P7││P8││P9││
│  └──┘└──┘└──┘│ │  └──┘└──┘└──┘│ │  └──┘└──┘└──┘│
└──────────────┘ └──────────────┘ └──────────────┘
```

### Core Resource Lifecycle

```yaml
# Deployment: manages Pod replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3  # Desired state: 3 Pods
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: order-service:1.2.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"     # 0.25 CPU guaranteed
            memory: "256Mi"
          limits:
            cpu: "500m"     # Max 0.5 CPU (throttled above)
            memory: "512Mi" # OOM killed above
        readinessProbe:     # Is pod ready to serve traffic?
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:      # Should pod be restarted?
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: host
---
# Service: stable network endpoint
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 8080
    targetPort: 8080
---
# Ingress: external HTTP routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-api
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
```

### The Reconciliation Loop

```python
# Kubernetes core concept: reconciliation
# You DECLARE desired state. K8s RECONCILES actual → desired.

class DeploymentController:
    """
    Continuously compares desired state (spec) with actual state (status).
    Takes actions to reconcile differences.
    """
    
    def reconcile(self, deployment):
        desired_replicas = deployment.spec.replicas
        actual_replicas = self.count_healthy_pods(deployment)
        
        if actual_replicas < desired_replicas:
            # Scale up: create Pods
            pods_to_create = desired_replicas - actual_replicas
            for _ in range(pods_to_create):
                self.create_pod(deployment.spec.template)
                self.log(f"Created pod for {deployment.name}")
        
        elif actual_replicas > desired_replicas:
            # Scale down: delete excess Pods
            pods_to_delete = actual_replicas - desired_replicas
            excess_pods = self.get_excess_pods(deployment, pods_to_delete)
            for pod in excess_pods:
                self.delete_pod(pod)
                self.log(f"Deleted excess pod {pod.name}")
        
        # Check pod template changes (rolling update)
        if self.pod_template_changed(deployment):
            self.perform_rolling_update(deployment)
```

### Scheduling

```python
class Scheduler:
    """
    Decides which Node each Pod should run on.
    """
    
    def schedule(self, pod, nodes):
        feasible_nodes = self.filter(pod, nodes)
        if not feasible_nodes:
            return None  # Pod remains Pending
        
        scored_nodes = self.score(pod, feasible_nodes)
        return max(scored_nodes, key=lambda n: n.score)
    
    def filter(self, pod, nodes):
        """Remove nodes that can't run this pod"""
        feasible = []
        for node in nodes:
            if not self.has_sufficient_cpu(node, pod):
                continue
            if not self.has_sufficient_memory(node, pod):
                continue
            if pod.spec.node_selector and not self.matches_labels(node, pod):
                continue
            if pod.spec.affinity and not self.satisfies_affinity(node, pod):
                continue
            feasible.append(node)
        return feasible
    
    def score(self, pod, nodes):
        """Rank remaining nodes by fitness"""
        for node in nodes:
            node.score = 0
            # Prefer nodes with least requested resources (bin packing)
            node.score += self.resource_utilization_score(node)
            # Prefer spreading pods of same service across nodes
            node.score += self.pod_anti_affinity_score(node, pod)
            # Prefer nodes already having the container image cached
            node.score += self.image_locality_score(node, pod.image)
        return nodes
```

## Layman's Explanation

### The Orchestra Conductor
Kubernetes is an orchestra conductor for containers:

- **Pods**: The musicians. Each musician (container) plays one instrument, but sometimes a drummer and a percussionist must sit together (sidecar containers in the same Pod).
- **Deployment**: The sheet music. "Play this piece with 3 violinists at all times." If a violinist leaves (Pod crashes), the conductor immediately finds a replacement.
- **Service**: The music hall's address. Audience members (other services) don't need to know which violinist is playing—they just go to "First Violin Section" (Service DNS name) and the conductor routes them to an available violinist.
- **Scheduler**: The seating arrangement. The conductor decides where each musician sits based on instrument size, needed proximity to other sections, and available space.
- **etcd**: The conductor's score and notes. Every decision, every musician assignment, every change is recorded here.
- **Ingress**: The ticket booth. Audience (external traffic) enters through here and is directed to the right performance (service).

### The Declarative Model
Traditional operations (imperative): "SSH into server 5, start the order service on port 8080, ensure it restarts on failure."

Kubernetes operations (declarative): "I want the order service running with 3 replicas, on port 8080, with 512MB memory each." Kubernetes figures out WHERE to run them, HOW to start them, and MAINTAINS that state forever.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **StatefulSet vs Deployment**: StatefulSets provide stable, unique Pod identities and persistent storage—essential for databases, Kafka, and clustered applications. Deployments are for stateless workloads that are interchangeable. Running a database as a Deployment (without StatefulSet) results in data loss when Pods are rescheduled.
- **Service Mesh vs kube-proxy**: Basic Kubernetes networking (kube-proxy, iptables) provides simple load balancing. For advanced features (mTLS, traffic splitting, circuit breaking, observability), a service mesh (Istio, Linkerd, Consul) running as a sidecar is required. The mesh adds complexity but provides east-west traffic control.
- **Cluster Sizing**: Minimum production cluster: 3 control plane nodes (for etcd quorum) + 3+ worker nodes (for workload resilience). Node pools allow mixing instance types (compute-optimized for app, memory-optimized for caching). Auto-scaling (Cluster Autoscaler, Karpenter) adjusts node count based on pending Pods.
- **Observability Stack**: Kubernetes itself emits nothing useful for debugging. Must deploy: metrics server (for HPA), Prometheus + Grafana (monitoring), Fluentd/Fluent Bit + Loki (logging), Jaeger/Tempo (tracing). The architect must plan the observability infrastructure as part of the Kubernetes deployment.

### Business Impact
- **Multi-Cloud Portability**: Kubernetes provides a consistent API across AWS (EKS), GCP (GKE), Azure (AKS), and on-premises. This enables multi-cloud strategies and reduces vendor lock-in.
- **Developer Self-Service**: With Kubernetes, developers deploy their own services via GitOps (ArgoCD, Flux). No ticket to "the ops team to provision 3 VMs." This directly accelerates time-to-market.
- **Resource Efficiency**: Kubernetes bin-packs containers onto nodes, typically achieving 40-60% higher resource utilization than VM-based deployments. At cloud scale, this translates to millions in savings.

## On-Premises Examples

### kubeadm (Cluster Setup)
```bash
# Control plane node
kubeadm init --pod-network-cidr=10.244.0.0/16

# Worker nodes join
kubeadm join 192.168.1.100:6443 \
  --token abc123.xxx \
  --discovery-token-ca-cert-hash sha256:xxx

# Install CNI (Container Network Interface)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### kubectl Essential Commands
```bash
# View resources
kubectl get pods,services,deployments,ingress -n production

# Describe (detailed state, events)
kubectl describe pod order-service-abc123

# Logs
kubectl logs -f deployment/order-service
kubectl logs order-service-abc123 -c sidecar-container  # Specific container

# Execute in container
kubectl exec -it order-service-abc123 -- /bin/sh

# Port forward (local development)
kubectl port-forward svc/order-service 8080:8080

# Scale
kubectl scale deployment order-service --replicas=5

# Rollout
kubectl rollout restart deployment/order-service
kubectl rollout status deployment/order-service
kubectl rollout undo deployment/order-service  # Rollback

# Resource usage
kubectl top pods -n production
kubectl top nodes
```

### Helm (Package Manager)
```yaml
# Chart.yaml
apiVersion: v2
name: order-service
version: 1.2.0

# values.yaml
replicaCount: 3
image:
  repository: order-service
  tag: "1.2.0"
resources:
  limits:
    cpu: 500m
    memory: 512Mi

# Install
helm install order-service ./order-service-chart \
  --set replicaCount=5 \
  --namespace production

# Upgrade
helm upgrade order-service ./order-service-chart \
  --set image.tag=1.3.0

# Rollback
helm rollback order-service 1
```

## AWS Examples: See EKS/blog.md

## GCP Examples: See GKE/blog.md

## Azure Examples: See AKS/blog.md

## On-Prem Alternatives: See OnPrem-Alternatives/blog.md

## Summary

| Concept | Purpose |
|---------|---------|
| Pod | Smallest deployable unit (one or more containers) |
| Deployment | Manages stateless, replicated Pods |
| StatefulSet | Manages stateful Pods with stable identities |
| Service | Stable network endpoint, internal load balancing |
| Ingress | HTTP routing from outside cluster |
| ConfigMap | Non-sensitive configuration |
| Secret | Sensitive data (passwords, tokens) |
| Namespace | Resource isolation and multi-tenancy |

Kubernetes is the operating system of the cloud—it provides a universal API for deploying, scaling, and managing containerized workloads across any infrastructure. Its power comes from the declarative model: you describe the desired state, and Kubernetes continuously reconciles reality to match it. The architect's job is to define the right abstractions (Deployments, Services, Ingresses), configure the right operational parameters (resource limits, health checks, auto-scaling), and ensure the supporting ecosystem (monitoring, logging, security) is in place before production workloads go live.
