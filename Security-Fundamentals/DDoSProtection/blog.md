# DDoS Protection

## Introduction
Distributed Denial of Service (DDoS) attacks are among the oldest and most persistent threats on the internet. They attempt to overwhelm a target system, service, or network with a flood of traffic from multiple compromised sources, rendering it unavailable to legitimate users. The first major DDoS attack was recorded in 1999, and attacks have grown exponentially—the largest recorded attack exceeded 3.47 Tbps (Microsoft, 2021) and 71 million requests per second (Google, 2022).

DDoS protection has evolved from simple rate limiting at the server level to a multi-layered, globally distributed defense architecture combining anycast networks, traffic scrubbing centers, and machine learning-based anomaly detection. For solution architects, DDoS resilience must be designed into the system from day one—retrofitting it after an attack has already begun is too late.

## Definition
A **DDoS Attack** is a malicious attempt to disrupt normal traffic to a targeted server, service, or network by overwhelming the target or its surrounding infrastructure with a flood of internet traffic from many different, often compromised, sources (botnets).

**DDoS Protection** is a combination of strategies, services, and technologies designed to detect, absorb, and mitigate DDoS attacks while maintaining service availability for legitimate users.

### Attack Categories

#### Volumetric Attacks (Layer 3/4)
Overwhelm network bandwidth with massive amounts of traffic:
- **UDP Flood**: Massive UDP packets to random ports, consuming bandwidth
- **DNS Amplification**: Small DNS query elicits large response (up to 50x amplification), spoofed source IP hits victim
- **NTP Amplification**: Exploits NTP `monlist` command for up to 556x amplification
- **SYN Flood**: Exhausts server's connection table with half-open TCP connections
- **ICMP Flood (Ping Flood)**: Overwhelms with ICMP Echo Request packets

#### Protocol Attacks (Layer 3/4)
Exploit weaknesses in network protocols to consume server or intermediate resources:
- **SYN-ACK Flood**: Floods with responses to nonexistent SYN requests
- **Ping of Death**: Malformed ICMP packets causing buffer overflow
- **Smurf Attack**: ICMP requests with spoofed source IP broadcast to network

#### Application-Layer Attacks (Layer 7)
Target specific application resources with seemingly legitimate requests:
- **HTTP Flood**: Massive GET/POST requests to resource-intensive endpoints
- **Slowloris**: Opens many connections, sends partial HTTP headers very slowly, exhausting connection pools
- **REST API Exhaustion**: Targeting expensive API endpoints (search, report generation)
- **GraphQL Query Depth Attacks**: Deeply nested queries consuming server CPU

## Concept Explanation

### Defense Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     DDoS DEFENSE LAYERS                      │
│                                                             │
│  Layer 1: Anycast DNS + Edge Routing                        │
│    └── Distribute traffic across global PoPs                 │
│                                                             │
│  Layer 2: Traffic Scrubbing Centers                         │
│    └── Filter attack traffic, forward clean traffic          │
│                                                             │
│  Layer 3: CDN / Reverse Proxy / Load Balancer               │
│    └── Rate limiting, connection limits, IP reputation       │
│                                                             │
│  Layer 4: WAF (Application Layer Protection)                │
│    └── Bot detection, request inspection, challenge/response │
│                                                             │
│  Layer 5: Origin Shield / Backend Protection                │
│    └── SYN cookies, connection pools, auto-scaling           │
│                                                             │
│  Layer 6: Monitoring + Auto-Response                        │
│    └── Anomaly detection, automatic mitigation triggers      │
└─────────────────────────────────────────────────────────────┘
```

### Key Defense Techniques

#### Anycast Network Distribution
Instead of a single IP address resolving to a single server, anycast routes traffic to the nearest of many geographically distributed nodes. This scatters attack traffic across hundreds of locations, making it mathematically impossible to overwhelm any single point.

```
Without Anycast:
  All traffic → Single IP → Single server → Overwhelmed

With Anycast:
  All traffic → Single IP → Nearest PoP (of 300+) → Attack scattered
  Tokyo attacker → Tokyo PoP (handles its share)
  London attacker → London PoP (handles its share)
  No single PoP sees the full attack volume
