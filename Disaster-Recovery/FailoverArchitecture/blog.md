# Failover Architecture

## Introduction
Failover is the automatic or manual process of switching from a failed primary system to a redundant standby system. It is the mechanism that translates DR planning into operational reality. While DR patterns define the overall strategy, failover architecture defines the specific technical mechanisms—DNS-level, load balancer-level, database-level, and application-level—that detect failures and redirect traffic.

A well-designed failover architecture handles not just catastrophic regional outages but also degraded systems (high latency, partial failures) and "gray failures" (systems that are technically up but responding incorrectly). The challenge is balancing speed (automated failover) against safety (preventing false-positive failovers that cause split-brain or data corruption).

## Definition

**Failover** is the process of transferring operations from a primary system or site to a standby system or site when the primary becomes unavailable, degraded, or compromised.

**Failback** is the process of returning operations to the primary system after it has been restored.

**Key Concepts**:
- **Split-Brain**: Both primary and standby believe they are active simultaneously, potentially causing data divergence
- **Fencing**: Mechanism to ensure the old primary is truly dead before promoting the standby
- **Health Check**: The signal that triggers failover; can be simple (port open) or sophisticated (end-to-end business transaction test)

## Concept Explanation

### Failover Layers

```
Layer 1: DNS Failover (Route 53, Cloud DNS, Azure DNS)
  Detection: Health checks from multiple geographic locations
  Failover Time: 60-300 seconds (DNS TTL + propagation)
  Scope: Entire site/region
  
Layer 2: Load Balancer Failover (ALB/NLB, Cloud LB, Front Door)
  Detection: Target group/backend health checks
  Failover Time: 10-30 seconds
  Scope: Service/application tier
  
Layer 3: Database Failover (Multi-AZ, Read Replica Promotion)
  Detection: Heartbeat, consensus protocol
  Failover Time: 60-120 seconds (automated), minutes (manual)
  Scope: Database tier
  
Layer 4: Application-Level Failover (Circuit Breaker, Retry)
  Detection: Request timeouts, error responses
  Failover Time: Sub-second to seconds
  Scope: Individual service dependency
```

### DNS Failover

```python
# Multi-region health check and DNS failover
class DNSFailoverOrchestrator:
    def __init__(self):
        self.primary = RegionConfig('us-east-1', 'primary.example.com')
        self.secondary = RegionConfig('eu-west-1', 'secondary.example.com')
        self.tertiary = RegionConfig('ap-southeast-1', 'tertiary.example.com')
    
    def check_and_route(self):
        # Check regions in priority order
        for region in [self.primary, self.secondary, self.tertiary]:
            if self._is_healthy(region):
                if self._current_target != region:
                    self._update_dns(region.endpoint, ttl=60)
                    self._current_target = region
                return
        
        # All regions unhealthy
        self._serve_static_error_page()
    
    def _is_healthy(self, region):
        # Check from multiple geographic locations to avoid false positives
        results = []
        for checker in GLOBAL_HEALTH_CHECKERS:
            try:
                response = checker.get(f"https://{region.endpoint}/health", timeout=5)
                results.append(response.status_code == 200)
            except:
                results.append(False)
        
        # Require majority of checkers to confirm health
        return sum(results) > len(results) / 2
```

### Database Failover

#### Automated Multi-AZ Failover
```
Primary AZ fails:
  1. RDS detects primary failure (heartbeat lost for 30+ seconds)
  2. Initiates automatic failover to standby in different AZ
  3. Standby promoted to primary (DNS CNAME updated)
  4. Application connections broken → must reconnect
  5. Reconnection picks up new primary via DNS resolution
  6. Total failover time: 60-120 seconds
```

#### Manual Cross-Region Failover (Read Replica Promotion)
```python
def cross_region_failover(dr_region='eu-west-1'):
    rds = boto3.client('rds', region_name=dr_region)
    
    # 1. Stop writes to primary (if possible)
    # 2. Promote read replica
    rds.promote_read_replica(
        DBInstanceIdentifier='orders-db-dr'
    )
    
    # 3. Wait for promotion to complete
    waiter = rds.get_waiter('db_instance_available')
    waiter.wait(DBInstanceIdentifier='orders-db-dr')
    
    # 4. Update application configuration
    new_endpoint = rds.describe_db_instances(
        DBInstanceIdentifier='orders-db-dr'
    )['DBInstances'][0]['Endpoint']['Address']
    
    # 5. Update DNS / Parameter Store / Config
    ssm.put_parameter(
        Name='/prod/database/host',
        Value=new_endpoint,
        Type='String',
        Overwrite=True
    )
    
    # 6. Rolling restart of application to pick up new config
    restart_ecs_services(cluster='prod', services=['order-service'])
```

### Quorum-Based Leader Election (Split-Brain Prevention)

