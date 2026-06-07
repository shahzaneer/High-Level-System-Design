# Hybrid Cloud Architecture

## Introduction
Hybrid cloud is the reality for most enterprises—not a temporary state on the way to "all cloud." A 2023 Flexera survey found 72% of enterprises use a hybrid cloud model combining on-premises data centers with public cloud. The drivers are diverse: regulatory data sovereignty requirements, latency-sensitive edge workloads, mainframe applications that cannot migrate, sunk-cost data center investments, and workloads that are predictably cheaper on-premises.

Hybrid cloud architecture is the discipline of making on-premises and cloud environments work as a cohesive system—shared identity, consistent networking, unified observability, and portable workloads. It's not about maintaining two separate environments; it's about building one operational model that spans locations. The architecture defines how data flows between environments, how users authenticate, how applications are deployed, and how security is enforced consistently.

## Definition

**Hybrid Cloud** is a computing environment that combines on-premises infrastructure (or private cloud) with public cloud services, with orchestration and data exchange between them. Key characteristics:

- **Unified Identity**: Single identity provider across environments (Azure AD, Okta, AWS SSO federated to on-prem AD)
- **Consistent Networking**: Private, encrypted connectivity (Direct Connect, ExpressRoute, Cloud Interconnect, VPN)
- **Common Operations**: Single monitoring, logging, and management plane
- **Portable Workloads**: Applications can be deployed to either environment with minimal changes

## Concept Explanation

### Hybrid Cloud Architecture Patterns

#### Pattern 1: Cloud-Bursting
On-premises handles baseline load; cloud absorbs spikes.

```
Normal operations:
[Users] → [On-Prem Apps (baseline capacity)]
                ↓
          [Stretched Cluster or Auto-Scale Trigger]
                ↓
During peak:
[Users] → [Cloud Apps (burst capacity)]  +  [On-Prem Apps (still running)]
```

```python
class CloudBurstingController:
    def __init__(self):
        self.on_prem_capacity = 1000  # requests/second
        self.cloud_min_instances = 0
        self.cloud_max_instances = 50
    
    def monitor_and_scale(self, current_rps):
        if current_rps > self.on_prem_capacity * 0.8:  # 80% threshold
            # Burst to cloud
            additional_capacity = (current_rps - self.on_prem_capacity) / 100
            cloud_instances = min(additional_capacity, self.cloud_max_instances)
            self._scale_cloud(cloud_instances)
        
        if current_rps < self.on_prem_capacity * 0.5:  # Scale down
            self._scale_cloud(0)
```

#### Pattern 2: Edge + Cloud
Processing at edge locations (retail stores, factories, IoT), centralized in cloud.

```
[Edge Locations] ──filtered/aggregated──→ [Cloud (analytics, ML, global state)]
     │
     ├── Local processing (low latency)
     ├── Offline operation (intermittent connectivity)
     └── Data sync when connected
```

#### Pattern 3: Data Residency + Cloud Processing
Sensitive data stays on-premises; processed results go to cloud.

```
[On-Premises (PII/PHI data)]  ──anonymized/aggregated──→  [Cloud (analytics, ML)]
     │                                                          │
     ├── Raw customer data (never leaves)                       ├── Dashboards
     ├── Payment processing (PCI scope)                        ├── Recommendations
     └── Health records (HIPAA scope)                          └── Reporting
```

#### Pattern 4: DR and Backup
Cloud as the DR target for on-premises workloads.

```
[On-Premises Primary] ──continuous replication──→ [Cloud DR Site]
                                                    ├── Pilot light (DB replicas)
                                                    ├── Warm standby (scaled-down replica)
                                                    └── Backup storage (immutable copies)
```

### Hybrid Networking

#### Direct Connect / ExpressRoute
```hcl
# AWS Direct Connect
resource "aws_dx_connection" "on_prem" {
  name      = "on-prem-connection"
  bandwidth = "10Gbps"
  location  = "EqDC2"
}

resource "aws_dx_private_virtual_interface" "main" {
  connection_id    = aws_dx_connection.on_prem.id
  name             = "on-prem-vif"
  vlan             = 100
  address_family   = "ipv4"
  bgp_asn          = 65000
  
  amazon_address   = "169.254.1.1/30"
  customer_address = "169.254.1.2/30"
  
  dx_gateway_id    = aws_dx_gateway.main.id
}

# Private, consistent latency, bypasses internet
# Bandwidth: 1 Gbps - 100 Gbps
# Latency: Predictable (sub-5ms for metro, varies by distance)
```

#### VPN as Backup
```bash
# IPSec VPN as failover for Direct Connect
# If Direct Connect fails, route traffic through VPN automatically
# via BGP failover (lower preference for VPN routes)
```

### Unified Identity

