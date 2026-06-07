# Network Security

## Introduction
Network security is the oldest and most fundamental layer of defense in system architecture. While modern security has embraced "identity as the perimeter," the network layer remains critical—it segments resources, controls traffic flow, prevents lateral movement after breach, and provides defense in depth. The shared responsibility model in cloud computing makes network security a joint effort: the cloud provider secures the physical network, while the architect secures the virtual network, subnets, firewall rules, and connectivity.

The shift from monolithic data centers to virtual private clouds, hybrid connectivity, and microservices has made network security more complex and more critical. A misconfigured security group that opens port 22 to `0.0.0.0/0` is one of the most common cloud security findings—and one of the easiest attack vectors.

## Definition
**Network Security** is the practice of protecting the integrity, confidentiality, and availability of data as it traverses networks. It encompasses:

- **Network Segmentation**: Dividing networks into isolated segments (subnets, VPCs, VLANs) to contain breaches and control traffic
- **Firewall / Security Groups**: Stateful or stateless rules that allow or deny traffic based on protocol, port, source, and destination
- **Access Control Lists (ACLs)**: Stateless rules for subnet-level traffic filtering
- **VPN and Private Connectivity**: Encrypted tunnels for connecting on-premises to cloud or cloud-to-cloud
- **Intrusion Detection/Prevention (IDS/IPS)**: Monitoring network traffic for malicious patterns and automatically blocking them
- **DDoS Protection**: Absorbing and filtering volumetric attacks at the network edge
- **Traffic Monitoring**: Flow logs, packet capture, and network observability for security analysis

## Concept Explanation

### The Network Security Stack

```
┌──────────────────────────────────────────────┐
│ Layer 7: WAF, API Gateway (application)      │
├──────────────────────────────────────────────┤
│ Layer 4: Security Groups, Stateful Firewalls │
├──────────────────────────────────────────────┤
│ Layer 3: NACLs, Route Tables, Subnets        │
├──────────────────────────────────────────────┤
│ Layer 2: VPC/VNet, Private Connectivity      │
├──────────────────────────────────────────────┤
│ Layer 1: Physical, DDoS Edge Protection      │
└──────────────────────────────────────────────┘
```

### Network Segmentation

#### Defense in Depth: The Three-Tier Architecture

```
┌─────────────────────────────────────────────┐
│              INTERNET                        │
└──────────────┬──────────────────────────────┘
               │
       ┌───────▼────────┐
       │  PUBLIC SUBNET  │  (Load Balancers, Bastion Hosts, NAT Gateways)
       │  10.0.1.0/24    │  Inbound: Internet (specific ports)
       └───────┬────────┘  Outbound: Internet (NAT)
               │
       ┌───────▼────────┐
       │  PRIVATE SUBNET │  (Application Servers, Containers, Lambda VPC)
       │  10.0.2.0/24    │  Inbound: Public subnet only
       └───────┬────────┘  Outbound: Internet via NAT, Database subnet
               │
       ┌───────▼────────┐
       │  DATABASE SUBNET│  (RDS, ElastiCache, DocumentDB)
       │  10.0.3.0/24    │  Inbound: Application subnet only
       └────────────────┘  Outbound: None (or VPC endpoints for backups)
```

```hcl
# AWS Three-Tier Network
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Public subnet (load balancers)
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.${count.index * 32}/27"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  map_public_ip_on_launch = true
}

# Private subnet (application)
resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.${count.index * 32}/27"
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

# Database subnet (isolated)
resource "aws_subnet" "database" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.3.${count.index * 32}/27"
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

### Security Groups vs NACLs

| Feature | Security Group (Stateful) | Network ACL (Stateless) |
|---------|--------------------------|------------------------|
| State | Return traffic auto-allowed | Must explicitly allow inbound + outbound |
| Scope | Attached to individual resources | Attached to subnet (applies to all resources) |
| Rules | Allow rules only | Allow AND Deny rules |
| Order | All rules evaluated (most permissive) | Rules evaluated by number (first match wins) |
| Use case | Resource-level access control | Subnet-level boundary protection |

```hcl
# Security Groups (stateful, resource-level)
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # HTTPS from anywhere
    description = "HTTPS access"
  }

  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    cidr_blocks     = ["10.0.100.0/24"]  # SSH only from bastion subnet
    description     = "SSH from bastion"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }
}

