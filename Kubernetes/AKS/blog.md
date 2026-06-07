# Azure Kubernetes Service (AKS)

## Introduction
Azure Kubernetes Service (AKS) is Microsoft's managed Kubernetes offering, launched in 2017. It integrates deeply with the Azure ecosystem: Azure Active Directory (Entra ID) for authentication, Azure Monitor for observability, Azure Policy for governance, and Azure Networking for VNet integration. AKS is particularly popular in enterprises already invested in the Microsoft ecosystem—those using .NET, Azure DevOps, Active Directory, and SQL Server.

AKS differentiates itself with strong enterprise governance features (Azure Policy for Kubernetes, Pod Identity / Workload Identity), tight security integration (Defender for Containers, Key Vault), and Windows container support (a unique feature among managed K8s services for organizations with legacy .NET Framework workloads).

## Concept Explanation

### AKS Architecture

```bash
# Create AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name production-cluster \
  --kubernetes-version 1.30 \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 20 \
  --network-plugin azure \
  --network-policy calico \
  --enable-managed-identity \
  --enable-aad \
  --aad-admin-group-object-ids GROUP_ID \
  --enable-azure-rbac \
  --enable-private-cluster \
  --enable-oidc-issuer \
  --auto-upgrade-channel stable \
  --node-os-upgrade-channel NodeImage

# AKS Automatic (Preview - similar to GKE Autopilot)
az aks create --name auto-cluster --sku Automatic
```

### Authentication: Azure AD Integration

```bash
# Cluster Admin: members of this AAD group get cluster-admin
az aks create --aad-admin-group-object-ids "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"

# Azure RBAC for Kubernetes Authorization
az aks create --enable-azure-rbac
# Uses Azure RBAC roles for K8s authorization instead of native K8s RBAC
# Roles: Azure Kubernetes Service RBAC Cluster Admin, Reader, Writer

# kubectl authentication:
az aks get-credentials --resource-group myRG --name production-cluster
# Gets token from Azure AD, writes to kubeconfig
```

### Workload Identity

```yaml
# Pod → Azure Managed Identity (pod-level identity)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-processor
  namespace: production
  annotations:
    azure.workload.identity/client-id: "CLIENT_ID_OF_MANAGED_IDENTITY"
---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: order-processor
  containers:
  - name: app
    # Azure SDK automatically uses managed identity via workload identity
    # Access Key Vault, Storage, SQL without connection strings or secrets
```

### Networking

```bash
# Azure CNI: Pods get IPs from VNet subnet
az aks create \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/.../subnets/pod-subnet \
  --service-cidr 10.100.0.0/16 \
  --dns-service-ip 10.100.0.10

# Azure CNI Overlay: Overlay network with private CIDR (more Pods)
az aks create \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 10.244.0.0/16

# Bring your own CNI (BYOCNI): Cilium, Calico, etc.
az aks create \
  --network-plugin none  # Install your own CNI
```

### Ingress: Application Gateway Ingress Controller (AGIC)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-api
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
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
# Automatically configures Azure Application Gateway (WAF-enabled L7 LB)
```

### Storage

```yaml
# Azure Disk CSI Driver
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: order-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 100Gi
---
# Azure Files CSI Driver (ReadWriteMany for shared storage)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-config
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-premium
  resources:
    requests:
      storage: 10Gi
---
# Azure Blob CSI Driver (object storage via CSI)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: blob-data
spec:
  storageClassName: azureblob-fuse-premium
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Ti
```

### Security

```bash
# Azure Policy for Kubernetes
az aks enable-addons --addons azure-policy --name production-cluster

# Policies enforced automatically:
# - Containers must not run as root
# - Only allowed registries (ACR only)
# - Read-only root filesystem
# - Resource limits required on all containers

# Defender for Containers
az aks enable-addons --addons azure-defender --name production-cluster
# Runtime threat protection, vulnerability scanning

# Key Vault integration via CSI Driver
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kv
spec:
  provider: azure
  parameters:
    keyvaultName: myAppKeyVault
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
    tenantId: "TENANT_ID"
