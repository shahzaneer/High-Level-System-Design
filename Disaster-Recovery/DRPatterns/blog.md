# Disaster Recovery Patterns

## Introduction
Disaster Recovery (DR) patterns define how systems recover from catastrophic failures—natural disasters, data center outages, cyber attacks, or human errors that take down entire environments. Unlike high availability (which handles component-level failures transparently), DR addresses the scenario where an entire primary site is unavailable, and operations must resume at a secondary site—potentially in a different geographic region.

The fundamental trade-off in DR architecture is cost vs. recovery speed. The more quickly you need to recover, the more you must invest in standby infrastructure. AWS defines four canonical DR patterns that form the industry standard: Backup & Restore, Pilot Light, Warm Standby, and Multi-Site Active/Active. Understanding these patterns and when to apply them is essential for any architect designing resilient systems.

## Definition

**Disaster Recovery** is the set of policies, tools, and procedures that enable the recovery or continuation of vital technology infrastructure and systems following a natural or human-induced disaster.

**DR Pattern** is a standardized architectural approach to implementing disaster recovery, defined by the RPO (Recovery Point Objective) and RTO (Recovery Time Objective) it achieves and the cost of maintaining standby infrastructure.

## Concept Explanation

### The Four Canonical DR Patterns

```
Recovery Time ──────────────────────────────────────────────→
Fast                                                      Slow
│                                                           │
├── Multi-Site Active/Active (RTO: near-zero, RPO: near-zero)
├── Warm Standby (RTO: minutes, RPO: seconds-minutes)
├── Pilot Light (RTO: tens of minutes, RPO: minutes-hours)
└── Backup & Restore (RTO: hours, RPO: hours-days)
                                                           │
Cost ──────────────────────────────────────────────────────→
Low                                                      High
```

### 1. Backup & Restore (Lowest Cost, Slowest Recovery)

```
Primary Region                    DR Region
┌──────────────┐                  ┌──────────────┐
│ App Servers  │                  │   NOTHING    │
│ Database     │                  │   RUNNING    │
│ Load Balancer│                  │              │
└──────┬───────┘                  └──────▲───────┘
       │                                │
       └──── Backups (S3/Blob) ─────────┘
            Cross-region copy
```

**How it works**:
- Backups (database snapshots, AMIs, configuration) are copied to the DR region
- No infrastructure runs in DR region (only backup storage costs)
- During DR: deploy everything from backups (infrastructure + data)

**RTO**: Hours to days (deploy infrastructure, restore databases, validate)  
**RPO**: Hours to days (since last backup)

```python
# DR runbook for Backup & Restore
def dr_backup_restore():
    # 1. Deploy infrastructure from IaC
    run_terraform('dr-region', 'terraform/dr/')
    
    # 2. Restore database from latest snapshot
    snapshot = get_latest_snapshot('orders-db', region='dr')
    db_id = restore_rds_from_snapshot(snapshot.snapshot_id)
    
    # 3. Restore application from AMI/deployment
    launch_ecs_service('order-service', region='dr')
    
    # 4. Update DNS to point to DR region
    update_route53('api.example.com', dr_load_balancer.dns_name)
    
    # 5. Validate
    run_smoke_tests(dr_load_balancer.dns_name)
```

### 2. Pilot Light (Moderate Cost, Moderate Recovery)

```
Primary Region                    DR Region
┌──────────────┐                  ┌──────────────┐
│ App Servers  │                  │ DB Replica   │ (syncing)
│ (running)    │                  │ Core Services│ (stopped/minimal)
├──────────────┤                  ├──────────────┤
│ Database     │──replication─────→│ Database     │ (running)
│ (active)     │                  │ (standby)    │
└──────────────┘                  └──────────────┘
```

**How it works**:
- Database replicates to DR region (the "pilot light")
- Core infrastructure components exist but are scaled to minimum (or stopped)
- During DR: scale up DR infrastructure, point application to DR database, update DNS