```

#### Traffic Scrubbing
Malicious traffic is separated from legitimate traffic at specialized scrubbing centers:

1. Traffic is routed through scrubbing center (via BGP route change or always-on)
2. Scrubbing center analyzes traffic patterns, applies filters
3. Known attack patterns (UDP floods, DNS amplification response without query) are dropped
4. Clean traffic is forwarded to the origin, often through GRE tunnel or direct connect
5. Legitimate users experience minimal or no impact

#### SYN Cookies (SYN Flood Defense)
```python
# Kernel-level: SYN cookie implementation
# Instead of allocating connection state on SYN:
# 1. Encode connection parameters into the SYN-ACK sequence number
# 2. Only allocate state when ACK is received (proving real client)
# 3. Sequence number is cryptographically verified

# Linux kernel config
echo 1 > /proc/sys/net/ipv4/tcp_syncookies
```

#### Rate Limiting and Throttling
```nginx
# Connection-based limits
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
limit_conn conn_limit 10;  # Max 10 concurrent connections per IP

# Rate-based limits
limit_req_zone $binary_remote_addr zone=req_limit:10m rate=30r/s;
limit_req zone=req_limit burst=50 nodelay;

# Client timeout tuning (Slowloris defense)
client_body_timeout 10s;
client_header_timeout 10s;
keepalive_timeout 5s;
send_timeout 10s;
```

#### JavaScript/CAPTCHA Challenges
Embedded in CDN/WAF responses to distinguish humans from bots:

```html
<!-- Cloudflare/CloudFront challenge page -->
<html>
<body>
  <script>
    // Complex calculation that bots can't easily execute
    // Proves browser is real
    var challenge = solve_challenge();
    document.cookie = "cf_clearance=" + challenge;
    location.reload();
  </script>
  <noscript>Enable JavaScript and cookies to continue</noscript>
</body>
</html>
```

#### Blackhole Routing (Last Resort)
Route all traffic to target IP into a null route (dropped). Effective against volumetric attacks but drops ALL traffic—legitimate and malicious. Used only as a last resort when other mitigations fail.

```bash
# BGP blackhole / RTBH (Remotely Triggered Black Hole)
# Router configuration
ip route 203.0.113.0 255.255.255.255 Null0
# Traffic to 203.0.113.0 is dropped
```

### Attack Detection

**Baseline Traffic Profiling**: Establish normal traffic patterns (requests per second, bandwidth, unique IPs, geolocation distribution). Anomaly detection triggers when deviations exceed thresholds:

```python
import numpy as np
from collections import deque

class DDoSDetector:
    def __init__(self, window_size=60, threshold_std=3):
        self.window = deque(maxlen=window_size)
        self.threshold_std = threshold_std
    
    def record_metric(self, rps):
        self.window.append(rps)
    
    def is_attack(self, current_rps):
        if len(self.window) < 30:
            return False
        mean = np.mean(self.window)
        std = np.std(self.window)
        return current_rps > mean + self.threshold_std * std

