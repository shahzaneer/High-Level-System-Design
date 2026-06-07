# Containers: Core Concepts

## Introduction
Containers have transformed how applications are built, shipped, and run. While operating system virtualization (chroot, FreeBSD jails, Solaris Zones) existed for decades, Docker's 2013 release made containers accessible to mainstream developers by combining Linux kernel features (namespaces, cgroups) with a simple CLI and a portable image format. The paradigm shift was profound: "works on my machine" became "works everywhere the container runtime runs."

Today, containers are the default deployment artifact for cloud-native applications. The Open Container Initiative (OCI) standardized the image format and runtime specification, enabling an ecosystem of interoperable tools (Docker, Podman, containerd, CRI-O). Understanding containers—what they are, how they work, and how to build them securely—is foundational knowledge for every solution architect.

## Definition

A **container** is a lightweight, standalone, executable package that includes everything needed to run a piece of software: code, runtime, system tools, libraries, and settings. Containers isolate applications from each other and from the host system while sharing the host's operating system kernel.

**Key distinctions**:
- **Container ≠ VM**: VMs virtualize hardware (each has its own OS kernel). Containers virtualize the OS (they share the host kernel). This makes containers much lighter—they start in milliseconds vs minutes, and a host can run hundreds of containers vs tens of VMs.
- **Image**: A read-only template with instructions for creating a container. Built from a Dockerfile.
- **Container**: A runnable instance of an image.
- **Registry**: A repository for storing and distributing images (Docker Hub, ECR, GCR, ACR).

## Concept Explanation

### How Containers Work (Linux Kernel Features)

```
CONTAINER = NAMESPACES + CGROUPS + UNION FILESYSTEM

NAMESPACES (Isolation - "What you can see"):
  PID namespace:     Container sees only its own processes (PID 1 inside = some PID on host)
  NET namespace:     Container has its own network stack, IP address, ports
  MNT namespace:     Container has its own filesystem mount points
  UTS namespace:     Container has its own hostname
  IPC namespace:     Container has its own inter-process communication
  USER namespace:    Container can have its own users (root in container ≠ root on host)

CGROUPS (Resource Limits - "What you can use"):
  CPU:      Limit container to 2 CPU cores
  Memory:   Limit container to 512MB RAM
  Disk I/O: Limit container to 100 MB/s
  Network:  Limit container bandwidth

UNION FILESYSTEM (Image Layers):
  Each instruction in a Dockerfile creates a layer.
  Layers are stacked; the container sees a unified view.
  Layers are shared between containers (copy-on-write).
```

### Container vs VM Architecture

```
VIRTUAL MACHINES:                          CONTAINERS:
┌──────────┐ ┌──────────┐                ┌──────────┐ ┌──────────┐
│  App A   │ │  App B   │                │  App A   │ │  App B   │
├──────────┤ ├──────────┤                ├──────────┤ ├──────────┤
│  Bins/   │ │  Bins/   │                │  Bins/   │ │  Bins/   │
│  Libs    │ │  Libs    │                │  Libs    │ │  Libs    │
├──────────┤ ├──────────┤                ├──────────┤ ├──────────┤
│ Guest OS │ │ Guest OS │                │Container │ │Container │
├──────────┤ ├──────────┤                │ Runtime  │ │ Runtime  │
│Hypervisor│ │Hypervisor│                ├──────────┴─┴──────────┤
├──────────┴─┴──────────┤                │      Host OS          │
│       Host OS          │                ├───────────────────────┤
├────────────────────────┤                │     Infrastructure    │
│    Infrastructure      │                └───────────────────────┘
└────────────────────────┘

Size: VM = GBs, Container = MBs
Startup: VM = minutes, Container = milliseconds
Density: 10s of VMs per host, 100s of containers per host
```

### Dockerfile Best Practices

```dockerfile
# BAD: Large image, no caching, security issues
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y python3 python3-pip
COPY . /app
RUN pip install -r requirements.txt
CMD ["python3", "app.py"]

# Size: ~400MB. Layers not optimized. Runs as root.

# GOOD: Multi-stage build, minimal base, non-root user
# Stage 1: Build
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Runtime (minimal)
FROM python:3.12-slim
WORKDIR /app
# Copy only the dependencies from builder (not build tools)
COPY --from=builder /root/.local /root/.local
COPY app.py .
# Run as non-root user
RUN useradd --create-home appuser
USER appuser
# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1
EXPOSE 8080
CMD ["python", "app.py"]

# Size: ~150MB. Optimized layers. Non-root user. Health check.
```

