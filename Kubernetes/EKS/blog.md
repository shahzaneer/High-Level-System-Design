# Amazon EKS (Elastic Kubernetes Service)

## Introduction
Amazon EKS is AWS's managed Kubernetes service, launched in 2018 after customer demand for a native AWS Kubernetes offering. Unlike self-managed Kubernetes where you run the control plane, EKS manages the control plane (API server, etcd, scheduler, controller manager) across multiple Availability Zones for high availability. You manage the worker nodes and the workloads running on them.

EKS is deeply integrated with the AWS ecosystem: IAM for authentication (no separate Kubernetes RBAC user management), VPC for networking (Pods get VPC IPs via the AWS VPC CNI), ELB for load balancing (Services automatically provision ALBs/NLBs), and ECR for container images. This integration makes EKS the natural choice for AWS-native Kubernetes deployments.

## Concept Explanation

### EKS Architecture

```
┌──────────────────────────────────────────────────────────┐
│                 AWS MANAGED CONTROL PLANE                 │
│  ┌────────────────────────────────────────────────────┐  │
│  │  API Server │ Scheduler │ Controller Mgr │ etcd    │  │
│  │  (across 3 AZs for HA)                           │  │
│  └──────────────────┬───────────────────────────────┘  │
└─────────────────────┼────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Node Group │ │   Node Group │ │   Node Group │
│   (us-east-1a)│ │  (us-east-1b)│ │  (us-east-1c)│
│              │ │              │ │              │
│  ┌──┐┌──┐┌──┐│ │  ┌──┐┌──┐┌──┐│ │  ┌──┐┌──┐┌──┐│
│  │P ││P ││P ││ │  │P ││P ││P ││ │  │P ││P ││P ││
│  └──┘└──┘└──┘│ │  └──┘└──┘└──┘│ │  └──┘└──┘└──┘│
└──────────────┘ └──────────────┘ └──────────────┘
```

### Cluster Provisioning

```hcl
# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "production-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.30"

  vpc_config {
    subnet_ids = concat(
      aws_subnet.private[*].id,  # Pod subnets
      aws_subnet.public[*].id    # Control plane endpoints
    )
    endpoint_private_access = true   # Accessible within VPC
    endpoint_public_access  = true   # Accessible from internet (with auth)
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator", 
                                "controllerManager", "scheduler"]

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy
  ]
}

# Managed Node Group
resource "aws_eks_node_group" "app" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "app-nodes"
  node_role_arn   = aws_iam_role.eks_node.arn
  subnet_ids      = aws_subnet.private[*].id

  instance_types = ["t4g.large", "t4g.xlarge"]

  scaling_config {
    desired_size = 3
    max_size     = 10
    min_size     = 1
  }

  update_config {
    max_unavailable = 1  # Rolling update: 1 node at a time
  }

  labels = {
    workload = "application"
  }

  taint {
    key    = "dedicated"
    value  = "application"
    effect = "NO_SCHEDULE"
  }
}

# Fargate Profile (serverless Pods, no node management)
resource "aws_eks_fargate_profile" "batch" {
  cluster_name           = aws_eks_cluster.main.name
  fargate_profile_name   = "batch-jobs"
  pod_execution_role_arn = aws_iam_role.fargate_execution.arn
  subnet_ids             = aws_subnet.private[*].id

  selector {
    namespace = "batch"
  }
}
```

### IAM Integration (IRSA - IAM Roles for Service Accounts)

```yaml
# Service Account with IAM role
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-processor
  namespace: production
  annotations:
    # Map K8s service account to IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-processor-role
---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: order-processor
  containers:
  - name: app
    # AWS SDK automatically uses IAM role from service account
    # No access keys, no secrets—pod identity
```

### AWS Load Balancer Controller

```yaml
# Ingress → Application Load Balancer (ALB)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-api
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
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
# Service type LoadBalancer → Network Load Balancer (NLB)
apiVersion: v1
kind: Service
metadata:
  name: order-service-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internal
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 8080
  selector:
    app: order-service
```

### Networking: AWS VPC CNI

```
AWS VPC CNI gives every Pod a VPC IP address (not an overlay network IP).

Benefits:
- Pods communicate using VPC IPs (Security Groups, NACLs work on Pods)
- Application Load Balancer can target Pods directly (no NodePort)
- VPC Flow Logs show Pod-to-Pod traffic

Trade-offs:
- Pod IPs consume VPC IP addresses (plan your subnet sizing!)
- Pod density limited by available IPs on the ENI (t3.medium = 12 Pods max default)
- Need to configure prefix delegation for higher Pod density
```