# Usage
detector = DDoSDetector()
# Every second: detector.record_metric(current_rps)
# if detector.is_attack(current_rps): trigger_mitigation()
```

## Layman's Explanation

### The Highway Analogy
Imagine your store (web server) sits on a highway (internet connection). Normally, 100 customers per hour drive to your store—the highway handles this easily.

**DDoS Attack**: A malicious competitor hires 10,000 people (botnet) to drive their cars onto the highway at the same time, creating a massive traffic jam. Legitimate customers can't reach your store because the highway is completely gridlocked. Your store is fine—it's the road (network) that's overwhelmed. OR, the attackers reach your store, rush in, and your staff can't serve real customers (application layer attack).

**DDoS Protection Layers**:
1. **Anycast**: Instead of one store on one road, you have stores on 300 different roads worldwide. Attackers spread across all roads—none gets fully jammed.
2. **Traffic Scrubbing**: There's a checkpoint before your store. Suspicious vehicles (UDP flood packets) are diverted to an inspection lot and never reach your store.
3. **Rate Limiting**: The bouncer only lets 30 people per minute into your store, no matter how many are outside.
4. **Challenge Page**: The bouncer stops each person and asks "Are you a robot?" Only humans can answer correctly.
5. **Blackhole**: In the worst case, you close the entire road. Nobody reaches the store, but the traffic jam can't spread to neighboring roads.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Always-On vs On-Demand Protection**: Always-on (AWS Shield Advanced, Cloudflare Business+) provides constant protection with sub-second mitigation. On-demand (triggering scrubbing when attack detected) is cheaper but has a gap between detection and mitigation—often 5-15 minutes of downtime before kicking in.
- **Origin IP Exposure**: If attackers discover the origin IP (behind CDN/load balancer), they can bypass all DDoS protection and attack directly. Origin IP must be kept secret—never published in DNS, never referenced in error pages, changed if ever exposed.
- **Cost Management**: DDoS protection services charge by clean traffic volume or fixed subscription. During an attack, a 100 Gbps attack could cost thousands per hour in bandwidth if not on an unmetered plan. Fixed-cost subscription (Shield Advanced $3,000/month + data transfer) provides cost certainty.
- **Application-Layer Defense**: Volumetric defenses don't stop Layer 7 attacks that look like legitimate traffic. An attacker requesting `/api/generate-report` 500 times/second with valid JWTs requires application-level intelligence (WAF rate limiting per endpoint, per user, not just per IP).

### Business Impact
- **Revenue Loss**: Average DDoS attack costs $218,000 (Kaspersky). For e-commerce, every minute of downtime during peak periods can cost $10,000+ in lost sales. Gaming and financial services lose more due to SLA penalties and customer churn.
- **Ransom DDoS**: Attackers increasingly demand ransom payments (often in Bitcoin) to stop attacks. Organizations without protection are forced to either pay (encouraging more attacks) or sustain business-crippling downtime.
- **Reputation**: Extended downtime during an attack signals poor security maturity to customers, partners, and investors. Particularly damaging for B2B SaaS where uptime SLA is a contractual obligation.
- **Operational Exhaustion**: DDoS response without automated protection consumes entire SRE/security teams for days—coordinating with ISPs, writing firewall rules, and spinning up additional infrastructure.

## On-Premises Examples

### iptables/nftables Rate Limiting
```bash
# Limit incoming SYN packets
iptables -A INPUT -p tcp --syn -m limit --limit 50/s --limit-burst 100 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# Block excessive connections from single IP
iptables -A INPUT -p tcp --dport 443 -m connlimit --connlimit-above 50 --connlimit-mask 32 -j DROP

# Rate limit ICMP
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

### Fail2ban (Intrusion Prevention)
```ini
# /etc/fail2ban/jail.local
[nginx-http-auth]
enabled = true
port    = http,https
filter  = nginx-http-auth
logpath = /var/log/nginx/error.log
maxretry = 5
findtime = 60
bantime  = 600
```

### Linux Kernel Tuning
```bash
# Increase SYN backlog
sysctl -w net.ipv4.tcp_max_syn_backlog=2048

# Enable SYN cookies
sysctl -w net.ipv4.tcp_syncookies=1

# Reduce SYN-ACK retries
sysctl -w net.ipv4.tcp_synack_retries=2

# Increase somaxconn (socket listen backlog)
sysctl -w net.core.somaxconn=1024

# Enable TCP Fast Open for faster SYN handling
sysctl -w net.ipv4.tcp_fastopen=3
```

### BGP Flowspec (Advanced Traffic Filtering)
```bash
# BGP Flowspec to drop traffic matching specific characteristics
# Used by large networks/ISPs for automated DDoS mitigation
router bgp 65000
  address-family ipv4 flowspec
    # Drop UDP traffic to port 53 from any source to 203.0.113.1
    # Essentially filters DNS amplification attacks
```

### HAProxy Anti-DDoS Configuration
```haproxy
frontend web
    bind *:80,*:443
    
    # Connection rate limiting
    stick-table type ip size 100k expire 30s store conn_rate(10s)
    tcp-request connection track-sc0 src
    tcp-request connection reject if { sc_conn_rate(0) gt 100 }
    
    # HTTP rate limiting
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 50 }
    
    # Slowloris protection
    timeout http-request 5s
    timeout client 30s
```