```

### Windows Containers

```bash
# AKS supports Windows node pools
az aks nodepool add \
  --cluster-name production-cluster \
  --name winpool \
  --os-type Windows \
  --os-sku Windows2022 \
  --node-count 2 \
  --node-vm-size Standard_D4s_v5

# Linux and Windows nodes in the same cluster
# Pods scheduled to appropriate nodes via nodeSelector:
# nodeSelector:
#   kubernetes.io/os: windows
```

### Vertical Pod Autoscaler (VPA)

```bash
# Azure-managed VPA add-on
az aks enable-addons \
  --addons vertical-pod-autoscaler \
  --cluster-name production-cluster
```

### Cost Optimization

```bash
# Start/Stop cluster during off-hours (dev/staging)
az aks stop --name dev-cluster --resource-group myRG
az aks start --name dev-cluster --resource-group myRG

# Spot node pool (up to 90% discount)
az aks nodepool add \
  --cluster-name production-cluster \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1  # Pay up to on-demand price

# Reserved Instances: 1-year or 3-year commitment
# Apply reservation to AKS nodes running on Azure VMs
```

### Monitoring & Logging

```bash
# Container Insights (Azure Monitor)
az aks enable-addons --addons monitoring --name production-cluster

# Prometheus managed by Azure Monitor
az aks update --enable-azure-monitor-metrics --name production-cluster

# Grafana managed by Azure
az grafana create --name myGrafana --resource-group myRG
# Connect to Azure Monitor managed Prometheus as data source
```

## Layman's Explanation

AKS is Microsoft's home-field advantage for Kubernetes. If your company uses Windows, .NET, Active Directory, and Visual Studio, AKS is the natural choice—it's where the Microsoft ecosystem lives. The Azure AD integration means you log into Kubernetes with the same corporate credentials you use for email and Office 365. The Application Gateway integration gives you a WAF-enabled load balancer without configuring it separately. For enterprises committed to Microsoft, AKS makes Kubernetes feel like a native Azure service, not an add-on.

## Why Solution Architects Must Acquire This

### AKS-Specific Decisions
- **Azure CNI vs Azure CNI Overlay vs BYOCNI**: Azure CNI gives Pods VNet IPs (native integration, VNet service endpoints for Pods), but can exhaust IP space. Overlay solves IP exhaustion. BYOCNI gives maximum control (Cilium for eBPF, Hubble for observability) but requires operational expertise.
- **Azure RBAC vs Kubernetes RBAC**: Azure RBAC manages K8s permissions through Azure roles (familiar to Azure admins). Kubernetes RBAC is K8s-native. Choose Azure RBAC if your operations team is Azure-focused; K8s RBAC if your platform team is K8s-native.
- **Windows Containers**: AKS is the only major managed K8s service with mature Windows container support. For organizations migrating .NET Framework apps to containers without rewriting them to .NET Core/Linux, this is a critical feature.
- **Private Cluster**: AKS can be deployed with no public API server endpoint—only accessible within the VNet or via Azure Bastion. For enterprises with strict network security requirements, this is often mandatory.

## Summary

| AKS Feature | Benefit |
|------------|---------|
| Azure AD Integration | Corporate SSO for Kubernetes access |
| Azure Policy | Automated compliance enforcement |
| Workload Identity | Pods authenticate to Azure without secrets |
| Application Gateway | WAF-enabled ingress controller |
| Windows Containers | Legacy .NET workloads on Kubernetes |
| Key Vault CSI Driver | Mount secrets as files, no env vars |
| Defender for Containers | Runtime threat detection, vulnerability scanning |
| Private Cluster | No public API server endpoint |

AKS is the enterprise-friendly Kubernetes platform, particularly strong in Microsoft-centric organizations. The Azure AD integration, Windows container support, Application Gateway ingress, and deep security features (Defender, Key Vault, Policy) make it the natural choice for organizations already running on Azure. The tight governance integration—Azure Policy enforcing compliance automatically, Defender detecting runtime threats—reduces the operational burden of running production Kubernetes in regulated environments.
