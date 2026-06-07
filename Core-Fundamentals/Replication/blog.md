# Database Replication

## Introduction
Database Replication is the process of copying and maintaining database objects across multiple servers to improve availability, fault tolerance, and read performance. The concept emerged from the need to protect data against hardware failures and to scale read-heavy workloads. While early replication solutions were simple master-slave setups, modern architectures support complex multi-master, multi-region topologies that enable global applications to serve users from the nearest data center.

Replication is not just a backup strategy—it is a fundamental building block for high availability, disaster recovery, read scalability, and geographic distribution. Every production database in a serious system uses some form of replication.

## Definition
**Database Replication** is the process of storing copies of data on multiple nodes (servers, data centers, or regions) to ensure redundancy and reliability. Changes made on one node (the leader/primary/master) are propagated to other nodes (followers/replicas/secondaries) through a replication log. The system can then serve reads from any replica and fail over to a replica if the primary fails.

## Concept Explanation

### Replication Topologies

#### Leader-Follower (Master-Slave)
The most common topology. One leader handles all writes. Followers replicate from the leader and serve reads.

```
[Client Writes] ──→ [Leader] ──→ [Follower 1]
                         ├──────→ [Follower 2]
                         └──────→ [Follower 3]
```

- **Writes**: Leader only (ensures consistency)
- **Reads**: Any node (potentially stale on followers)
- **Failover**: Manual or automated promotion of a follower to leader

#### Leader-Leader (Master-Master / Multi-Master)
Multiple nodes accept writes simultaneously and replicate changes to each other.

```
[Client] ──→ [Leader A] ←──replication──→ [Leader B] ←── [Client]
```

- **Writes**: Any leader
- **Reads**: Any leader
- **Challenge**: Conflict resolution (which write wins when both leaders update the same row?)
- **Use**: Multi-region active-active deployments, collaborative editing

#### Cascading Replication
Followers replicate from other followers, creating a tree topology that reduces load on the leader.

```
[Leader] ──→ [Follower 1] ──→ [Follower 1a]
         └──→ [Follower 2] ──→ [Follower 2a]
                             └──→ [Follower 2b]
```

Useful for geographically distributed replica chains.

### Replication Methods

#### Synchronous Replication
The leader waits for confirmation from follower(s) before acknowledging the write to the client. Guarantees the follower has the data but increases write latency.

```sql
-- PostgreSQL synchronous replication
ALTER SYSTEM SET synchronous_commit = 'remote_apply';
-- or set per transaction:
SET LOCAL synchronous_commit = 'remote_apply';
BEGIN;
INSERT INTO orders (...) VALUES (...);
COMMIT; -- blocks until at least one synchronous standby confirms
```

#### Asynchronous Replication
The leader acknowledges the write immediately and propagates changes to followers asynchronously. Low write latency but risk of data loss if the leader crashes before replication completes.

```sql
-- PostgreSQL default: asynchronous
ALTER SYSTEM SET synchronous_commit = 'off';
```

#### Semi-Synchronous Replication
The leader waits for at least one follower to receive the change (in its log) but not necessarily apply it. A middle ground between synchronous and asynchronous.

### Replication Lag
The delay between a write on the leader and its availability on a follower. Lag causes:

- **Stale reads**: User writes data, refresh shows old data (because the read went to a replica behind on replication)
- **Read-your-writes inconsistency**: User writes on leader, app reads from a follower, sees old data

Mitigation strategies:
- **Read-after-write consistency**: Route reads for recently modified data to the leader
- **Monotonic reads**: Ensure a user always reads from the same replica
- **Causal consistency**: Track causality between operations to ensure related writes are read in order

```python
# Read-your-writes: track last write time, read from leader if recent
def get_user(user_id, user_last_write_time):
    if time.time() - user_last_write_time < 5:  # within 5 seconds of write
        return leader_db.query(user_id)       # read from leader
    return replica_db.query(user_id)            # read from replica
```

### Failover
The process of promoting a follower to leader when the leader fails:

- **Manual**: DBA runs promotion commands. Slow but controlled.
- **Automated**: Orchestrators (Patroni, Orchestrator, RDS Multi-AZ) detect failure and promote. Fast but risk of split-brain.

**Split-brain**: Both old and new leaders accept writes, causing data divergence. Mitigated by fencing tokens, quorum-based leader election, or STONITH (Shoot The Other Node In The Head).

## Layman's Explanation