**RTO**: Tens of minutes (scale existing infrastructure)  
**RPO**: Seconds to minutes (database is continuously replicating)

```hcl
# DR region: Pilot Light - database running, apps scaled to zero
resource "aws_db_instance" "dr" {
  identifier = "orders-db-dr"
  instance_class = "db.t4g.small"  # Small instance, just replication
  replicate_source_db = aws_db_instance.primary.identifier
  
  skip_final_snapshot = true
}

resource "aws_ecs_service" "dr" {
  name = "order-service-dr"
  desired_count = 0  # Scaled to zero until DR event
  
  # Service definition exists, ready to scale up
}
```

### 3. Warm Standby (Higher Cost, Faster Recovery)

```
Primary Region                    DR Region
┌──────────────┐                  ┌──────────────┐
│ App Servers  │                  │ App Servers  │ (minimal, running)
│ (full scale) │                  │ Load Balancer│ (ready)
├──────────────┤                  ├──────────────┤
│ Database     │──replication─────→│ Database     │ (running, syncing)
│ (active)     │                  │ (warm)       │
└──────────────┘                  └──────────────┘
```

**How it works**:
- Scaled-down but fully functional environment runs in DR region
- All services running, just at minimum capacity
- During DR: auto-scale to full capacity, update DNS

**RTO**: Minutes (scale existing services, DNS propagation)  
**RPO**: Seconds (continuous replication)

### 4. Multi-Site Active/Active (Highest Cost, Near-Zero Recovery)

```
Region A                           Region B
┌──────────────┐                  ┌──────────────┐
│ App Servers  │                  │ App Servers  │
│ (full scale) │◄─────DNS─────────►│ (full scale) │
├──────────────┤                  ├──────────────┤
│ Database     │◄────synchronous──►│ Database     │
│ (active)     │   replication    │ (active)     │
└──────────────┘                  └──────────────┘
```

**How it works**:
- Both regions serve production traffic simultaneously
- DNS routes users to the nearest healthy region
- Databases synchronously replicate (or use multi-master with conflict resolution)

**RTO**: Near-zero (other region already serving traffic)  
**RPO**: Near-zero (synchronous replication or multi-master)

```hcl
# Active/Active: Route 53 latency-based routing
resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  latency_routing_policy {
    region = "us-east-1"
  }
  set_identifier = "primary"
  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_dr" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  latency_routing_policy {
    region = "eu-west-1"
  }
  set_identifier = "secondary"
  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

### DR Health Monitoring and Automated Failover

```python
class DRHealthMonitor:
    def __init__(self):
        self.primary_region = 'us-east-1'
        self.dr_region = 'us-west-2'
        self.failure_threshold = 3  # consecutive health check failures
    
    def check_health(self):
        primary_healthy = self._health_check(self.primary_region)
        
        if not primary_healthy:
            self.failure_count += 1
            if self.failure_count >= self.failure_threshold:
                self._initiate_failover()
        else:
            self.failure_count = 0
    
    def _initiate_failover(self):
        # 1. Promote DR database to primary
        dr_db = self._get_db(self.dr_region)
        if dr_db.is_read_replica:
            dr_db.promote_to_primary()
        
        # 2. Scale up DR services
        self._scale_ecs_service('order-service', self.dr_region, desired_count=6)
        
        # 3. Update DNS (Route 53 health checks handle this automatically
        #    if configured with evaluate_target_health = true)
        
        # 4. Notify operations team
        self._send_alert(
            severity='CRITICAL',
            message=f'DR Failover initiated at {datetime.utcnow()}'
        )
```

### DR Testing Patterns

```
Test Types:
  1. Tabletop Exercise: Walk through the runbook with the team
     Frequency: Quarterly
     Impact: None (discussion only)
  
  2. Simulated Failover: Isolated environment test
     Frequency: Semi-annually
     Impact: None (uses test environment)
  
  3. Live Failover: Switch production to DR, switch back
     Frequency: Annually
     Impact: Brief service disruption during DNS propagation
