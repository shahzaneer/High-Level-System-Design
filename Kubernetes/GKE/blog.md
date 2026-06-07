# Google Kubernetes Engine (GKE)

## Introduction
Google Kubernetes Engine (GKE) is the original managed Kubernetes service—launched in 2015, two years before AWS EKS and Azure AKS existed. This first-mover advantage reflects Google's deep Kubernetes expertise: Google invented Kubernetes based on its internal Borg system, and Google engineers continue to be the largest contributors to the Kubernetes open-source project.

GKE differentiates itself through deep integration with Google's networking infrastructure (Cloud Load Balancing, VPC-native networking), advanced cluster management features (Autopilot mode, release channels), and Google-native services (Cloud Monitoring, Cloud Logging, Workload Identity). It is widely considered the most mature and feature-rich managed Kubernetes offering.

## Concept Explanation

### GKE Modes

```
GKE Standard:
  - You manage node pools (instance types, sizing, scaling)
  - Full control over cluster configuration
  - Pay for nodes (Compute Engine pricing)
  - Management fee: $0.10/cluster/hour (free for first cluster)

GKE Autopilot:
  - Google manages nodes (provisions, scales, patches)
  - Pay per Pod (vCPU, memory, storage)
  - Management fee included in Pod pricing
  - No node management at all
  - Automatic security hardening, node auto-upgrade
  - Best for: teams wanting "serverless Kubernetes"
```

### Cluster Provisioning

```bash
# GKE Standard cluster
gcloud container clusters create production-cluster \
  --region=us-central1 \
  --num-nodes=1 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=20 \
  --machine-type=e2-standard-4 \
  --release-channel=regular \
  --workload-pool=my-project.svc.id.goog \
  --enable-master-authorized-networks \
  --master-authorized-networks=10.0.0.0/8 \
  --enable-private-nodes \
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-ip-alias \
  --cluster-ipv4-cidr=/14 \
  --services-ipv4-cidr=/20 \
  --enable-dataplane-v2 \
  --enable-shielded-nodes

# GKE Autopilot cluster
gcloud container clusters create-auto production-autopilot \
  --region=us-central1 \
  --release-channel=regular
```

### Release Channels

```
Rapid:     Latest K8s features, weekly updates
           → For dev/staging, feature testing
Regular:   Stable K8s versions, monthly updates
           → Default for most workloads
Stable:    Most mature versions, quarterly updates
           → For critical production workloads

gcloud container clusters create production-cluster \
  --release-channel=regular
```

### Workload Identity (Pod → GCP Service Account)

```bash
# Create GCP service account
gcloud iam service-accounts create gke-order-processor

# Grant permissions
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:gke-order-processor@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Bind K8s service account to GCP service account
gcloud iam service-accounts add-iam-policy-binding \
  gke-order-processor@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project.svc.id.goog[production/order-processor]"
```

```yaml
# Kubernetes service account with Workload Identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-processor
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: gke-order-processor@my-project.iam.gserviceaccount.com
---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: order-processor
  containers:
  - name: app
    image: order-processor:latest
    # Automatically uses GCP service account credentials
    # No service account keys needed
```

### Networking: Dataplane V2

```
GKE Dataplane V2 (eBPF-based):
  - Replaces iptables-based kube-proxy with eBPF
  - Native Network Policy enforcement (no Calico needed)
  - Better performance (fewer rules to evaluate)
  - Built-in network observability
  
  Enable: --enable-dataplane-v2

VPC-native networking:
  - Pods get IPs from GCP VPC subnets (alias IP ranges)
  - Pod IPs are routable within the VPC
  - No overlay networking overhead
```

### Cloud Load Balancing Integration

```yaml
# Ingress → Global HTTPS Load Balancer
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-api
  annotations:
    kubernetes.io/ingress.class: gce
    kubernetes.io/ingress.global-static-ip-name: order-api-ip
    networking.gke.io/managed-certificates: order-api-cert
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
---
# Managed Certificate (auto-provisions + auto-renews)
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: order-api-cert
spec:
  domains:
    - api.example.com
---
# Container-native load balancing (direct Pod targeting)
apiVersion: v1
kind: Service
metadata:
  name: order-service
  annotations:
    cloud.google.com/neg: '{"exposed_ports": {"8080": {}}}'
spec:
  type: ClusterIP
  ports:
  - port: 8080
# Network Endpoint Group (NEG): Load balancer routes directly to Pod IPs
# No iptables DNAT, no NodePort, lower latency
```