Imagine you're a popular chef (the leader database) with your original recipe book. Every time you create a new recipe (write), your assistant chefs (followers) copy it into their own notebooks. When customers ask for recipes (read queries), any assistant can answer.

**Synchronous replication**: You don't tell the customer "recipe added!" until your assistant has fully copied it. Slow but safe—if you drop dead, the assistant has it.

**Asynchronous replication**: You announce "done!" immediately and trust the assistant to copy later. Fast, but if you die before they finish copying, that recipe is lost forever.

**Replication lag**: Your assistant is still copying yesterday's recipes. A customer asks them for today's special—they don't have it yet. This is like a user refreshing a page and seeing old data because their read hit a lagging replica.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Availability Requirements**: How much downtime is acceptable? Synchronous replication with automated failover for 99.99% uptime vs asynchronous with manual failover for 99.9%.
- **Data Loss Tolerance (RPO)**: Recovery Point Objective—how much data can you lose? Zero data loss requires synchronous replication. Five minutes of data loss allows asynchronous with frequent replication.
- **Recovery Time Objective (RTO)**: How fast must you recover? Automated failover takes 30-120 seconds. Manual failover takes minutes to hours.
- **Read Scaling Strategy**: Read replicas scale reads linearly. But replication lag means they're eventually consistent. Does your application tolerate stale reads?
- **Geographic Distribution**: Multi-region replication for latency reduction (serve users from nearest replica) and disaster recovery (survive entire region failure).

### Business Impact
- **Revenue Protection**: Every minute of downtime during peak hours can cost millions in e-commerce or trading systems. Replication with automated failover mitigates this.
- **User Experience**: Read replicas in multiple regions reduce latency from 200ms (cross-ocean) to 10ms (local). Studies show 100ms latency increase reduces Amazon's revenue by 1%.
- **Compliance**: Data sovereignty laws (GDPR) require data to stay within specific regions. Replication must be configured to respect these boundaries.
- **Disaster Recovery**: Survive data center fires, earthquakes, or cloud region outages. Replication across 3+ AZs or regions is mandatory for business continuity.

## On-Premises Examples

### PostgreSQL Streaming Replication
```bash
# Primary server (postgresql.conf)
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1024    # MB of WAL to retain

# Standby server
pg_basebackup -h primary_host -D /var/lib/postgresql/data -P -R

# Create standby.signal file for read-only replica
touch /var/lib/postgresql/data/standby.signal
```

```sql
-- Configure synchronous standby on primary
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 2 (replica1, replica2, replica3)';
-- This means: wait for at least 2 of the 3 listed replicas
```

### MySQL Group Replication (Multi-Master)
```sql
-- On each node
SET GLOBAL group_replication_bootstrap_group = ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group = OFF;

-- Conflict detection on primary keys
-- If two nodes insert the same PK, the second commit is rolled back
```

### Orchestrator (Automated Failover)
```bash
# Orchestrator manages MySQL replication topology
orchestrator -config orchestrator.conf.json http

# API to perform graceful failover
curl -X POST http://orchestrator:3000/api/graceful-master-takeover/cluster1
```

## AWS Examples

### Amazon RDS Multi-AZ (Synchronous)
Automatic synchronous replication to a standby in another AZ:

```hcl
resource "aws_db_instance" "orders" {
  identifier     = "orders-db"
  engine         = "postgres"
  instance_class = "db.r6g.xlarge"
  
  multi_az               = true   # synchronous standby in another AZ
  backup_retention_period  = 30
  backup_window            = "03:00-04:00"
  
  # Automated failover: ~60-120 seconds during AZ outage
  # RDS converts CNAME to point to the promoted standby
}
```

### Amazon RDS Read Replicas (Asynchronous)
Horizontal read scaling within or across regions:

```hcl
resource "aws_db_instance" "orders_replica" {
  identifier          = "orders-read-replica"
  replicate_source_db = aws_db_instance.orders.identifier
  instance_class      = "db.r6g.large"
  
  # Up to 15 read replicas per source
  # Can be in different AZ or different region
  # Cross-region: ~5-10 minutes typically 1+ hour for large DBs
}
```

### Amazon Aurora Cluster
Proprietary distributed storage with up to 15 read replicas sharing the same storage volume (no replication lag for storage layer):

```hcl
resource "aws_rds_cluster" "aurora" {
  cluster_identifier = "aurora-cluster"
  engine            = "aurora-postgresql"
  master_username   = "admin"
  master_password   = var.db_password
  
  # Aurora replicates data 6 ways across 3 AZs at storage layer
  # Reader endpoints automatically distribute reads across replicas
}

resource "aws_rds_cluster_instance" "reader" {
  count              = 3
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r6g.large"
  engine            = "aurora-postgresql"
}
```

