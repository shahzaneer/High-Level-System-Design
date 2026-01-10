# Azure Load Balancer Examples

---

## 1. Azure Application Gateway

* **Layer:** L7 (HTTP/HTTPS, WebSocket)
* **Algorithms:**

  * Round robin
  * Cookie-based session affinity
  * Connection draining

**Key Features:**

* WAF: Integrated Web Application Firewall (OWASP rules)
* Autoscaling: v2 SKU supports autoscaling
* URL-based routing: Path and host-based
* SSL/TLS: Offload and end-to-end SSL
* Rewrite: HTTP header and URL rewrite
* Redirection: HTTP to HTTPS, URL redirection
* Multiple site hosting: Up to 100 sites on one gateway
* Azure Private Link: Private connectivity

**Configuration Example (ARM template excerpt):**

```json
{
  "type": "Microsoft.Network/applicationGateways",
  "apiVersion": "2021-05-01",
  "name": "myAppGateway",
  "location": "eastus",
  "properties": {
    "sku": {
      "name": "WAF_v2",
      "tier": "WAF_v2",
      "capacity": 2
    },
    "gatewayIPConfigurations": [...],
    "frontendIPConfigurations": [...],
    "frontendPorts": [...],
    "backendAddressPools": [
      {
        "name": "webBackend",
        "properties": {
          "backendAddresses": [
            {"ipAddress": "10.0.1.4"},
            {"ipAddress": "10.0.1.5"}
          ]
        }
      },
      {
        "name": "apiBackend",
        "properties": {
          "backendAddresses": [
            {"ipAddress": "10.0.2.4"},
            {"ipAddress": "10.0.2.5"}
          ]
        }
      }
    ],
    "httpListeners": [...],
    "requestRoutingRules": [
      {
        "name": "apiRule",
        "properties": {
          "ruleType": "PathBasedRouting",
          "httpListener": {"id": "[resourceId('httpListener')]"},
          "urlPathMap": {"id": "[resourceId('urlPathMap')]"}
        }
      }
    ],
    "urlPathMaps": [
      {
        "name": "pathMap",
        "properties": {
          "defaultBackendAddressPool": {"id": "[resourceId('webBackend')]"},
          "pathRules": [
            {
              "name": "apiPath",
              "paths": ["/api/*"],
              "backendAddressPool": {"id": "[resourceId('apiBackend')]"}
            }
          ]
        }
      }
    ],
    "webApplicationFirewallConfiguration": {
      "enabled": true,
      "firewallMode": "Prevention",
      "ruleSetType": "OWASP",
      "ruleSetVersion": "3.2"
    }
  }
}
```

**When to Use:**

* Web applications
* Need WAF protection
* Multi-site hosting
* Azure-native solution
* End-to-end encryption required

**Real-World Scenario:**

* Corporate web portal:

  * `portal.company.com` → Portal backend
  * `api.company.com` → API backend
  * WAF blocks SQL injection and XSS attacks
  * Autoscales during business hours (8 AM - 6 PM)

---

## 2. Azure Load Balancer (Standard)

* **Layer:** L4 (TCP, UDP)
* **Algorithms:**

  * 5-tuple hash (source IP, source port, destination IP, destination port, protocol)
  * Source IP affinity (2-tuple or 3-tuple)

**Types:**

* Public: Internet-facing
* Internal: Private, within VNet

**Key Features:**

* High availability: Zone-redundant
* Health probes: TCP, HTTP, HTTPS
* Outbound rules: Control outbound connections (SNAT)
* HA Ports: Load balance all ports (useful for NVAs)
* Backend pools: VMs, VMSS, availability sets
* Cross-region: Standard SKU supports cross-region

**Configuration Example (Azure CLI):**

```bash
# Create public IP
az network public-ip create \
    --resource-group myRG \
    --name myPublicIP \
    --sku Standard \
    --allocation-method Static

# Create load balancer
az network lb create \
    --resource-group myRG \
    --name myLoadBalancer \
    --sku Standard \
    --public-ip-address myPublicIP \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool

# Create health probe
az network lb probe create \
    --resource-group myRG \
    --lb-name myLoadBalancer \
    --name myHealthProbe \
    --protocol tcp \
    --port 80 \
    --interval 15 \
    --threshold 2

# Create load balancing rule
az network lb rule create \
    --resource-group myRG \
    --lb-name myLoadBalancer \
    --name myHTTPRule \
    --protocol tcp \
    --frontend-port 80 \
    --backend-port 80 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe \
    --idle-timeout 15 \
    --enable-tcp-reset true

# Add VMs to backend pool
az network nic ip-config address-pool add \
    --resource-group myRG \
    --nic-name myNic1 \
    --ip-config-name ipconfig1 \
    --lb-name myLoadBalancer \
    --address-pool myBackEndPool
```

