# Docker & Image Best Practices

## Introduction
Docker images are the building blocks of containerized applications. A well-crafted image is small, secure, deterministic, and optimized for caching. A poorly crafted image is bloated (2GB instead of 200MB), insecure (runs as root, contains secrets), and slow to build (no layer caching, rebuilds everything on every change).

The difference between a good and bad Docker image is not cosmetic—it directly impacts build speed (developer feedback loop), deployment speed (image pull time affects scale-out speed), security (more packages = more CVEs), and cost (larger images consume more registry storage and network bandwidth). Mastering Docker image construction is a prerequisite for any containerized deployment.

## Key Practices

### Multi-Stage Builds

```dockerfile
# BAD: Single stage, build tools in production image
FROM python:3.12
COPY . .
RUN pip install -r requirements.txt
RUN apt-get update && apt-get install -y gcc build-essential
RUN pip install some-c-extension
CMD ["python", "app.py"]
# Size: ~900MB. Contains gcc, header files, build artifacts.

# GOOD: Multi-stage - build tools never reach production
FROM python:3.12 AS builder
RUN apt-get update && apt-get install -y gcc build-essential
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.12-slim AS runtime
COPY --from=builder /root/.local /root/.local
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
# Size: ~180MB. No build tools. Minimal attack surface.
```

### Layer Order for Cache Optimization

```dockerfile
# OPTIMAL: Dependencies installed before code (cached unless deps change)
FROM node:20-slim
WORKDIR /app

# 1. Install system dependencies (changes rarely)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# 2. Copy dependency files first (changes occasionally)
COPY package.json package-lock.json ./

# 3. Install dependencies (expensive; cached if package.json unchanged)
RUN npm ci --production

# 4. Copy application code last (changes most frequently)
COPY . .

# 5. Build (if needed; often done in multi-stage)
RUN npm run build

CMD ["node", "dist/server.js"]
```

### Minimal Base Images

```
Image Size Comparison:
  ubuntu:latest         → 78MB
  debian:bookworm-slim  → 74MB
  alpine:3.19           → 7MB (!)
  distroless/static     → 2MB (!!)
  scratch               → 0MB (FROM scratch for Go/Rust statically compiled)

Trade-offs:
  Alpine: Smallest general-purpose, but uses musl libc (not glibc)
          Some Python packages with C extensions fail on Alpine
  
  Debian-slim: Good balance of size (~80MB) and compatibility (glibc)
  
  Distroless: Ultra-secure (no shell, no package manager, no utilities)
              Can't docker exec into it for debugging
              Perfect for production; use debug image for development
  
  Recommendation: Start with debian-slim, graduate to distroless for production
```

### Non-Root User

```dockerfile
# BAD: runs as root (default)
FROM python:3.12-slim
COPY app.py .
CMD ["python", "app.py"]
# Container process is UID 0 (root)
# If container is compromised, attacker has root on the host

# GOOD: runs as non-root user
FROM python:3.12-slim
RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --create-home appuser
WORKDIR /home/appuser/app
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["python", "app.py"]
```

### .dockerignore

```dockerignore
# Prevent secrets and build artifacts from entering the image
.git
.gitignore
.env
*.log
node_modules
__pycache__
*.pyc
.pytest_cache
.venv
venv
*.md
Dockerfile
docker-compose*.yml
.idea
.vscode
*.swp
*.swo
```

### Deterministic Builds (Pinning Versions)

```dockerfile
# BAD: floating tags (non-deterministic)
FROM python:3.12  # What's "3.12" today? Could be different tomorrow

# BETTER: pin to minor version
FROM python:3.12.3

# BEST: pin to SHA256 digest (fully deterministic)
FROM python:3.12.3-slim@sha256:abc123def456...
```

### Healthcheck

```dockerfile
FROM python:3.12-slim
COPY app.py .

# Liveness: is the process running?
# Readiness: is it ready to serve? (check /health endpoint)

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["python", "app.py"]
```

### Signal Handling (Graceful Shutdown)

```dockerfile
# Docker sends SIGTERM, waits grace period (default 10s), then SIGKILL
# Application must handle SIGTERM for graceful shutdown

# BAD: shell form (CMD python app.py) - PID 1 is /bin/sh, doesn't forward signals
CMD python app.py

# GOOD: exec form - PID 1 is the application, receives signals directly
CMD ["python", "app.py"]

# GOOD: tini as init process (handles zombie processes, signal forwarding)
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["python", "app.py"]
# Docker now has --init flag to add tini automatically
```

### Full Production-Ready Dockerfile

```dockerfile
# Production-ready Python Dockerfile
FROM python:3.12.3-slim@sha256:abc123... AS builder

# Build dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential gcc && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /build
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.12.3-slim@sha256:abc123... AS runtime

# Install runtime system deps only
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd --gid 1000 app && \
    useradd --uid 1000 --gid app --no-create-home app

WORKDIR /app

# Copy dependencies from builder
COPY --from=builder --chown=app:app /root/.local /home/app/.local

# Copy application
COPY --chown=app:app . .

# Set environment
ENV PATH=/home/app/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Switch to non-root user
USER app

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Expose port (documentation; does NOT publish port)
EXPOSE 8080

# Exec form for signal handling
ENTRYPOINT ["python", "-m", "app"]
```

## On-Prem, AWS, GCP, Azure Examples

```bash
# Build with BuildKit (faster, better caching)
DOCKER_BUILDKIT=1 docker build \
  --cache-from=order-service:latest \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t order-service:1.0.0 .

# Image layer analysis
dive order-service:1.0.0  # Interactive layer explorer

# Image size history
docker image history order-service:1.0.0

# AWS ECR: image scanning on push
aws ecr put-image-scanning-configuration \
  --repository-name order-service \
  --image-scanning-configuration scanOnPush=true

# GCP: vulnerability scanning
gcloud artifacts docker images scan order-service:1.0.0

# Azure: ACR Tasks for automated builds
az acr build --registry myacr --image order-service:1.0.0 .
```

## Summary

| Practice | Benefit |
|----------|---------|
| Multi-Stage Builds | Small images, no build tools in production |
| Layer Ordering | Fast builds via caching |
| Minimal Base Image | Small attack surface, fast pulls |
| Non-Root User | Defense in depth |
| Pinned Versions | Deterministic, reproducible builds |
| Healthcheck | Orchestrator knows if container is healthy |
| Signal Handling | Graceful shutdown, no dropped requests |

A well-crafted Docker image is a security control, a performance optimization, and a reliability mechanism. The image that runs in production should contain exactly what the application needs to run and nothing more—no build tools, no development dependencies, no shell if not needed. Every byte in the image is a byte that must be scanned for vulnerabilities, transferred across the network, and stored in the registry. The architect must establish image standards that every team follows, enforced through automated checks in CI/CD.
