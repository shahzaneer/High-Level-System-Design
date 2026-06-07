# On-Premises & Alternative Kubernetes

## Introduction
While cloud-managed Kubernetes (EKS, GKE, AKS) dominates production deployments, on-premises Kubernetes remains critical for regulated industries, edge computing, air-gapped environments, and organizations with significant data center investments. The ecosystem has matured to provide enterprise-grade Kubernetes on your own infrastructure—OpenShift (Red Hat), Rancher (SUSE), VMware Tanzu, and vanilla kubeadm—each with different trade-offs between operational simplicity, feature completeness, and cloud integration.

The key insight: on-premises Kubernetes is 10x harder to operate than cloud-managed Kubernetes. The cloud providers handle control plane HA, etcd backups, node patching, and API server upgrades. On-premises, you must handle all of this yourself. The choice of distribution is largely about how much of this operational burden the platform absorbs for you.

## Concept Explanation

### Distribution Comparison

```
┌──────────────────────────────────────────────────────────────┐
│                   KUBERNETES DISTRIBUTIONS                    │
├──────────┬─────────────┬──────────┬──────────┬──────────────┤
│ kubeadm  │ OpenShift   │ Rancher  │ VMware   │ Mirantis     │
│ (CNCF)   │ (Red Hat)   │ (SUSE)   │ Tanzu    │ Docker EE    │
├──────────┼─────────────┼──────────┼──────────┼──────────────┤
│ Pure     │ Enterprise  │ Multi-   │ vSphere  │ Docker-      │
│ upstream │ K8s +       │ cluster  │ integra- │ native       │
│ K8s      │ developer   │ mgmt     │ tion     │              │
│          │ platform    │          │          │              │
├──────────┼─────────────┼──────────┼──────────┼──────────────┤
│ Free     │ Subscription│ Free+    │ License  │ License      │
│          │             │ Subscrip │          │              │
├──────────┼─────────────┼──────────┼──────────┼──────────────┤
│ Most ops │ Integrated  │ Central  │ vSphere  │ Tight Docker │
│ burden   │ CI/CD,      │ manage-  │ admin    │ integration  │
│          │ registry,   │ ment     │ familiar │              │
│          │ monitoring  │ pane     │          │              │
└──────────┴─────────────┴──────────┴──────────┴──────────────┘
```

### kubeadm (Vanilla Kubernetes)

```bash
# Control plane initialization
kubeadm init \
  --control-plane-endpoint="k8s-api.internal:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12

# Join additional control plane nodes (HA: 3 control planes for etcd quorum)
kubeadm join k8s-api.internal:6443 \
  --token TOKEN \
  --discovery-token-ca-cert-hash sha256:HASH \
  --control-plane --certificate-key CERT_KEY

# Join worker nodes
kubeadm join k8s-api.internal:6443 --token TOKEN \
  --discovery-token-ca-cert-hash sha256:HASH

# Install CNI (Flannel, Calico, Cilium)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Install MetalLB (Load Balancer for bare metal)
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
```

```yaml
# MetalLB configuration: provides LoadBalancer Service type on-prem
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250  # Pool of IPs for LoadBalancer services
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
```

### Red Hat OpenShift

```
OpenShift = Kubernetes + Enterprise Platform:

  Security:
    - SELinux on every node (mandatory)
    - Rootless containers by default (Security Context Constraints)
    - Integrated image registry with vulnerability scanning
    - Built-in OAuth/OpenID identity provider integration
  
  Developer Experience:
    - Source-to-Image (S2I): build images from source without Dockerfile
    - OpenShift Pipelines (Tekton): CI/CD integrated
    - Developer Console: GUI for application management
    - CodeReady Workspaces: browser-based IDE
  
  Operations:
    - Cluster Version Operator: automated upgrades
    - Integrated monitoring (Prometheus + Grafana + Alertmanager)
    - Integrated logging (Elasticsearch + Fluentd + Kibana)
    - Operator Lifecycle Manager: managed operator deployments
```

```bash
# OpenShift installation (IPI - Installer Provisioned Infrastructure)
openshift-install create install-config --dir=./cluster
# Edit install-config.yaml with cluster configuration
openshift-install create cluster --dir=./cluster

# oc CLI (kubectl + OpenShift extensions)
oc login https://api.openshift.internal:6443
oc new-project order-processing
oc new-app python:3.12~https://github.com/org/order-service.git  # S2I build
oc expose svc/order-service  # Create route (Ingress)
```

**Security Context Constraints (SCC)**:
```yaml
# OpenShift enforces pod security differently than vanilla K8s
# SCCs are like Pod Security Standards, but more powerful
oc describe scc restricted  # Default SCC: no root, no privileged
# OpenShift blocks root containers by default
# Must explicitly grant SCC if container truly needs it
```

### Rancher (SUSE)

```
Rancher = Multi-Cluster Kubernetes Management:

  - Single pane of glass for hundreds of clusters
  - Import existing clusters (EKS, GKE, AKS) or create new ones
  - Centralized authentication (LDAP, SAML, GitHub, AD)
  - Centralized RBAC across all clusters
  - Integrated monitoring, logging, alerting
  - Fleet: GitOps at scale across clusters
  - Longhorn: distributed block storage for stateful workloads
```

```bash
# Deploy Rancher (single Docker container or Helm on K8s)
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  --privileged \
  rancher/rancher:latest

# Or via Helm
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.internal.com \
  --set replicas=3
```

```yaml
# Rancher Fleet: GitOps across clusters
kind: GitRepo
apiVersion: fleet.cattle.io/v1alpha1
metadata:
  name: order-service
spec:
  repo: https://github.com/org/fleet-deployments
  branch: main
  paths:
  - apps/order-service
  targets:
  - clusterSelector:
      matchLabels:
        env: production
```