resource "aws_security_group" "database" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # Only from app SG
    description     = "PostgreSQL from application layer"
  }

  # NO SSH ingress
  # NO direct internet ingress
  # Egress only to VPC endpoints (S3, KMS, etc.)
}

# Network ACLs (stateless, subnet-level—defense in depth)
resource "aws_network_acl" "database" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.database[*].id

  # Deny all outbound internet
  egress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 65535
  }
}
```

### Micro-Segmentation (Zero Trust Networking)

Beyond traditional three-tier: every workload has its own security group, and traffic is explicitly allowed between specific services only.

```yaml
# Kubernetes NetworkPolicy (micro-segmentation)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: production
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
    - podSelector:
        matchLabels:
          app: payment-service
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
    - podSelector:
        matchLabels:
          app: message-queue
    ports:
    - protocol: TCP
      port: 9092
  # Order service CANNOT talk to: monitoring service, any external IP, other namespaces
```

### Private Connectivity

#### VPC Peering
Direct network connection between two VPCs using private IPs:

```
VPC-A (Production)  ←──peering──→  VPC-B (Shared Services)
10.0.0.0/16                         172.16.0.0/16
```

```hcl
resource "aws_vpc_peering_connection" "prod_to_shared" {
  vpc_id      = aws_vpc.prod.id
  peer_vpc_id = aws_vpc.shared.id
  auto_accept = false  # require manual acceptance for security
}
```

#### AWS PrivateLink / GCP Private Service Connect / Azure Private Link
Expose services privately without traversing the public internet:

```hcl
# PrivateLink: access service via private IP in your VPC
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]
}
# Traffic to S3 never leaves AWS network, no internet gateway needed
```

#### Transit Gateway (Hub-and-Spoke Architecture)
Central networking hub connecting multiple VPCs and on-premises:

```
         ┌──────────────┐
         │  Transit GW  │
         └───┬───┬───┬──┘
        ┌────┘   │   └────┐
    ┌───▼──┐ ┌──▼──┐ ┌───▼──┐
    │ VPC-A│ │VPC-B│ │VPC-C │
    └──────┘ └─────┘ └──────┘
         │                  │
    ┌────▼──┐          ┌───▼────┐
    │  On-  │          │Shared  │
    │ Prem  │          │Services│
    └───────┘          └────────┘
```

### Network Monitoring and Detection

#### VPC Flow Logs
Capture metadata about every network flow:

```
2 123456789010 eni-0a1b2c3d 10.0.1.100 10.0.2.50 443 45678 6 25 4500 1620140766 1620140826 ACCEPT OK
|                   |           |           |        |   |      | |  |    |         |         |       |
version        interface      src         dst    src  dst  prot |  bytes   start     end     action  log
                                                               packets
```

```python
# Analyze flow logs for suspicious patterns
import boto3

def analyze_flow_logs(bucket, prefix):
    # Detect: SSH traffic from unexpected IPs, excessive rejected flows,
    # flows to known malicious IPs, large data transfers to unusual destinations
    pass
```

#### Intrusion Detection (IDS/IPS)
```bash
# Snort (open-source IDS/IPS)
snort -A console -q -c /etc/snort/snort.conf -i eth0