```
┌──────────────────────────────────────────────┐
│         Active Directory (On-Prem)            │
│  ┌──────────────────────────────────────┐    │
│  │  Users, Groups, Computers            │    │
│  └──────────────┬───────────────────────┘    │
└─────────────────┼────────────────────────────┘
                  │
             AD Connect / ADFS Sync
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌───────┐    ┌───────┐    ┌───────┐
│  AWS  │    │ Azure │    │  GCP  │
│SSO/ADC│    │  AD   │    │ GCDS  │
└───────┘    └───────┘    └───────┘

# User logs in once (SSO), accesses any environment
# Identity is the same across on-prem and cloud
# Permissions differ per environment (Role mapping)
```

### Consistent Operations

```yaml
# Single monitoring across hybrid environments
monitoring:
  metrics:
    - Prometheus (on-prem) federated to Grafana Cloud
    - CloudWatch + Azure Monitor + GCP Monitoring → Central dashboard
  logs:
    - Fluentd agents everywhere → Splunk/Elasticsearch cloud
  alerts:
    - Single PagerDuty/Opsgenie for all environments
  dashboards:
    - Grafana: one pane of glass for on-prem + cloud metrics

# Single CI/CD pipeline
deployment:
  tool: GitLab CI / GitHub Actions
  targets:
    - on-prem Kubernetes (Rancher/OpenShift)
    - AWS EKS
    - Azure AKS
    - GCP GKE
  # Same pipeline deploys to any environment
```

### Application Design for Portability

```python
# Abstract cloud-specific services behind interfaces
from abc import ABC, abstractmethod

class BlobStorage(ABC):
    @abstractmethod
    def put(self, key: str, data: bytes): ...
    @abstractmethod
    def get(self, key: str) -> bytes: ...

class S3Storage(BlobStorage):  # AWS
    def put(self, key, data):
        s3.put_object(Bucket='data', Key=key, Body=data)

class MinioStorage(BlobStorage):  # On-prem S3-compatible
    def put(self, key, data):
        minio_client.put_object(Bucket='data', ObjectName=key, data=data)

class AzureStorage(BlobStorage):  # Azure
    def put(self, key, data):
        blob_client.upload_blob(name=key, data=data)

# Application uses the abstraction
storage = get_storage_backend()  # Resolved from config/environment
storage.put('orders/2024-06-15.json', order_data)
```

## Layman's Explanation

### The Vacation Home
You have a primary house (on-premises data center) and a vacation home (public cloud):

- **Identity**: Same set of keys opens both houses (single sign-on)
- **Networking**: A private road connects them (Direct Connect). During heavy snow on the private road, you can take the public highway (VPN backup)
- **Bursting**: During the holidays when extended family visits, you overflow to the vacation home because your primary house can only sleep 8
- **Data Residency**: Your birth certificate and tax documents stay in the primary home's safe (PII on-premises). Holiday photos go to the vacation home (analytics in cloud)
- **Consistent Operations**: The same cleaning service handles both houses (same monitoring tools). The same security system protects both (same IAM policies)
- **DR**: If the primary house has a fire, you can live in the vacation home full-time (DR failover)

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Data Gravity**: Data has gravity—large datasets attract applications. If 100TB of transactional data lives on-premises, moving that application to the cloud may require moving the data (expensive, slow) or accessing it across the network (high latency). Data placement is often the deciding factor in hybrid architecture.
- **Latency Budget**: Applications designed for local networks (sub-ms latency) fail when split across environments (Direct Connect adds 5-50ms). Each cross-environment call must be evaluated against the application's latency tolerance.
- **Network Bandwidth Costs**: Cloud egress ($0.05-$0.12/GB) makes naively moving data between on-prem and cloud expensive. Architecture must minimize cross-environment data transfer—process data where it lives, transfer only results.
- **Operational Consistency**: Running Kubernetes on-prem (OpenShift, Rancher) and in cloud (EKS, AKS, GKE) with identical configurations is ideal but complex. Differences in storage drivers, networking CNI, and load balancers must be abstracted.

### Business Impact
- **Regulatory Compliance**: Some data CANNOT leave on-premises (government classified, certain financial data, some healthcare data). Hybrid cloud is not a choice—it's a regulatory requirement.
- **Cost Optimization**: Steady-state, predictable workloads are often cheaper on-premises (sunk cost infrastructure). Variable, bursty workloads are cheaper in cloud. Hybrid lets each workload run where it's most cost-effective.
- **Acquisition Integration**: Acquired companies often run on different cloud providers. Hybrid architecture enables gradual integration without forced immediate migrations.

## On-Premises Examples

### HashiCorp Consul (Multi-Cloud Service Mesh)
```yaml
# Consul federation between on-prem and cloud
# datacenter config (on-prem)
datacenter: "dc1"
primary_datacenter: "dc1"
connect:
  enabled: true

# datacenter config (AWS)
datacenter: "aws-us-east-1"
primary_datacenter: "dc1"
retry_join:
  - "provider=aws tag_key=consul tag_value=server"
```

