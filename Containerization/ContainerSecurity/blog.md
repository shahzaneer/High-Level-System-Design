# Container Security

## Introduction
Container security is fundamentally different from traditional server security. In a VM-centric world, a server might run for years—you patch it monthly, configure a firewall, and install an antivirus. In a container world, instances live for hours or minutes, are created and destroyed automatically, and share the host kernel. The attack surface shifts: from persistent servers that need hardening to ephemeral containers that need secure images, and from perimeter firewalls to cluster-level network policies.

The "4 C's" of cloud-native security (Cloud → Cluster → Container → Code) form a layered defense. Container security specifically addresses the container and code layers: image vulnerabilities, runtime security, least-privilege execution, and supply chain integrity. A single vulnerable base image can expose thousands of container instances across the cluster.

## Key Practices

### The 4 C's of Cloud-Native Security

```
CLOUD (Infrastructure Security)
  ├── IAM, VPC, encryption, compliance
  │
CLUSTER (Kubernetes Security)
  ├── RBAC, Network Policies, Pod Security Standards, audit logging
  │
CONTAINER (Image + Runtime Security)
  ├── Minimal images, non-root, read-only FS, no privileged mode
  │
CODE (Application Security)
  ├── SAST, dependency scanning, secret management, input validation
```

### Image Security

```dockerfile
# Secure by default Dockerfile pattern
FROM python:3.12-slim@sha256:abc...  # Pin by digest, not tag

# Install only necessary packages, clean up package cache
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl && \
    rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r app && useradd -r -g app app

# Copy with explicit ownership
COPY --chown=app:app . /app

WORKDIR /app
USER app

# Read-only root filesystem (writable /tmp as tmpfs)
# docker run --read-only --tmpfs /tmp

CMD ["python", "app.py"]
```

### Runtime Security Constraints (Kubernetes)

```yaml
apiVersion: v1
kind: Pod
spec:
  # Pod Security Standard: Restricted
  securityContext:
    runAsNonRoot: true          # Must not run as root
    runAsUser: 1000              # Specific UID
    runAsGroup: 3000
    fsGroup: 2000                # Filesystem group
    seccompProfile:
      type: RuntimeDefault      # Block dangerous syscalls
  
  containers:
  - name: app
    image: order-service:latest
    
    securityContext:
      allowPrivilegeEscalation: false   # Cannot gain more privileges
      readOnlyRootFilesystem: true      # Cannot write to container FS
      capabilities:
        drop: ["ALL"]                   # Drop ALL Linux capabilities
        add: ["NET_BIND_SERVICE"]       # Add only what's needed
      runAsNonRoot: true
    
    volumeMounts:
    - name: tmp
      mountPath: /tmp         # Writable /tmp
    - name: cache
      mountPath: /var/cache   # Writable cache
  
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

### Network Policies (Zero-Trust Networking)

```yaml
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
  # Allow from API gateway only
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    - namespaceSelector:
        matchLabels:
          name: ingress
    ports:
    - protocol: TCP
      port: 8080
  
  egress:
  # Allow to database only
  - to:
    - podSelector:
        matchLabels:
          app: order-database
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow DNS resolution
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  
  # Deny all other egress (no internet access unless explicitly allowed)
```

### Supply Chain Security

```bash
# 1. Sign commits (prove authorship)
git commit -S -m "Fix security vulnerability"

# 2. Sign container images (prove image integrity)
cosign sign --key cosign.key $IMAGE:$TAG

# 3. Verify signatures before deployment
cosign verify --key cosign.pub $IMAGE:$TAG

# 4. Generate SBOM (know what's inside)
syft $IMAGE:$TAG -o spdx-json > sbom.json

# 5. Attestation (prove build provenance)
cosign attest --predicate build-provenance.json --key cosign.key $IMAGE
```

### Vulnerability Management

```bash
# Continuous scanning in CI/CD and registry
trivy image --severity CRITICAL,HIGH order-service:latest
trivy image --exit-code 1 --severity CRITICAL order-service:latest  # Fail on critical

# grype (Anchore)
grype order-service:latest --fail-on high

# docker scout
docker scout quickview order-service:latest
docker scout cves order-service:latest --only-fixed
```

### Runtime Threat Detection

```yaml
# Falco: runtime security monitoring
# Detects: unexpected processes, file access, network connections, syscalls

# Falco rule: detect shell spawned in container
- rule: Terminal shell in container
  desc: A shell was spawned in a container
  condition: >
    spawned_process
    and container
    and shell_procs
    and proc.tty != 0
  output: >
    Shell spawned in container (user=%user.name container=%container.name)
  priority: WARNING

# Falco rule: detect writing to /etc (persistence attempt)
- rule: Write below etc
  condition: >
    evt.dir = < and evt.type = openat
    and container
    and fd.name startswith /etc
  output: "File below /etc opened for writing"
  priority: WARNING
```

## AWS, GCP, Azure Examples

```hcl
# AWS Inspector: automated vulnerability scanning
resource "aws_inspector2_enabler" "main" {
  resource_types = ["ECR"]
}
```

```bash
# GCP Binary Authorization: only allow verified images
gcloud container clusters update prod-cluster \
  --enable-binary-authorization

# GCP Artifact Analysis: automatic vulnerability scanning
gcloud artifacts docker images scan order-service:latest
```

```bash
# Azure Defender for Containers
az aks enable-addons --addons azure-defender --name prod-cluster
```

## Summary

| Layer | Control |
|-------|---------|
| Image | Minimal base, pin digests, scan for CVEs, sign images |
| Registry | Private registry, vulnerability scanning, signed images only |
| Runtime | Non-root, read-only FS, drop capabilities, seccomp |
| Network | Network Policies, deny all default, explicit allows |
| Supply Chain | Signed commits, signed images, SBOM, provenance attestations |
| Detection | Falco, Defender, runtime anomaly detection |

Container security operates at multiple layers: secure the image (minimal, scanned, signed), constrain the runtime (non-root, restricted, seccomp), limit the network (zero-trust policies), and verify the supply chain (SBOM, attestations). No single control is sufficient—defense in depth is the strategy. The architect must ensure security controls are automated in the CI/CD pipeline and enforced at admission time, not dependent on manual configuration.