```

## Layman's Explanation

### The Spare Tire Analogy
Your car (production system) needs to survive a tire blowout (disaster):

- **Backup & Restore**: Your car has no spare tire. If you get a flat, you call a tow truck, they take your car to a shop, and the shop orders a new tire and installs it. You're stranded for hours (RTO = 4 hours). You might lose your grocery run (RPO = lost data for the trip).

- **Pilot Light**: You have a spare tire in your trunk AND the jack and wrench are there—but the tire isn't inflated. When you get a flat, you need to inflate the spare before mounting it. 20 minutes delay.

- **Warm Standby**: Your spare tire is inflated, the jack is in the right position, the lug wrench is in hand. You get a flat—15 minutes and you're back on the road.

- **Active/Active**: You have run-flat tires. If one goes flat, the other tires carry the load and you keep driving at reduced speed to the nearest shop. Zero downtime.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **DR Region Selection**: Must be far enough from primary to survive regional disasters (different seismic zones, flood plains, power grids) but close enough to meet latency requirements for synchronous replication. Typically 300+ miles separation.
- **Data Replication vs Application Replication**: Database replication is the hardest part of DR. Multi-master is ideal but complex (conflict resolution). Read replicas are simpler but require promotion. The database architecture choice constrains the DR pattern.
- **Automated vs Manual Failover**: Automated failover reduces RTO but risks false positives (a brief network blip triggers unnecessary failover). Manual with automated health monitoring balances speed with human judgment.
- **DR Cost Optimization**: Reserve capacity in DR region (cheaper), use Spot instances for non-critical DR workloads, or use on-demand for DR testing only. The cost of DR should be proportional to the business impact of downtime.

### Business Impact
- **Business Continuity**: Direct correlation between RTO and revenue loss. An e-commerce platform losing $100K/hour during peak justifies Active/Active ($500K+/year). A blog with $0/hour justifies Backup & Restore ($100/month).
- **Regulatory Requirements**: Financial services (SEC Rule 613), healthcare (HIPAA contingency plan), and critical infrastructure require documented and tested DR plans.
- **Insurance Premiums**: Cyber insurance and business interruption insurance require DR plans. Premiums correlate with DR maturity. Better DR = lower premiums.

## On-Premises Examples

### PostgreSQL Cross-Region Replication
```bash
# Primary (us-east)
# postgresql.conf
wal_level = logical
max_wal_senders = 5
wal_keep_size = 1024

# DR (us-west) - streaming replica
pg_basebackup -h primary.internal -D /var/lib/postgresql/data -R -P

# Failover: promote secondary
pg_ctlcluster 14 main promote
```

### MySQL Multi-Source Replication
```sql
-- DR site: replicate from multiple primaries (cascading)
CHANGE MASTER TO
  MASTER_HOST='primary-ny.internal',
  MASTER_USER='repl',
  MASTER_PASSWORD='password'
  FOR CHANNEL 'channel_ny';

CHANGE MASTER TO
  MASTER_HOST='primary-lon.internal',
  MASTER_USER='repl',
  MASTER_PASSWORD='password'
  FOR CHANNEL 'channel_lon';

START SLAVE FOR CHANNEL 'channel_ny';
START SLAVE FOR CHANNEL 'channel_lon';
```

## AWS Examples

### RDS Cross-Region Read Replica (Pilot Light / Warm Standby)
```hcl
resource "aws_db_instance" "dr" {
  provider          = aws.dr_region
  identifier        = "orders-db-dr"
  replicate_source_db = aws_db_instance.primary.arn
  instance_class    = "db.t4g.small"  # Minimal during normal ops

  skip_final_snapshot = true
}

# During DR: promote to standalone
# aws rds promote-read-replica --db-instance-identifier orders-db-dr
```

### Aurora Global Database (Warm Standby / Active-Active reads)
```hcl
resource "aws_rds_global_cluster" "global" {
  global_cluster_identifier = "orders-global"
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  
  # Storage replicated across regions
  # Typical cross-region replication lag: < 1 second
}