# Suricata (multi-threaded IDS/IPS)
suricata -c /etc/suricata/suricata.yaml -i eth0
```

## Layman's Explanation

### The Office Building Security System
A modern office building's physical security perfectly illustrates network security:

**Network Segmentation**: The building has different zones. The lobby (public subnet) is open to visitors. The office floors (private subnet) require a key card after lobby hours. The server room (database subnet) requires special access and a separate key card, and you can only get there by first passing through the office floor.

**Security Groups**: Each door has a lock (security group rule). The main entrance allows anyone during business hours (HTTPS from 0.0.0.0/0). The server room door only opens for IT staff (port 5432 only from app security group). You can't even find the server room door from the lobby.

**NACLs**: The building's security desk logs everyone who enters (stateless tracking). They check both entry AND exit—if you came in through the lobby, you must also show your badge when leaving the server room (stateless: both directions are checked separately).

**Micro-segmentation**: Even within the office floor, the finance team's cubicles have an extra locked door. Even if a visitor somehow gets past the lobby, they can't access finance without a finance badge.

**Flow Logs**: Every door records who went through it, at what time, and in which direction. Security reviews these logs weekly to spot anomalies—like someone accessing the server room at 3 AM on a Saturday.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **VPC/VNet Design**: CIDR block sizing (too small requires rework; overlapping blocks break peering), subnet strategy (public/private/data across AZs), and IP address management (BYOIP, static vs dynamic). Poor initial design becomes a multi-year migration project.
- **Egress Controls**: Most cloud security focuses on ingress. Egress is equally critical—a compromised instance must be prevented from exfiltrating data. Default-open egress is a security anti-pattern. Use VPC endpoints and explicit egress rules.
- **East-West Traffic**: In microservices, 80% of traffic is service-to-service (east-west), not client-to-server (north-south). Security groups and network policies must be as granular for east-west as for north-south.
- **Hybrid Connectivity**: VPN vs Direct Connect/Cloud Interconnect/ExpressRoute. VPN is cheaper but lower throughput and higher latency. Direct Connect is higher throughput and more reliable but requires physical connection. Both routes must be secured (encrypted, authenticated).

### Business Impact
- **Breach Containment**: The 2013 Target breach (40M credit cards) happened because attackers pivoted from the HVAC vendor's network to the payment system. Proper network segmentation would have isolated the HVAC network from the payment network.
- **Compliance**: PCI DSS requires network segmentation to isolate the Cardholder Data Environment (CDE) from the rest of the network. SOC 2 requires network controls (CC6.6). HIPAA requires access controls on electronic Protected Health Information (ePHI).
- **Cost Optimization**: VPC endpoints (PrivateLink) eliminate NAT gateway data processing charges for traffic to AWS services. Traffic between AZs in the same region costs $0.01/GB—network design impacts cloud bills at scale.
- **Operational Security**: Bastion hosts and Session Manager (instead of SSH keys) eliminate standing access to instances. No open SSH ports on any instance, ever. This closes one of the most common cloud attack vectors.

## On-Premises Examples

### pfSense / OPNsense Firewall
```
WAN Interface (ISP)
     │
  ┌──▼──────────┐
  │   pfSense    │  <── Firewall Rules, NAT, VPN, IDS/IPS
  └──┬──────┬───┘
     │      │
  ┌──▼──┐ ┌─▼───┐
  │ LAN │ │ DMZ │  <── Public-facing servers (isolated)
  └─────┘ └─────┘
```

```bash
# pfSense rules
# Allow HTTPS from anywhere to DMZ web server
pass in on WAN proto tcp from any to 203.0.113.10 port 443

# Block all other inbound to DMZ
block in on WAN to DMZ

# Allow LAN to access internet
pass out on WAN from LAN to any

# Block LAN from accessing DMZ (except specific management ports)
block in on LAN to DMZ
```

### iptables Firewall Configuration
```bash
#!/bin/bash
# Default: drop everything
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow SSH from internal network only
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Allow HTTPS from anywhere
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow HTTP (redirect to HTTPS)
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Allow application port from specific subnet
iptables -A INPUT -p tcp --dport 8080 -s 10.0.1.0/24 -j ACCEPT

# Log and drop everything else
iptables -A INPUT -j LOG --log-prefix "IPTables-Dropped: " --log-level 4
iptables -A INPUT -j DROP
```

### WireGuard VPN (Modern, Simple VPN)
```ini
# /etc/wireguard/wg0.conf (server)
[Interface]
Address = 10.100.0.1/24
PrivateKey = SERVER_PRIVATE_KEY
ListenPort = 51820

[Peer]
# Office network
PublicKey = OFFICE_PEER_PUBLIC_KEY
AllowedIPs = 10.100.0.2/32, 10.0.0.0/8  # route office subnet through VPN