```python
class QuorumFailover:
    """
    Requires majority vote before failover to prevent split-brain.
    Uses external observers (witness nodes) across AZs.
    """
    
    def __init__(self, nodes, witness_nodes):
        self.nodes = nodes  # 3 nodes across 3 AZs
        self.witness = witness_nodes  # 2 witness nodes in separate AZs
    
    def should_failover(self, failed_node):
        # Check: did we lose quorum?
        healthy_nodes = [n for n in self.nodes if n.is_healthy()]
        healthy_witnesses = [w for w in self.witness if w.is_healthy()]
        
        total_healthy = len(healthy_nodes) + len(healthy_witnesses) * 0.5
        total = len(self.nodes) + len(self.witness) * 0.5
        
        # Need > 50% of total voting power
        if total_healthy > total / 2:
            # We have quorum - safe to elect new leader
            new_leader = elect_leader(healthy_nodes)
            
            # FENCE the old primary (shoot the other node in the head)
            fence_node(failed_node)
            
            return new_leader
        else:
            # No quorum - must wait (CAP: choosing consistency over availability)
            raise NoQuorumError("Cannot safely failover without quorum")
    
    def fence_node(self, node):
        """Ensure old primary cannot accept writes (prevent split-brain)"""
        # 1. Revoke IAM permissions
        # 2. Modify security group (remove all inbound)
        # 3. STONITH: force-stop the instance
        # 4. Release Elastic IP / network interface
```

### Graceful Degradation

```python
class GracefulDegradation:
    """Not all features need to survive a failover"""
    
    TIER_1_SERVICES = ['auth', 'orders', 'payment']  # Must survive
    TIER_2_SERVICES = ['recommendations', 'analytics']  # Nice to have
    TIER_3_SERVICES = ['reports', 'admin-dashboard']  # Can be down
    
    def handle_failover(self, available_resources):
        # During DR, we may have limited capacity
        # Run only tier 1 services at full capacity
        # Tier 2 at reduced capacity
        # Tier 3 disabled
        
        for service in self.TIER_1_SERVICES:
            self._scale(service, self.TIER_1_SERVICES, capacity='full')
        
        for service in self.TIER_2_SERVICES:
            if available_resources > 0.5:
                self._scale(service, capacity='reduced')
        
        # Provide degraded user experience
        self._enable_fallback_ui()  # "Recommendations unavailable"
```

### Automated Failback

```python
def orchestrate_failback():
    """Return to primary after it's recovered"""
    # 1. Ensure primary is fully healthy and caught up
    primary_status = check_full_health('primary')
    if not primary_status.fully_healthy:
        raise Exception("Primary not ready for failback")
    
    # 2. Re-establish replication (DR → Primary reverse sync)
    sync_data_from_dr_to_primary()
    
    # 3. Validate data consistency
    if not verify_data_consistency('primary', 'dr'):
        raise Exception("Data mismatch detected")
    
    # 4. Switch traffic gradually (canary)
    #    Route 5% → validate → 25% → validate → 50% → 100%
    for percentage in [5, 25, 50, 100]:
        route_traffic_to_primary(percentage)
        time.sleep(120)  # Wait for metrics
        if error_rate_increased():
            rollback_to_dr()
            raise Exception("Failback aborted due to errors")
    
    # 5. Resume normal operations
    #    Primary is now primary, DR returns to standby
```

## Layman's Explanation

### The Power Grid Failover
Your home (application) is connected to the main power grid (primary region). The utility company has:

- **Automatic Transfer Switch (Load Balancer)**: Your home's backup generator detects the grid outage within 10 seconds and starts automatically. Almost no interruption.

- **Multiple Power Plants (Active/Active)**: Your city is powered by two power plants simultaneously. If one plant trips, the other carries the full load. You never notice.

- **Circuit Breakers (Circuit Breaker Pattern)**: When an appliance shorts out, only that circuit trips. The rest of the house stays powered. No cascading failure.

- **Fuse Box vs Main Breaker (Graceful Degradation)**: The main breaker protects the whole house. But ideally, individual fuses (per-service failover) prevent a toaster problem from taking down the refrigerator.

- **Manual Transfer (Manual Failover)**: For critical equipment that can't auto-switch, an operator sees the alert and throws a physical switch within 2 minutes. Slower but safer.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Automated vs Manual Failover**: Automated failover reduces RTO but requires sophisticated health checking to prevent false positives. A brief network blip shouldn't trigger a full cross-region failover. Manual failover is safer but slower. Hybrid: automated for single-AZ failures, manual approval for cross-region.
- **Health Check Design**: A simple TCP check on port 443 confirms the load balancer is running but says nothing about the application. An HTTP GET /health confirms the web server but not the database connection. An end-to-end transaction check (place test order, verify in DB, clean up) is the most reliable but most expensive. Layers of checks inform different decisions.
- **Fencing Mechanisms**: The most dangerous scenario in failover is split-brain—both old and new primary accepting writes. Strong fencing (STONITH, security group modification, IAM revocation) must be designed and tested.
- **DNS TTL Strategy**: For DNS-based failover, the TTL (Time To Live) determines the maximum staleness. Low TTL (60s) = faster failover but more DNS queries and cost. High TTL (300s) = cheaper but some users wait up to 5 minutes to see the failover. Some clients ignore TTL entirely.