### VMware Tanzu

```
VMware Tanzu = Kubernetes for vSphere shops:

  - Deep vSphere integration (vCenter, vSAN, NSX-T)
  - Supervisor Cluster (vSphere with Tanzu): K8s API on vSphere
  - TKG (Tanzu Kubernetes Grid): multi-cloud K8s
  - Use vSphere storage policies for K8s PVs
  - Use NSX-T for K8s networking and load balancing
  - Antrea CNI (built for VMware networking)
```

### Essential On-Prem Components

```yaml
# MetalLB: LoadBalancer services (bare metal)
# OR HAProxy + Keepalived for external load balancing

# Rook/Ceph: distributed storage (Ceph on K8s)
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v17.2
  storage:
    useAllNodes: true
    useAllDevices: true

# cert-manager: automated TLS certificate management
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: ca-key-pair
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-internal-tls
spec:
  secretName: api-internal-tls
  dnsNames:
  - api.internal.example.com
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer

# ExternalDNS + CoreDNS for service discovery
# Harbor: private container registry
# Keycloak: identity provider (OIDC for K8s auth)
```

### On-Prem Networking Considerations

```
On-prem networking challenges:

1. Load Balancing:
   Cloud: ALB/NLB/CLB - one annotation away
   On-prem: MetalLB (L2/BGP), HAProxy, F5 BIG-IP, Nginx Ingress

2. Storage:
   Cloud: EBS/EFS, Persistent Disk, Azure Disk/Files - provisioned automatically
   On-prem: Rook/Ceph, Longhorn, OpenEBS, Portworx, NFS

3. Ingress:
   Cloud: Application Load Balancer, Cloud Load Balancing, App Gateway
   On-prem: Nginx Ingress, HAProxy Ingress, Traefik with NodePort/LoadBalancer IP

4. DNS:
   Cloud: Route 53/Cloud DNS/Azure DNS external-dns integration
   On-prem: CoreDNS + ExternalDNS with RFC2136, or manual DNS
```

## Layman's Explanation

### Building Your Own Power Plant
Cloud Kubernetes (EKS/GKE/AKS) is like buying electricity from the power grid—you pay for what you use, and the utility handles generation, transmission, and reliability. 

On-premises Kubernetes (kubeadm) is like building your own power plant—you need to maintain the generators (control plane), the transmission lines (networking), the backup systems (etcd backups), and the repair crew (operations team). It's more work, but you control everything.

OpenShift/Rancher is like buying a pre-fabricated power plant kit—all the pieces are there, engineered to work together, with instructions and warranty. You still need to operate it, but you're not designing it from scratch.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Build vs Buy (kubeadm vs OpenShift/Rancher)**: kubeadm is $0 but requires 3+ dedicated K8s engineers to operate. OpenShift costs $15K+/node/year but includes enterprise support, integrated tooling, and validated upgrades. The crossover point where OpenShift becomes cheaper than staffing a K8s team is around 50-100 nodes.
- **Storage Platform**: On-prem K8s doesn't have EBS/EFS equivalent. You must choose and operate a distributed storage system: Rook/Ceph (most capable, highest complexity), Longhorn (simplest, good for smaller clusters), or traditional SAN/NAS integration (CSI plugins for Dell EMC, NetApp, Pure Storage).
- **Load Balancing Strategy**: BGP (MetalLB BGP mode) + router integration provides the most cloud-like LoadBalancer experience. L2 mode is simpler but has single-node bottleneck. External load balancer (F5, HAProxy pair) is operationally familiar to network teams but less K8s-native.
- **Upgrade Strategy**: Cloud K8s upgrades are a button click or automatic. On-prem upgrades require careful planning: etcd backup, control plane upgrade (one at a time for HA), worker node drain/upgrade, API deprecation checking. OpenShift's Cluster Version Operator automates much of this.

### Business Impact
- **Air-Gapped Environments**: Government, defense, and critical infrastructure often CANNOT connect to the internet. On-prem K8s with local registries (Harbor), local Helm charts, and offline documentation is the only option.
- **Data Sovereignty**: Some regulations require data to remain in specific physical locations or under specific physical control. On-prem K8s satisfies this definitively.
- **Sunk-Cost Data Centers**: Organizations with $50M+ invested in data centers cannot abandon them for cloud without a 5-10 year migration timeline. On-prem K8s modernizes the existing infrastructure rather than replacing it.

## Summary

| Distribution | Best For | Cost | Operational Burden |
|-------------|----------|------|-------------------|
| kubeadm | DIY, cost-sensitive, K8s experts | $0 (licensing) | Very High |
| OpenShift | Enterprise, regulated, developer platform | $$$$ (subscription) | Medium |
| Rancher (RKE2) | Multi-cluster, centralized management | Free / Rancher Prime $$$ | Medium (clusters) + Low (management) |
| VMware Tanzu | vSphere shops | $$$$ (license) | Medium |
| RKE2 (Rancher's K8s) | FIPS-compliant, security-focused | Free | Medium-Low |

On-premises Kubernetes is operationally demanding but strategically essential for regulated, air-gapped, and data-sovereignty-constrained environments. The choice of distribution—kubeadm for DIY, OpenShift for enterprise safety, Rancher for multi-cluster, Tanzu for vSphere—determines the operational burden, security posture, and developer experience. The best distribution is the one your team has the expertise to operate securely and reliably. An unpatched, misconfigured on-prem K8s cluster is far more dangerous than a well-operated one, regardless of which distribution it runs.
