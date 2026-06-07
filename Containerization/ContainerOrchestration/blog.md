# Container Orchestration

## Introduction
Container orchestration is the automated management of containerized applications at scale. While Docker simplifies running a single container on a single machine, container orchestration platforms (Kubernetes, Docker Swarm, Nomad, ECS) solve the multi-host, multi-container problem: scheduling containers across a cluster of machines, maintaining desired replica counts, routing traffic, handling failures, scaling, and managing configuration.

The need for orchestration emerges naturally as container deployments grow. A single developer can manage 5 containers manually. 50 containers need basic automation. 500 containers require orchestration. 5,000+ containers require a mature orchestration platform with advanced scheduling, auto-scaling, service mesh, and GitOps workflows. Kubernetes is the industry standard, but understanding the alternatives helps architects choose the right tool for the right context.

## Concept Overview

### Orchestration Responsibilities

```
SCHEDULING:
  Which node should run which container?
  → Resource requirements, affinity/anti-affinity, taints/tolerations

SERVICE DISCOVERY:
  How do containers find each other?
  → DNS, environment variables, service mesh

LOAD BALANCING:
  How is traffic distributed?
  → Internal (ClusterIP), External (LoadBalancer, Ingress)

SELF-HEALING:
  What happens when a container crashes?
  → Restart, reschedule on different node

SCALING:
  How to handle more load?
  → Horizontal Pod Autoscaler, Cluster Autoscaler

CONFIGURATION:
  How to manage settings across environments?
  → ConfigMaps, Secrets, environment variables

ROLLING UPDATES:
  How to deploy new versions?
  → Rolling, Blue/Green, Canary

STORAGE:
  How to persist data?
  → PersistentVolumes, CSI drivers, StatefulSets
```

### Orchestrator Comparison

```
┌──────────────────────────────────────────────────────────────────┐
│                   ORCHESTRATION PLATFORMS                         │
├─────────────┬──────────┬──────────────┬──────────┬──────────────┤
│ Kubernetes  │ Docker   │ HashiCorp    │ AWS ECS  │ Azure        │
│             │ Swarm    │ Nomad        │          │ Container    │
│             │          │              │          │ Apps         │
├─────────────┼──────────┼──────────────┼──────────┼──────────────┤
│ Best for:   │ Best for:│ Best for:    │ Best for:│ Best for:    │
│ Complex     │ Simple   │ Mixed        │ AWS-     │ Serverless   │
│ microserv-  │ setups,  │ workloads    │ native,  │ containers   │
│ ices, multi │ smaller  │ (containers  │ simpli-  │ on Azure     │
│ -cloud      │ clusters │ + VMs +      │ city     │              │
│             │          │ batch)       │          │              │
├─────────────┼──────────┼──────────────┼──────────┼──────────────┤
│ Complexity: │ Lowest   │ Moderate     │ Low      │ Very Low     │
│ Very High   │          │              │          │ (managed)    │
└─────────────┴──────────┴──────────────┴──────────┴──────────────┘
```

### When NOT to Use Kubernetes

```
Use simpler alternatives when:

- You have < 5 services → Docker Compose or ECS Fargate
- You have no dedicated platform team → ECS Fargate, Cloud Run, ACI
- Your workloads are mostly batch → Nomad or AWS Batch
- You have simple scaling needs → App Service, Cloud Run
- Kubernetes is trendy but your team isn't ready to operate it

Operations Reality:
  Kubernetes requires: CNI, CSI, Ingress Controller, cert-manager,
  external-dns, monitoring stack, logging stack, tracing stack,
  security policies, backup solution, GitOps tool...

  That's 10+ additional components beyond Kubernetes itself.
  Managed K8s (EKS/GKE/AKS) reduces but doesn't eliminate this.
```

### Nomad (HashiCorp) - Simpler Alternative

```hcl
# Nomad: single binary, orchestrates containers + VMs + batch
job "order-service" {
  datacenters = ["dc1"]
  type = "service"

  group "server" {
    count = 3

    network {
      port "http" { to = 8080 }
    }

    service {
      name = "order-service"
      port = "http"
    }

    task "server" {
      driver = "docker"

      config {
        image = "order-service:1.0.0"
        ports = ["http"]
      }

      resources {
        cpu    = 500
        memory = 256
      }
    }
  }
}
```

### Service Discovery Patterns

```yaml
# Pattern 1: DNS-based (Kubernetes default)
# order-service.production.svc.cluster.local → ClusterIP → Pod IPs

# Pattern 2: Client-side (Consul, Eureka)
# Application queries service registry, gets healthy endpoint list
consul_client.catalog.service("order-service")

# Pattern 3: Proxy-side (Service Mesh)
# Envoy/Linkerd sidecar handles discovery transparently
# Application calls localhost:8080 → sidecar routes to healthy backend

# Pattern 4: Cloud-native (Cloud Map, Service Directory)
aws servicediscovery discover-instances \
  --service-name order-service \
  --namespace-name production
```

### Scheduling Strategies

```yaml
# Spread across AZs (fault tolerance)
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        topologyKey: topology.kubernetes.io/zone
        labelSelector:
          matchLabels:
            app: order-service

# Co-locate services that communicate (latency)
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - topologyKey: kubernetes.io/hostname
      labelSelector:
        matchLabels:
          app: redis-cache

# Dedicated nodes (taints/tolerations)
tolerations:
- key: "workload"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

## AWS (ECS), GCP (Cloud Run), Azure (Container Apps)

### ECS (Elastic Container Service)
```hcl
resource "aws_ecs_service" "order_service" {
  name            = "order-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.order.arn
  desired_count   = 3
  launch_type     = "FARGATE"  # Serverless, no EC2 management

  network_configuration {
    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.order.id]
  }
}
```

### Cloud Run
```bash
gcloud run deploy order-service \
  --image gcr.io/my-project/order-service \
  --platform managed \
  --concurrency 80 \
  --max-instances 10
```

### Azure Container Apps
```bash
az containerapp create \
  --name order-service \
  --environment managedEnvironment \
  --image registry.azurecr.io/order-service:v1 \
  --target-port 8080 \
  --min-replicas 1 --max-replicas 10
```

## Summary

| Orchestrator | Complexity | Best For |
|-------------|-----------|----------|
| Kubernetes | Very High | Complex microservices, multi-cloud, large teams |
| Nomad | Medium | Mixed workloads, simpler than K8s, HashiCorp ecosystem |
| Docker Swarm | Low | Small deployments, simple setups |
| ECS Fargate | Low | AWS-native, serverless containers |
| Cloud Run | Very Low | GCP-native, serverless, HTTP workloads |
| Azure Container Apps | Very Low | Azure-native, serverless, KEDA-based scaling |

Container orchestration is about managing complexity at scale. Kubernetes is the industry standard for complex, multi-service architectures with dedicated platform teams. Simpler alternatives (ECS Fargate, Cloud Run, Nomad) are appropriate when the organization doesn't have the operational maturity for Kubernetes or when the workload doesn't justify the complexity. The architect's most important orchestration decision is choosing the right level of abstraction—not automatically defaulting to Kubernetes because it's the dominant solution.