### GKE Autoscaling

```yaml
# Horizontal Pod Autoscaler
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
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: External
    external:
      metric:
        name: loadbalancing.googleapis.com|https|request_count
        selector:
          matchLabels:
            resource.labels.forwarding_rule_name: order-api
      target:
        type: AverageValue
        averageValue: "100"
---
# Vertical Pod Autoscaler (adjusts requests/limits automatically)
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
```

### GKE Gateway (Next-Gen Ingress)

```yaml
# Gateway API: more expressive than Ingress
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: external-http
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: order-routes
spec:
  parentRefs:
  - name: external-http
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/orders
    backendRefs:
    - name: order-service
      port: 8080
```

### Monitoring & Logging

```bash
# GKE automatically integrates with Cloud Operations
# Enabled by default (can be disabled)

# Cloud Monitoring: System metrics, workload metrics, SLOs
gcloud monitoring dashboards list

# Cloud Logging: All container stdout/stderr + audit logs
gcloud logging read "resource.type=k8s_container AND resource.labels.cluster_name=production-cluster"

# Managed Prometheus
gcloud container clusters update production-cluster \
  --enable-managed-prometheus

# Managed Grafana
gcloud monitoring grafana create
```

### Security Features

```bash
# Binary Authorization: only allow verified images
gcloud container clusters update production-cluster \
  --enable-binary-authorization

# Shielded GKE Nodes: hardened node images
# (enabled by default for new clusters)

# Workload Identity: Pod → GCP IAM mapping (no service account keys)

# GKE Sandbox (gVisor): Pod-level kernel isolation
apiVersion: v1
kind: Pod
spec:
  runtimeClassName: gvisor  # Additional isolation layer

# Confidential GKE Nodes: encrypt data in-use
gcloud container clusters create production-cluster \
  --enable-confidential-nodes
```

### Cost Optimization

```bash
# Committed Use Discounts (1-year or 3-year)
# Apply to GKE nodes running on Compute Engine

# Spot Pods (preemptible): up to 91% discount
apiVersion: v1
kind: Pod
spec:
  nodeSelector:
    cloud.google.com/gke-spot: "true"
  terminationGracePeriodSeconds: 25  # Handle preemption gracefully

# Autopilot: pay per Pod, no idle node cost
# Good for variable workloads, dev/staging
```

## Layman's Explanation

GKE is the native son of Kubernetes—Google built the original, runs billions of containers on it internally, and offers the most mature managed version. GKE Autopilot is particularly noteworthy: it's Kubernetes without the server management. You write YAML (your "sheet music"), and Google handles everything else—the instruments (nodes), tuning (patching), and scaling. You get a Kubernetes API without any Kubernetes operations burden.

## Why Solution Architects Must Acquire This

### GKE-Specific Decisions
- **Standard vs Autopilot**: Autopilot eliminates node management but costs ~20-30% more per Pod and has fewer customization options. Standard mode gives full control at lower cost but requires node management. Choose Autopilot for teams without dedicated platform/SRE; Standard for teams that want node-level optimizations.
- **Dataplane V2**: Should be the default for new clusters. It provides better network performance, native Network Policy support, and built-in observability. Only use legacy kube-proxy mode if you have specific compatibility requirements.
- **Gateway API vs Ingress**: GKE is at the forefront of the Gateway API transition. Gateway API provides more expressive routing (header-based, weight-based), better role separation, and is the future direction of Kubernetes networking. Use Gateway API for new deployments.

## Summary

| GKE Feature | Benefit |
|------------|---------|
| Autopilot Mode | Serverless Kubernetes, no node management |
| Release Channels | Curated, tested version upgrades |
| Workload Identity | Pods get GCP IAM without static keys |
| Dataplane V2 | eBPF-based networking, native NetworkPolicy |
| Managed Prometheus | Google-managed Prometheus (no server ops) |
| Binary Authorization | Only signed/verified images run |
| Gateway API | Next-gen HTTP routing, multi-tenancy |
| NEG (Container-Native LB) | Load balancer → Pod directly, no NodePort |

GKE is the most mature managed Kubernetes service, reflecting Google's deep K8s expertise. It excels at "just works" Kubernetes with Autopilot mode eliminating node management entirely. The deep integration with Google Cloud services—Workload Identity, Cloud Monitoring, Cloud Logging, Certificate Manager—makes it the natural Kubernetes platform for GCP-native deployments. The combination of Autopilot + managed services enables teams to run production Kubernetes without a dedicated platform engineering team.