[Peer]
# Developer laptop
PublicKey = DEV_LAPTOP_PUBLIC_KEY
AllowedIPs = 10.100.0.3/32
```

```bash
# Start WireGuard
wg-quick up wg0

# Client configuration
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.100.0.3/24

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.company.com:51820
AllowedIPs = 10.0.0.0/8  # Route all internal traffic through VPN
PersistentKeepalive = 25
```

### OpenVPN Access Server
```bash
# Access Server with LDAP authentication
# Only specific users can access specific subnets
# Multi-factor authentication via TOTP
```

## AWS Examples

### VPC Security Architecture
```hcl
# Full VPC with security controls
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  enable_dns_support   = true
  enable_dns_hostnames = true
}

# Internet Gateway (only for public subnets)
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

# NAT Gateway (for private outbound internet)
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
}

# VPC Flow Logs to S3 + CloudWatch
resource "aws_flow_log" "main" {
  traffic_type         = "ALL"  # Capture accepted + rejected
  vpc_id               = aws_vpc.main.id
  log_destination_type = "s3"
  log_destination      = aws_s3_bucket.flow_logs.arn

  destination_options {
    file_format                = "parquet"
    hive_compatible_partitions = true
    per_hour_partition         = true
  }
}

# VPC Endpoints (private access to AWS services)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = concat(aws_route_table.private[*].id, aws_route_table.database[*].id)
}

resource "aws_vpc_endpoint" "kms" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.kms"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
}
```

### AWS Network Firewall (Managed IDS/IPS)
```hcl
resource "aws_networkfirewall_firewall" "main" {
  name                = "vpc-firewall"
  vpc_id              = aws_vpc.main.id
  subnet_mapping {
    subnet_id = aws_subnet.firewall.id
  }

  firewall_policy_arn = aws_networkfirewall_firewall_policy.main.arn
}

resource "aws_networkfirewall_firewall_policy" "main" {
  name = "main-policy"

  firewall_policy {
    stateless_default_actions          = ["aws:forward_to_sfe"]
    stateless_fragment_default_actions = ["aws:forward_to_sfe"]

    stateful_rule_group_reference {
      resource_arn = aws_networkfirewall_rule_group.block_malicious.arn
    }

    stateful_rule_group_reference {
      resource_arn = aws_networkfirewall_rule_group.allow_web.arn
    }
  }
}

resource "aws_networkfirewall_rule_group" "block_malicious" {
  name     = "block-malicious-domains"
  type     = "STATEFUL"
  capacity = 100

  rule_group {
    stateful_rule_options {
      rule_order = "STRICT_ORDER"
    }

    stateful_rule {
      action = "DROP"
      header {
        destination      = "ANY"
        destination_port = "ANY"
        direction        = "ANY"
        protocol         = "TCP"
        source           = "ANY"
        source_port      = "ANY"
      }
      rule_option {
        keyword = "sid:1; rev:1; msg:'Block known malicious domains'; "
      }
    }
  }
}
```

### AWS Systems Manager Session Manager (No SSH)
```hcl
# No open SSH ports on EC2 instances
# Use Session Manager for shell access via IAM