### Rancher (Multi-Cluster Kubernetes)
```bash
# Manage on-prem and cloud clusters from single pane of glass
# Import existing clusters
rancher cluster import EKS-cluster
rancher cluster import GKE-cluster
rancher cluster import on-prem-cluster

# Deploy to all clusters
kubectl apply -f deployment.yaml --context=on-prem
kubectl apply -f deployment.yaml --context=aws
```

## AWS Examples

### AWS Outposts (On-Premises AWS)
```hcl
resource "aws_outposts_outpost" "main" {
  name        = "on-prem-outpost"
  description = "On-premises AWS infrastructure"
  site_id     = "SITE-12345"

  supported_hardware_type = "OUTPOSTS_HARDWARE"
}
# AWS hardware in your data center
# Same APIs, same services (EC2, ECS, RDS, S3 on Outposts)
# Unified management in AWS console
```

### AWS Direct Connect + Transit Gateway
```hcl
resource "aws_dx_gateway_association" "main" {
  dx_gateway_id         = aws_dx_gateway.main.id
  associated_gateway_id = aws_ec2_transit_gateway.main.id
}

resource "aws_ec2_transit_gateway" "main" {
  description = "Hybrid network hub"
  # Connects: VPCs + Direct Connect + VPN
  # Single routing hub for hybrid architecture
}
```

### AWS Directory Service
```hcl
resource "aws_directory_service_directory" "main" {
  name     = "corp.example.com"
  password = var.directory_password
  edition  = "Enterprise"
  type     = "MicrosoftAD"

  vpc_settings {
    vpc_id     = aws_vpc.main.id
    subnet_ids = aws_subnet.private[*].id
  }
}
# Trust relationship with on-prem AD
# Users authenticate with same credentials
# AWS resources can join the domain
```

## GCP Examples

### Anthos (Multi-Cloud Kubernetes)
```bash
# GKE on-prem (Anthos clusters on VMware/Bare Metal)
gcloud container fleet memberships register on-prem-cluster \
  --gke-cluster-self-link=//...

# Unified management:
# - Same Kubernetes API everywhere
# - Same IAM (Workload Identity)
# - Same monitoring (Cloud Operations)
# - Same service mesh (Anthos Service Mesh / Istio)

# Deploy to on-prem from cloud
kubectl apply -f deployment.yaml --context=on-prem-cluster
```

### Cloud Interconnect
```bash
gcloud compute interconnect attachments create on-prem-attachment \
  --router=my-router \
  --interconnect=my-interconnect \
  --vlan-tag8021q=100

gcloud compute routers add-interface my-router \
  --interface-name=on-prem-int \
  --interconnect-attachment=on-prem-attachment \
  --ip-address=169.254.1.1 \
  --mask-length=30
```

## Azure Examples

### Azure Arc (Multi-Cloud Management)
```bash
# Extend Azure management to on-premises and other clouds
az connectedmachine connect \
  --resource-group myResourceGroup \
  --name on-prem-server-1 \
  --location eastus

# Now the on-prem server appears in Azure Portal
# Can apply Azure Policy, Azure Monitor, Azure Update Manager
```

### Azure Stack HCI
```bash
# Azure-consistent on-premises hyperconverged infrastructure
az stack-hci cluster create \
  --name on-prem-cluster \
  --resource-group myResourceGroup \
  --location eastus

# Run Azure services on-premises:
# - Azure Kubernetes Service (AKS) on HCI
# - Azure Virtual Desktop
# - Azure SQL Managed Instance
# All managed through Azure Portal
```

### Azure ExpressRoute
```bash
az network express-route create \
  --name express-route \
  --resource-group myResourceGroup \
  --bandwidth "10 Gbps" \
  --peering-location "Equinix DC2" \
  --provider "Equinix"

az network express-route gateway create \
  --name express-route-gateway \
  --vnet myVNet \
  --resource-group myResourceGroup
```

## Summary Decision Matrix

| Hybrid Architecture Decision | Recommendation |
|------------------------------|----------------|
| Connectivity | Direct Connect/ExpressRoute primary, VPN backup |
| Identity | Single AD/Azure AD source, sync to cloud directories |
| Container Orchestration | Kubernetes everywhere (EKS/GKE/AKS + on-prem K8s) |
| Monitoring | Single pane of glass (Grafana, Datadog, Splunk) |
| CI/CD | Single pipeline, environment-specific configs |
| Data Strategy | Process data where it lives; transfer only results |
| Security | Consistent IAM policies via IaC across all environments |

Hybrid cloud is the long-term reality for most enterprises. The architect's challenge is making the hybrid environment feel like one system, not two. Unified identity, consistent networking, standard container platform, and common monitoring create operational coherence. The goal is not to eliminate the on-premises environment but to make the boundary between on-premises and cloud invisible to developers and users alike.