```bash
# Enable prefix delegation for more Pods per node
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
```

### EKS Add-ons

```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "vpc-cni"
  addon_version = "v1.16.0"
}

resource "aws_eks_addon" "coredns" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "coredns"
}

resource "aws_eks_addon" "kube_proxy" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "kube-proxy"
}

resource "aws_eks_addon" "ebs_csi" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "aws-ebs-csi-driver"
  service_account_role_arn = aws_iam_role.ebs_csi.arn
}
```

### Monitoring Integration

```bash
# CloudWatch Container Insights
aws eks create-addon \
  --cluster-name production-cluster \
  --addon-name amazon-cloudwatch-observability

# Managed Prometheus
aws eks create-addon \
  --cluster-name production-cluster \
  --addon-name amazon-cloudwatch-metrics

# ADOT (AWS Distro for OpenTelemetry)
aws eks create-addon \
  --cluster-name production-cluster \
  --addon-name adot
```

### Security Best Practices

```yaml
# Pod Security Standards (PSS) - restricted profile
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Network Policy: restrict egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: order-database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### Secrets Management

```yaml
# AWS Secrets Manager integration via Secrets Store CSI Driver
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: order-service-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/database/credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: "username"
            objectAlias: "DB_USERNAME"
          - path: "password"
            objectAlias: "DB_PASSWORD"
---
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: secrets
      mountPath: /mnt/secrets
      readOnly: true
  volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: order-service-secrets
```

### Cluster Auto-Scaling (Karpenter)

```yaml
# Karpenter: faster, more flexible than Cluster Autoscaler
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        billing: standard
    spec:
      nodeClassRef:
        name: default
      requirements:
        - key: "karpenter.k8s.aws/instance-category"
          operator: In
          values: ["c", "m", "r", "t"]
        - key: "karpenter.k8s.aws/instance-cpu"
          operator: In
          values: ["2", "4", "8"]
        - key: "topology.kubernetes.io/zone"
          operator: In
          values: ["us-east-1a", "us-east-1b", "us-east-1c"]
        - key: "kubernetes.io/arch"
          operator: In
          values: ["amd64", "arm64"]
      limits:
        cpu: "100"
      disruption:
        consolidationPolicy: WhenUnderutilized
```

## Layman's Explanation

EKS is like having a professional orchestra conductor (AWS) manage the conductor's job (control plane)—the baton, the score, the podium—while you bring the musicians (worker nodes) and sheet music (workloads). The conductor ensures the performance runs smoothly, handles musician absences (auto-healing), and can call in more musicians when the music demands it (auto-scaling). You focus on the music (your application); AWS focuses on conducting.

## Why Solution Architects Must Acquire This

### EKS-Specific Decisions
- **Networking Mode**: AWS VPC CNI (Pod gets real VPC IP) vs Calico (overlay network). VPC CNI integrates with AWS networking services (Security Groups, VPC Flow Logs) but creates IP consumption concerns. Calico provides more fine-grained network policies but adds operational complexity.
- **Node Management**: Managed node groups (AWS handles AMI updates, scaling) vs self-managed (more control, more ops) vs Karpenter (dynamic provisioning, best for variable workloads) vs Fargate (serverless, per-Pod pricing, no node management). Most production clusters use a mix.
- **Authentication**: EKS uses AWS IAM for authentication (`aws eks update-kubeconfig` authenticates via IAM). Kubernetes RBAC for authorization. This means no static kubeconfig credentials; access is tied to IAM roles and audited via CloudTrail.

## Summary

| EKS Feature | Benefit |
|------------|---------|
| Managed Control Plane | No etcd management, auto-scaling, multi-AZ HA |
| IAM Integration | No static credentials, IAM → RBAC mapping |
| VPC CNI | Native AWS networking, Security Groups on Pods |
| Fargate | Serverless Pods, per-use pricing |
| IRSA | Pods get IAM roles without static credentials |
| Managed Add-ons | VPC CNI, CoreDNS, kube-proxy, EBS CSI managed by AWS |

EKS provides the Kubernetes control plane as a managed service while deeply integrating with the AWS ecosystem. The architect's primary decisions revolve around networking (VPC CNI vs alternatives), node management (managed node groups vs Karpenter vs Fargate), and AWS service integration (IAM, load balancing, secrets, monitoring). EKS is the default Kubernetes platform for AWS-native deployments, offering the best integration with the surrounding AWS service ecosystem.