### Business Impact
- **RTO Achievement**: The failover mechanism directly determines the achievable RTO. Automated 60-second failover vs manual 2-hour failover are fundamentally different business capabilities with different customer SLAs.
- **False Positive Cost**: An unnecessary failover can be as damaging as a real outage—data inconsistencies, broken user sessions, and operational confusion. The 2021 AWS us-east-1 incident included a multi-AZ failover that took longer than expected. Testing failover regularly reduces false positives.
- **Failback Complexity**: Returning from DR to primary is often more complex than the original failover—data must be reverse-synchronized, users redirected, and caches rebuilt. Failback planning is often neglected but critical for full recovery.

## On-Premises Examples

### Keepalived (Virtual IP Failover)
```bash
# keepalived.conf - Primary
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    
    virtual_ipaddress {
        10.0.1.100/24  # Virtual IP floats between nodes
    }
    
    track_script {
        chk_nginx
    }
}

vrrp_script chk_nginx {
    script "pidof nginx"
    interval 2
    weight -20  # Reduce priority if nginx dies → failover
}
```

### Patroni (PostgreSQL High Availability)
```yaml
# patroni.yml
scope: orders-db
namespace: /db/

restapi:
  listen: 0.0.0.0:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
```

## AWS Examples

### RDS Multi-AZ Automatic Failover
```hcl
resource "aws_db_instance" "main" {
  identifier = "orders-db"
  multi_az   = true  # Synchronous standby in different AZ
  
  # Automatic failover (60-120 seconds) when:
  # - Primary AZ outage
  # - Primary instance failure
  # - Storage failure on primary
  # - Manual failover initiated (reboot with failover)
}
```

```bash
# Manual failover for testing
aws rds reboot-db-instance \
  --db-instance-identifier orders-db \
  --force-failover
```

### Aurora Multi-Master (Continuous Availability)
```hcl
resource "aws_rds_cluster" "aurora_multi" {
  cluster_identifier = "orders-cluster"
  engine            = "aurora-mysql"
  engine_mode       = "multimaster"  # All instances can accept writes
  master_username   = "admin"
  master_password   = var.password
}

# If any writer fails, other writers continue accepting writes
# Zero failover time for writer failures
# Application must handle write conflicts (optimistic locking)
```

### Elastic Load Balancing Health Checks
```hcl
resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2    # 2 consecutive successes → healthy
    unhealthy_threshold = 3    # 3 consecutive failures → unhealthy
    matcher             = "200-299"
  }
  
  # Deregistration delay: time to drain connections before failover
  deregistration_delay = 30

  # Stickiness: route user back to same target (during normal ops)
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
  }
}
```

## GCP Examples

### Cloud SQL High Availability
```bash
gcloud sql instances create orders-db \
  --availability-type=REGIONAL \
  --enable-bin-log

# Automatic failover (< 60 seconds):
# - Zone failure detected
# - DNS record updated to standby's IP
# - Existing connections broken (must reconnect)
```

### Regional Managed Instance Groups (Auto-Healing)
```bash
gcloud compute instance-groups managed create app-group \
  --template=app-template \
  --size=3 \
  --region=us-central1

gcloud compute instance-groups managed set-autohealing app-group \
  --http-health-check=app-health-check \
  --initial-delay=300
```

## Azure Examples

### Azure SQL Auto-Failover Groups
```bash
az sql failover-group create \
  --name orders-failover-group \
  --server primary-server \
  --resource-group myResourceGroup \
  --partner-server dr-server \
  --partner-resource-group myResourceGroupDR \
  --failover-policy Automatic \
  --grace-period 1

# Grace period: minimum time before automatic failover (1-24 hours)
# Prevents false positives from brief outages
```

```bash
# Planned failover (zero data loss)
az sql failover-group set-primary \
  --name orders-failover-group \
  --resource-group myResourceGroupDR \
  --server dr-server

# Forced failover (potential data loss if primary unreachable)
az sql failover-group set-primary \
  --name orders-failover-group \
  --resource-group myResourceGroupDR \
  --server dr-server \
  --allow-data-loss
```

## Summary Decision Matrix

| Failover Mechanism | RTO | Complexity | Data Risk | Best For |
|-------------------|-----|-----------|-----------|----------|
| DNS Failover | 60-300s | Low | Low (async) | Regional outages, non-critical |
| LB Health Checks | 10-60s | Low | Low | Instance failures, auto-scaling |
| Multi-AZ Database | 60-120s | None (managed) | Zero | Single-AZ failure, managed DB |
| Cross-Region Read Replica | Minutes | Medium | Low (async) | Regional DR, Pilot Light |
| Multi-Master / Active-Active | Sub-second | High | Medium (conflicts) | Zero-downtime requirement |
| Application Circuit Breaker | Milliseconds | Medium | None (fail-open) | Service dependency failure |

Failover architecture is the engine that powers DR plans. The best DR strategy is worthless if the failover mechanism doesn't work when needed. Design health checks that are reliable enough to avoid false positives but sensitive enough to detect real failures. Implement strong fencing to prevent split-brain. Test failover regularly—quarterly at minimum, monthly for critical systems. A failover that has never been tested is not a failover plan; it's a theory.