resource "aws_iam_role" "ssm_role" {
  name = "ssm-instance-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.ssm_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Access via AWS CLI (no SSH key needed, IAM-controlled):
# aws ssm start-session --target i-0a1b2c3d4e5f6g7h8
```

## GCP Examples

### VPC Firewall Rules
```bash
# Hierarchical firewall: can be applied at VPC, folder, or organization level
# Deny all inbound by default (implicit)
gcloud compute firewall-rules create allow-https \
  --network=main-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server

# Allow app tier to database tier
gcloud compute firewall-rules create allow-app-to-db \
  --network=main-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:5432 \
  --source-tags=app-server \
  --target-tags=db-server

# Egress restriction: deny all outbound except VPC endpoints
gcloud compute firewall-rules create deny-all-egress \
  --network=main-vpc \
  --direction=EGRESS \
  --priority=65535 \
  --action=DENY \
  --rules=all \
  --destination-ranges=0.0.0.0/0
```

### VPC Service Controls (Data Perimeter)
```bash
# Create a service perimeter around sensitive data
gcloud access-context-manager perimeters create data-perimeter \
  --title="Data Perimeter" \
  --resources=projects/123456789 \
  --restricted-services=storage.googleapis.com,bigquery.googleapis.com \
  --ingress-policies=@ingress.yaml \
  --egress-policies=@egress.yaml

# This prevents data exfiltration: even if credentials are stolen,
# data cannot be copied outside the perimeter
```

### Cloud NAT (for private instances)
```bash
gcloud compute routers nats create nat-gateway \
  --router=main-router \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --region=us-central1
```

### Packet Mirroring (Traffic Inspection)
```bash
gcloud compute packet-mirrorings create traffic-inspection \
  --collector-ilb=collector-forwarding-rule \
  --network=main-vpc \
  --mirrored-subnets=subnet-1,subnet-2 \
  --filter-direction=INGRESS
```

## Azure Examples

### Azure Virtual Network (VNet)
```bash
# Hub-Spoke network architecture
az network vnet create \
  --name hub-vnet \
  --resource-group myResourceGroup \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name GatewaySubnet \
  --subnet-prefix 10.0.0.0/24

az network vnet create \
  --name spoke-prod-vnet \
  --resource-group myResourceGroup \
  --address-prefixes 10.1.0.0/16

# VNet peering
az network vnet peering create \
  --name hub-to-spoke \
  --resource-group myResourceGroup \
  --vnet-name hub-vnet \
  --remote-vnet spoke-prod-vnet \
  --allow-vnet-access

# Network Security Group (NSG) - equivalent to security group
az network nsg create \
  --name app-nsg \
  --resource-group myResourceGroup

az network nsg rule create \
  --name AllowHTTPS \
  --nsg-name app-nsg \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443 \
  --source-address-prefixes Internet
```

### Azure Firewall (Managed NVA)
```bash
az network firewall create \
  --name azure-firewall \
  --resource-group myResourceGroup \
  --location eastus

az network firewall network-rule create \
  --firewall-name azure-firewall \
  --resource-group myResourceGroup \
  --collection-name "WebAccess" \
  --name "AllowHTTPS" \
  --protocols "HTTPS" \
  --source-addresses "10.1.0.0/16" \
  --destination-addresses "*" \
  --destination-ports 443 \
  --action Allow \
  --priority 100
```

### Azure DDoS Protection + NSG Flow Logs
```bash
# Network Watcher flow logs
az network watcher flow-log create \
  --location eastus \
  --resource-group NetworkWatcherRG \
  --name app-flow-logs \
  --nsg app-nsg \
  --storage-account flowlogsstorage \
  --enabled true \
  --retention 90 \
  --format JSON
```

## Summary Decision Matrix

| Network Security Control | Purpose | Granularity | Provider Solutions |
|-------------------------|---------|-------------|-------------------|
| Security Groups / NSGs | Resource-level firewall | Per-instance/ENI | AWS SG, Azure NSG, GCP firewall rules |
| Network ACLs | Subnet-level filtering | Per subnet | AWS NACL, Azure NSG (subnet), GCP firewall (subnet) |
| VPC/VNet Segmentation | Environment isolation | Per VPC/VNet | AWS VPC, Azure VNet, GCP VPC |
| Network Firewall / IDS/IPS | Deep packet inspection, threat detection | Per VPC/VNet | AWS Network Firewall, Azure Firewall, GCP Cloud IDS |
| Private Connectivity | Service access without internet | Per endpoint | PrivateLink, Private Service Connect, Azure Private Link |
| Flow Logs | Traffic visibility | Per ENI/subnet/VPC | VPC Flow Logs, NSG Flow Logs, VPC Flow Logs |
| DDoS Protection | Volumetric attack absorption | Per resource/VNet | Shield, Cloud Armor, Azure DDoS Protection |

Network security is the foundation on which all other security layers are built. Proper segmentation, granular firewall rules, private connectivity, and traffic monitoring create a hardened environment where even if one layer fails (compromised credentials, vulnerable application), the network layer limits the blast radius. A solution architect must design the network as if it's already compromised—every segment, every security group rule, and every flow log is a control that contains and detects breaches.