**When to Use:**

* Non-HTTP protocols
* Need L4 performance
* Database load balancing
* Simple TCP/UDP distribution
* Cost-effective (cheaper than Application Gateway)

**Real-World Scenario:**

* SQL Server Always On: Load balance connections to SQL Server replicas

  * Internal Load Balancer
  * Backend: 3 SQL Server VMs
  * Health probe: TCP port 1433
  * Session affinity: Source IP (2-tuple)
  * Applications connect to LB IP, distributed across SQL replicas

---

## 3. Azure Front Door

* **Layer:** L7 (HTTP/HTTPS)
* **What It Is:** Global load balancer + CDN + WAF. Microsoft's edge network (similar to CloudFlare).
* **Algorithms:**

  * Latency-based routing
  * Priority-based routing
  * Weighted routing
  * Session affinity

**Key Features:**

* Global: Anycast IP, 100+ edge locations
* SSL offload: Managed certificates
* URL-based routing: Path-based
* Caching: Integrated CDN
* WAF: DDoS protection, bot protection
* Health probes: Automatic failover across regions
* Rules engine: Custom routing logic

**Configuration Example (ARM template excerpt):**

```json
{
  "type": "Microsoft.Network/frontDoors",
  "apiVersion": "2020-05-01",
  "name": "myFrontDoor",
  "location": "global",
  "properties": {
    "backendPools": [
      {
        "name": "usBackend",
        "backends": [
          {
            "address": "webapp-us.azurewebsites.net",
            "httpPort": 80,
            "httpsPort": 443,
            "priority": 1,
            "weight": 50
          }
        ],
        "healthProbeSettings": {
          "path": "/health",
          "protocol": "Https",
          "intervalInSeconds": 30
        },
        "loadBalancingSettings": {
          "sampleSize": 4,
          "successfulSamplesRequired": 2
        }
      },
      {
        "name": "euBackend",
        "backends": [
          {
            "address": "webapp-eu.azurewebsites.net",
            "priority": 1,
            "weight": 50
          }
        ]
      }
    ],
    "routingRules": [
      {
        "name": "defaultRoute",
        "frontendEndpoints": [{"id": "[resourceId('frontendEndpoint')]"}],
        "acceptedProtocols": ["Https"],
        "patternsToMatch": ["/*"],
        "routeConfiguration": {
          "backendPool": {"id": "[resourceId('usBackend')]"}
        }
      },
      {
        "name": "apiRoute",
        "patternsToMatch": ["/api/*"],
        "routeConfiguration": {
          "backendPool": {"id": "[resourceId('apiBackend')]"}
        }
      }
    ]
  }
}
```

**When to Use:**

* Global applications
* Need CDN + load balancing
* Disaster recovery across regions
* DDoS protection
* API acceleration

**Real-World Scenario:**

* Global e-commerce

  * Backends: East US, West Europe, Southeast Asia
  * User in Japan routed to Southeast Asia (lowest latency)
  * Automatic failover to East US if Southeast Asia fails
  * Static assets cached at 100+ edge locations
  * WAF blocks malicious traffic at edge

---

## 4. Azure Traffic Manager

* **Layer:** DNS-level (not a true load balancer, DNS-based routing)
* **What It Is:** DNS-based traffic router. Doesn't proxy traffic, just responds with different IPs based on routing method.
* **Algorithms:**

  * Priority: Failover pattern (primary/secondary)
  * Weighted: Percentage distribution
  * Performance: Routes to nearest endpoint (latency-based)
  * Geographic: Routes based on user location
  * Multivalue: Returns multiple healthy endpoints
  * Subnet: Routes based on source IP subnet

**When to Use:**

* DNS-level routing
* Multi-region failover
* Geographic distribution
* Can combine with other load balancers

**Real-World Scenario:**

* Global SaaS with data residency requirements:

  * EU users → EU region (GDPR compliance)
  * US users → US region
  * Asia users → Asia region
  * Traffic Manager uses Geographic routing method
  * Each region has Application Gateway for L7 load balancing

---