## AWS Examples

### AWS Shield Standard (Automatic, Free)
```hcl
# Shield Standard is automatically enabled for all AWS customers
# Protects against common Layer 3/4 attacks:
# - SYN/ACK floods, UDP floods, reflection attacks
# - Automatically applied at AWS network edge
# No configuration needed—automatic protection for CloudFront, ALB, Route 53
```

### AWS Shield Advanced (Enhanced, Paid)
```hcl
resource "aws_shield_protection" "alb" {
  name         = "app-alb-protection"
  resource_arn = aws_lb.app.arn
}

resource "aws_shield_protection" "cloudfront" {
  name         = "cdn-protection"
  resource_arn = aws_cloudfront_distribution.cdn.arn
}

resource "aws_shield_protection" "route53" {
  name         = "dns-protection"
  resource_arn = "arn:aws:route53:::hostedzone/${aws_route53_zone.main.id}"
}

# Shield Advanced features:
# - 24/7 access to AWS DDoS Response Team (DRT)
# - Cost protection (refunds for scale-out during attacks)
# - Advanced real-time metrics and attack forensics
# - Integration with AWS WAF for automated Layer 7 mitigation
```

### Shield Advanced + WAF Automated Response
```hcl
resource "aws_shield_protection" "app" {
  name         = "app-full-protection"
  resource_arn = aws_lb.app.arn

  application_layer_automatic_response_configuration {
    action {
      block {}
    }
    status = "ENABLED"
  }
  # When Shield detects application-layer DDoS:
  # 1. Automatically creates WAF rate-based rule
  # 2. Blocks the attack IPs for configured duration
  # 3. Notifies via SNS/Security Hub
}
```

### CloudFront + ALB Architecture
```
[User] → [Route 53 (anycast DNS)] → [CloudFront (600+ PoPs)]
                                        │
                        ┌───────────────┴───────────────┐
                        │                               │
                  Cache HIT → Served from edge     Cache MISS
                                                       │
                                              [ALB (protected)]
                                                       │
                                                    [ECS/EKS]
```

```hcl
# Origin access: only CloudFront can reach ALB
resource "aws_security_group" "alb" {
  name = "alb-sg"
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = []  # Only CloudFront prefix list
    prefix_list_ids = [
      data.aws_ec2_managed_prefix_list.cloudfront.id
    ]
  }
}

# Use AWS-managed CloudFront prefix list
data "aws_ec2_managed_prefix_list" "cloudfront" {
  name = "com.amazonaws.global.cloudfront.origin-facing"
}
```

### Route 53 DNS-Level Protection
```python
# Shuffle sharding: distribute resources across many virtual shards
# If one shard is attacked, only that shard's users are affected

# Route 53 resolves to different endpoints based on user characteristics
# Reducing blast radius of any single attack
```

## GCP Examples

### Cloud Armor Advanced (DDoS + WAF)
```bash
# Enable Cloud Armor Adaptive Protection (ML-based DDoS detection)
gcloud compute security-policies update app-waf-policy \
  --enable-ml-based-detection

# The ML model:
# 1. Learns normal traffic baseline over weeks
# 2. Detects anomalies (traffic pattern changes) in real-time
# 3. Generates suggested WAF rules to block attack traffic
# 4. Can auto-deploy suggested rules (with manual approval or automated)

# Configure adaptive protection auto-deploy
gcloud compute security-policies update app-waf-policy \
  --adaptive-protection-auto-deploy \
  --adaptive-protection-auto-deploy-alert-only=false \
  --adaptive-protection-auto-deploy-load-threshold=0.5 \
  --adaptive-protection-auto-deploy-confidence-threshold=0.8 \
  --adaptive-protection-auto-deploy-impacted-baseline-threshold=0.01 \
  --adaptive-protection-auto-deploy-expiration-sec=7200
```