resource "aws_rds_cluster" "primary" {
  global_cluster_identifier = aws_rds_global_cluster.global.id
  engine                    = "aurora-postgresql"
  # ...
}

resource "aws_rds_cluster" "secondary" {
  provider                  = aws.eu_west_1
  global_cluster_identifier = aws_rds_global_cluster.global.id
  engine                    = "aurora-postgresql"
  # Up to 15 read replicas per region
}
```

### DynamoDB Global Tables (Active/Active, Multi-Region)
```hcl
resource "aws_dynamodb_global_table" "orders" {
  name = "Orders"

  replica {
    region_name = "us-east-1"
  }

  replica {
    region_name = "eu-west-1"
  }

  replica {
    region_name = "ap-southeast-1"
  }
}
# Last-write-wins conflict resolution
# < 1 second cross-region replication
# Application writes to nearest region
```

### Route 53 DNS Failover
```hcl
resource "aws_route53_health_check" "primary" {
  fqdn              = "app.primary.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
  regions           = ["us-east-1", "us-west-1", "eu-west-1"]  # multi-region checkers
}

resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_failover" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "secondary"

  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }
}
```

## GCP Examples

### Cloud Spanner (Active/Active, Multi-Region)
```sql
-- Create multi-region Spanner instance
CREATE INSTANCE orders-global
  CONFIGURATION = 'nam-eur-asia3'  -- US + Europe + Asia
  NODE_COUNT = 3;

-- Automatically:
-- - Synchronous replication across all regions
-- - External consistency (TrueTime)
-- - Zero RPO, seconds RTO
-- - Application writes to any region
```

### Cloud SQL Cross-Region Read Replica
```bash
gcloud sql instances create orders-db-dr \
  --master-instance-name=orders-db-primary \
  --region=us-west1

# Failover: promote replica
gcloud sql instances promote-replica orders-db-dr
```

## Azure Examples

### Azure Site Recovery (ASR)
```bash
# Enable replication for VMs to DR region
az site-recovery protection-container mapping create \
  --name "primary-to-dr-mapping" \
  --resource-group myResourceGroup \
  --vault-name myRecoveryVault \
  --primary-protection-container "primary-container" \
  --recovery-protection-container "dr-container"

# Create recovery plan (orchestrated failover)
az site-recovery recovery-plan create \
  --name "app-failover-plan" \
  --resource-group myResourceGroup \
  --vault-name myRecoveryVault \
  --source-vm "app-server-1" "app-server-2" "db-server"
```

### Azure Front Door (Active/Active, Global)
```json
{
  "backendPools": [
    {
      "name": "primary-pool",
      "backends": [
        {"address": "app-useast.azurewebsites.net"}
      ],
      "healthProbeSettings": {
        "path": "/health",
        "intervalInSeconds": 30
      },
      "loadBalancingSettings": {
        "sampleSize": 4,
        "successfulSamplesRequired": 2
      }
    },
    {
      "name": "dr-pool",
      "backends": [
        {"address": "app-ukwest.azurewebsites.net"}
      ]
    }
  ]
}
```

## Summary

| DR Pattern | RPO | RTO | Relative Cost | When to Use |
|-----------|-----|-----|--------------|-------------|
| Backup & Restore | Hours-Days | Hours-Days | 1x | Non-critical, dev/test |
| Pilot Light | Minutes | Tens of minutes | 1.5-2x | Business apps, moderate criticality |
| Warm Standby | Seconds | Minutes | 2-3x | Revenue-critical, regulated |
| Active/Active | Near-zero | Near-zero | 4-5x+ | Life-critical, zero-downtime SLA |

The DR pattern for each system should be driven by the business's RPO and RTO requirements, not by what's technically impressive. A blog running Backup & Restore with 24-hour RPO is perfectly appropriate. A trading platform with Active/Active and sub-second RPO is necessary. The architect's role is to align DR investment with business risk—over-engineering DR wastes money; under-engineering DR risks the business.