### Layer Caching Strategy

```
Docker builds are layer-by-layer. Each instruction = one layer.
Layers are cached. Order matters for build speed.

OPTIMAL ORDER:
  1. Base image        (rarely changes)
  2. System packages   (rarely changes)
  3. Dependencies file (changes occasionally)
  4. Dependencies install (expensive, but cached if file unchanged)
  5. Application code  (changes frequently)
  6. Configuration     (changes per environment)

# Copy requirements.txt FIRST (before app code)
COPY requirements.txt .
RUN pip install -r requirements.txt  # Cached if requirements.txt unchanged

# Copy app code LAST (changes most frequently)
COPY . .
```

### Container Networking

```yaml
# Docker networking modes
Bridge (default):
  - Containers on same bridge network can communicate
  - Isolated from host network
  - Port mapping needed for external access: -p 8080:80

Host:
  - Container shares host network namespace
  - No isolation; container port = host port
  - Fastest networking (no bridge overhead)

Overlay:
  - Multi-host networking (Docker Swarm / Kubernetes)
  - Containers on different hosts communicate via VXLAN

None:
  - Container has no network (only loopback)
  - For batch jobs or security isolation
```

### Container Storage

```yaml
# Storage types
Volumes:
  - Managed by Docker (/var/lib/docker/volumes/)
  - Persistent across container restarts
  - Can be shared between containers
  docker run -v myvolume:/data myapp

Bind Mounts:
  - Mount host directory into container
  - Development workflow (code changes immediately reflected)
  docker run -v /host/path:/container/path myapp

tmpfs:
  - Temporary, in-memory filesystem
  - Never persisted to disk
  - Good for secrets, temporary data
  docker run --tmpfs /tmp myapp
```

## Layman's Explanation

### The Shipping Container Analogy
Containers (software) are named after shipping containers (physical) for good reason:

Before shipping containers (before Docker), every piece of cargo (application) was loaded onto ships differently. A box of electronics needed different handling than a crate of bananas. Each ship (server) had to be configured specifically for its cargo. Moving cargo between ships was a nightmare.

After shipping containers (after Docker): Every piece of cargo goes into a standardized 20-foot or 40-foot container. The container has standardized corners for cranes, standardized dimensions for stacking. The ship, the truck, and the train car all handle the SAME container format. The port doesn't need to know what's inside.

- **Image**: The blueprints for the container (dimensions, weight limits, what goes inside)
- **Container**: The actual loaded container on a ship (running instance)
- **Registry**: The container depot where you store and retrieve containers (Docker Hub)
- **Orchestrator (Kubernetes)**: The port authority that decides which container goes on which ship, balances the load, and replaces containers that fall overboard

### Why Not Just a VM?
A VM is like chartering an entire ship for your one container. Yes, it gets there, but you're paying for a whole ship, and loading/unloading takes forever. Docker is standardized containers on a shared ship—hundreds of containers, one ship, much cheaper and faster.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Base Image Selection**: Alpine (5MB, minimal, musl libc) vs Debian-slim (80MB, glibc, wider compatibility) vs Distroless (no shell, no package manager, ultra-secure). Alpine is lightest but musl can cause compatibility issues with some Python/C libraries. Distroless is most secure but impossible to debug interactively.
- **Multi-Stage Builds**: Separate build dependencies (compilers, headers, dev packages) from runtime dependencies. The build image can be 800MB with all build tools; the runtime image is 100MB with only what's needed to execute. This dramatically improves security (fewer attack vectors) and reduces image pull time.
- **Container Orchestration Platform**: Single-node Docker Compose for development. Managed container service (ECS Fargate, Cloud Run, ACI) for simple production. Kubernetes for complex multi-service, multi-team production. The jump from Docker Compose to Kubernetes is massive and should be justified by need, not trend.
- **Stateful vs Stateless Containers**: Containers are designed for stateless workloads (they can be killed and replaced anytime). Stateful containers (databases, message queues) require persistent volumes, careful scheduling, and backup strategies. Run databases on Kubernetes only with operators that handle these concerns.

### Business Impact
- **Consistency**: The same container image that developers test on their laptops is deployed to staging and production. No more "it works on my machine" deployment failures. This directly reduces deployment-related incidents.
- **Density and Cost**: A physical server that ran 5 VMs can run 50+ containers. This is a 10x improvement in infrastructure utilization, directly reducing cloud or data center costs.
- **CI/CD Velocity**: Container images are the universal deployment artifact. Build once (in CI), deploy everywhere (dev → staging → prod). This enables GitOps workflows, canary deployments, and instant rollbacks.