### Google Cloud Armor + Load Balancing
```bash
# Global anycast load balancing (Google's global network)
# Inherent volumetric DDoS absorption: traffic is distributed
# across Google's edge, which has massive capacity

gcloud compute url-maps create web-map \
  --default-service backend-service-web

gcloud compute backend-services create backend-service-web \
  --load-balancing-scheme=EXTERNAL \
  --protocol=HTTP \
  --security-policy=app-waf-policy \
  --global
```

### Cloud CDN for Absorption
```bash
gcloud compute backend-buckets create static-backend \
  --gcs-bucket-name=my-static-bucket \
  --enable-cdn

# During volumetric attack:
# 1. Cloud CDN serves cached content without hitting origin
# 2. Cache hit ratio protects origin from overload
# 3. Signed URLs / signed cookies restrict access to legitimate users only
```

## Azure Examples

### Azure DDoS Protection
```bash
# Enable DDoS Protection Standard on VNet
az network ddos-protection create \
  --name myDdosProtectionPlan \
  --resource-group myResourceGroup \
  --location eastus

az network vnet update \
  --name myVNet \
  --resource-group myResourceGroup \
  --ddos-protection-plan myDdosProtectionPlan \
  --ddos-protection true

# DDoS Protection Standard features:
# - Always-on traffic monitoring and auto-mitigation
# - Adaptive tuning based on application traffic patterns
# - Integration with Azure Monitor for alerts and metrics
# - Cost guarantee (credits for scale-out during attacks)
# - Access to DDoS rapid response team
```

### Azure DDoS Protection Telemetry
```python
from azure.mgmt.monitor import MonitorManagementClient

# Monitor DDoS metrics
metrics_client = MonitorManagementClient(credential, subscription_id)

metrics = metrics_client.metrics.list(
    resource_uri="/subscriptions/.../virtualNetworks/myVNet",
    metricnames="IfUnderDDoS,AttackBitsPerSecond,ActiveFlows",
    timespan="PT1H",
    interval="PT1M"
)
```

### Azure Front Door (Global DDoS Absorption)
```bash
az afd profile create \
  --profile-name global-cdn \
  --resource-group myResourceGroup \
  --sku Premium_AzureFrontDoor
  
# Front Door provides:
# - Global anycast distribution (100+ PoPs)
# - Automatic DDoS absorption at Azure network edge
# - TLS termination at edge
# - WAF integration for Layer 7 protection
```

### Cross-Cloud DDoS Strategy
```yaml
# Multi-layered defense combining all providers:
# Layer 1: Cloudflare (DNS proxy + CDN + DDoS scrubbing)
# Layer 2: AWS Shield Advanced + WAF (application resources)
# Layer 3: Azure DDoS Protection + Front Door (failover)
# Result: No single provider is the single point of failure
```

## Summary Decision Matrix

| DDoS Protection Layer | What It Defends Against | Provider Solutions | Latency Impact |
|----------------------|------------------------|-------------------|----------------|
| Anycast DNS/Distribution | Volumetric (L3/4) floods | Route 53, Cloud CDN, Azure Front Door, Cloudflare | None (improves latency) |
| Traffic Scrubbing | Reflection/Amplification attacks | Shield Advanced, Cloud Armor, Azure DDoS Protection | < 1ms (in-path) |
| WAF (Layer 7) | HTTP floods, Slowloris, API attacks | AWS WAF, Cloud Armor, Azure WAF v2 | < 1ms per rule evaluation |
| Rate Limiting | API exhaustion, brute force, scraping | WAF rate rules, API Gateway throttling | Negligible (counter-based) |
| SYN Cookies | SYN floods | Kernel-level (enabled by default in modern OS) | Slight CPU increase |
| Auto-scaling | Gradual traffic increases | EC2 Auto Scaling, GCE MIG, VMSS | Varies (1-5 min scale-up) |

DDoS protection is not an add-on—it's a fundamental architectural requirement for any internet-facing system. The defense must be layered: anycast distribution for volumetric attacks, WAF for application-layer attacks, rate limiting for API exhaustion, and always-on protection for instant mitigation. A solution architect must assume attacks will happen and design the system to absorb them without impacting legitimate users. The key principle: never expose a single point that, if overwhelmed, takes down the entire system.