### Amazon DynamoDB Global Tables
Multi-region, multi-master replication with last-write-wins conflict resolution:

```python
import boto3

dynamodb = boto3.client('dynamodb')

dynamodb.create_global_table(
    GlobalTableName='Orders',
    ReplicationGroup=[
        {'RegionName': 'us-east-1'},
        {'RegionName': 'eu-west-1'},
        {'RegionName': 'ap-southeast-1'}
    ]
)

# Writes to any region replicate to all others within <1 second
table = boto3.resource('dynamodb', region_name='eu-west-1').Table('Orders')
table.put_item(Item={'OrderID': 'ORD-001', 'Status': 'shipped'})
```

## GCP Examples

### Cloud SQL High Availability
Regional synchronous replication with automatic failover:

```bash
gcloud sql instances create orders-db \
  --availability-type=REGIONAL \
  --enable-bin-log \
  --retained-bin-log-count=7
  
# Read replicas for horizontal read scaling
gcloud sql instances create orders-read-1 \
  --master-instance-name=orders-db \
  --region=us-central1

# Cross-region read replica for disaster recovery
gcloud sql instances create orders-dr \
  --master-instance-name=orders-db \
  --region=europe-west1
```

### Cloud Spanner
Automatically replicates data across zones and regions with strong consistency:

```sql
CREATE DATABASE Orders;
-- Spanner automatically:
-- 1. Replicates data across 3 zones in a region (synchronous)
-- 2. Supports multi-region with synchronous replication between continents
-- 3. Uses TrueTime for external consistency even across regions

-- Configuring multi-region
ALTER DATABASE Orders SET
  OPTIONS (version_retention_period = '7d');
-- Data automatically replicated across all regions in the instance configuration
```

### Memorystore (Redis) Replication
```bash
gcloud redis instances create orders-cache \
  --redis-version=redis_7_0 \
  --tier=STANDARD_HA \       # enables replication with failover
  --replica-count=1
```

## Azure Examples

### Azure SQL Database Active Geo-Replication
Readable secondaries in same or different regions:

```bash
# Create geo-replica
az sql db replica create \
  --name orders-db \
  --resource-group myResourceGroup \
  --server primary-server \
  --partner-server secondary-server \
  --partner-resource-group myResourceGroupDR \
  --secondary-type Geo

# Force failover (planned)
az sql db replica set-primary \
  --name orders-db \
  --resource-group myResourceGroupDR \
  --server secondary-server
```

### Azure Database for PostgreSQL (Flexible)
```bash
az postgres flexible-server create \
  --name orders-db \
  --resource-group myResourceGroup \
  --high-availability ZoneRedundant \
  --standby-zone 2

# Read replicas for read scaling
az postgres flexible-server replica create \
  --replica-name orders-read-1 \
  --source-server orders-db
```

### Azure Cosmos DB Multi-Region
Turnkey global distribution with tunable consistency:

```python
from azure.cosmos import CosmosClient

client = CosmosClient(ENDPOINT, credential=KEY)

database = client.create_database_if_not_exists(
    'ecommerce'
)

# Multi-region writes enabled
container = database.create_container_if_not_exists(
    id='orders',
    partition_key=PartitionKey(path='/customerId'),
    default_ttl=86400
)

# Write to nearest region (multi-master)
client_asia = CosmosClient(ENDPOINT, credential=KEY, preferred_locations=['Southeast Asia'])
# Cosmos DB handles conflict resolution with CRDT or custom merge procedure
```

## Summary Decision Matrix

| Replication Type | RPO (Data Loss) | RTO (Recovery Time) | Write Latency | Use Case |
|-----------------|-----------------|---------------------|---------------|----------|
| Synchronous | Zero | 30-120 sec | Higher | Financial transactions, critical data |
| Asynchronous | Seconds to minutes | Minutes | Low | Analytics, reporting, eventual consistency OK |
| Multi-Master | Depends on conflict resolution | Sub-second | Low | Global apps, collaborative editing |
| Cascading | Depends on chain depth | Variable | Varies | Geographic distribution, CDN-like DB |

Replication is one of the most consequential decisions in system architecture. The choice between synchronous and asynchronous determines your system's resilience profile. The choice between leader-follower and multi-master determines your write scalability. Every solution architect must understand replication lag, failover behavior, and the trade-offs between data safety and performance.