## On-Premises Examples

### Docker CLI
```bash
# Build image
docker build -t order-service:1.0.0 .

# Run container
docker run -d \
  --name order-service \
  -p 8080:8080 \
  -e DATABASE_URL=postgresql://db:5432/orders \
  --memory=512m \
  --cpus=2 \
  --restart=unless-stopped \
  order-service:1.0.0

# View logs
docker logs -f order-service

# Execute command in running container
docker exec -it order-service /bin/bash
```

### Docker Compose (Multi-Container Development)
```yaml
version: "3.8"
services:
  order-service:
    build: ./order-service
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://order-db:5432/orders
    depends_on:
      order-db:
        condition: service_healthy
    
  order-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_PASSWORD: dev_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s

volumes:
  pgdata:
```

### Podman (Daemonless Alternative)
```bash
# Podman: Docker-compatible CLI, no daemon, rootless by default
podman build -t order-service .
podman run -d --name order-service order-service
podman generate kube order-service > deployment.yaml  # Export to K8s
```

## AWS Examples

### ECR (Elastic Container Registry)
```bash
# Create repository
aws ecr create-repository --repository-name order-service

# Login
aws ecr get-login-password | docker login --username AWS \
  --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Push image
docker tag order-service:1.0.0 $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/order-service:1.0.0
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/order-service:1.0.0

# Vulnerability scanning (automatic on push)
aws ecr describe-image-scan-findings \
  --repository-name order-service \
  --image-id imageTag=1.0.0
```

### ECS Fargate
```hcl
resource "aws_ecs_task_definition" "order_service" {
  family                   = "order-service"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"    # 0.5 vCPU
  memory                   = "1024"   # 1 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = "order-service"
    image = "${aws_ecr_repository.order_service.repository_url}:1.0.0"
    portMappings = [{ containerPort = 8080 }]
    
    environment = [
      { name = "ENVIRONMENT", value = "production" }
    ]
    
    secrets = [
      { name = "DB_PASSWORD", valueFrom = aws_secretsmanager_secret.db.arn }
    ]
    
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = aws_cloudwatch_log_group.order_service.name
        awslogs-region        = "us-east-1"
        awslogs-stream-prefix = "ecs"
      }
    }
    
    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
    }
  }])
}
```

## GCP Examples

### Cloud Run (Serverless Containers)
```bash
# Build with Cloud Build
gcloud builds submit --tag gcr.io/my-project/order-service

# Deploy to Cloud Run
gcloud run deploy order-service \
  --image gcr.io/my-project/order-service \
  --platform managed \
  --region us-central1 \
  --memory 512Mi \
  --cpu 1 \
  --max-instances 10 \
  --allow-unauthenticated
```

### Artifact Registry
```bash
gcloud artifacts repositories create containers \
  --repository-format=docker \
  --location=us-central1

docker tag order-service us-central1-docker.pkg.dev/my-project/containers/order-service:v1
docker push us-central1-docker.pkg.dev/my-project/containers/order-service:v1
```

## Azure Examples

### Azure Container Registry (ACR)
```bash
az acr create --name myacrregistry --resource-group myRG --sku Standard

az acr build --registry myacrregistry --image order-service:v1 .
```

### Azure Container Instances (ACI)
```bash
az container create \
  --name order-service \
  --image myacrregistry.azurecr.io/order-service:v1 \
  --cpu 1 --memory 1.5 \
  --port 8080 \
  --environment-variables ENVIRONMENT=production
```

## Summary

| Container Technology | Best For |
|---------------------|----------|
| Docker | Development, CI/CD, standard container format |
| Docker Compose | Local development, multi-container testing |
| containerd / CRI-O | Production Kubernetes runtime |
| Podman | Daemonless, rootless containers |
| ECS Fargate | AWS-managed serverless containers |
| Cloud Run | GCP-managed serverless containers |
| ACI | Azure serverless containers |

Containers are the universal application packaging format. The same image flows from developer laptop through CI pipeline to production deployment—on any cloud, on Kubernetes, or as serverless containers. The architect's role is to define the container strategy: base image standards, build pipeline, registry, and deployment target. Containers solve the "works on my machine" problem definitively, but they shift complexity to orchestration, networking, and security—which is why Kubernetes exists.
