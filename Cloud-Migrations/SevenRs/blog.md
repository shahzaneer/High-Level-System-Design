# The 7 R's of Cloud Migration

## Introduction
The 7 R's framework is the standard decision model for cloud migration strategy, originally introduced by Gartner and extended by AWS. It provides a structured way to evaluate each application in a portfolio and determine the optimal migration approach—balancing effort, cost, risk, and benefit. Not every application belongs in the cloud, and those that do may take different paths to get there.

The framework emerged from thousands of enterprise migrations where a "one-size-fits-all" approach failed. Some applications can move with minimal changes ("lift and shift"), others need modifications to leverage cloud capabilities ("replatform"), and others should be rewritten entirely or replaced with SaaS alternatives. Understanding when to apply each strategy is the difference between a successful migration and an expensive failure.

## Definition

**The 7 R's** are migration strategies that define how an application will transition to the cloud:

1. **Retire** — Decommission applications that are no longer needed
2. **Retain** — Keep applications on-premises (regulatory, latency, or dependency reasons)
3. **Rehost** (Lift & Shift) — Move applications to the cloud without modification
4. **Relocate** — Move to the cloud without purchasing new hardware, without changing application
5. **Replatform** (Lift, Tinker & Shift) — Make minor cloud optimizations without changing core architecture
6. **Repurchase** (Replace / Drop & Shop) — Replace with a SaaS offering
7. **Refactor / Rearchitect** — Fundamentally redesign for cloud-native capabilities

## Concept Explanation

### Decision Framework

```python
def classify_application(app):
    """
    Determine which R to apply based on application characteristics
    """
    
    # R1: RETIRE - No longer needed
    if app.last_access_date < (datetime.now() - timedelta(days=365)):
        if app.data.classification != 'compliance_retention':
            return 'RETIRE'
    
    # R2: RETAIN - Must stay on-premises
    if app.requires_mainframe or app.latency_requirement_ms < 1:
        return 'RETAIN'
    if app.regulatory_constraints.get('data_sovereignty') == 'on_prem_only':
        return 'RETAIN'
    if app.depends_on_tools that can't be migrated:
        return 'RETAIN'  # Until dependencies are resolved
    
    # R7: REPURCHASE - SaaS replacement available
    if app.type == 'CRM' and SaaS_vendors_available:
        return 'REPURCHASE'  # e.g., Salesforce, Dynamics 365
    if app.type == 'Email' or app.type == 'Office_Suite':
        return 'REPURCHASE'  # e.g., Google Workspace, Microsoft 365
    
    # R6: REFACTOR - Business case for cloud-native
    if app.strategic_importance == 'HIGH' and app.agility_needed:
        if app.monolithic and app.team_size >= 3:
            return 'REFACTOR'  # Break into microservices
    
    # R5: REPLATFORM - Minor optimizations
    if app.database == 'self-managed-postgresql':
        return 'REPLATFORM'  # Move to RDS (managed database)
    if app.runs_in_vm and app.cloud_ready:
        return 'REPLATFORM'  # VM → Containers
    
    # R3/R4: REHOST / RELOCATE - Minimal changes
    return 'REHOST'  # Default starting point for most enterprise apps
```

### R1: Retire

```
Discovery typically finds 10-20% of applications are candidates for retirement:
- Applications nobody uses anymore
- Duplicate systems (company acquired two CRMs)
- Legacy systems replaced but never decommissioned
- "Zombie" servers running but serving zero traffic
```

**Process**: Verify zero usage over 90+ days → notify stakeholders → archive data if needed → shut down.

### R2: Retain

```
Legitimate reasons to retain:
- Mainframe applications (COBOL, CICS, IMS) without migration path
- Ultra-low latency (sub-millisecond) where cloud latency is physically limited
- Regulatory data sovereignty (some government classified data)
- Applications being phased out in 6-12 months
- Deep hardware dependencies (specialized industrial controllers)
```

### R3: Rehost (Lift & Shift)

```
Process:
  1. Discover: inventory server specs (CPU, RAM, disk, network)
  2. Replicate: use migration tool (MGN, ASR, Migrate for Compute Engine)
  3. Sync: continuous data replication until cutover
  4. Cutover: brief downtime for final sync + DNS update
  5. Optimize: right-size instances post-migration (often over-provisioned)

Best for:
  - Large-scale migrations (hundreds of VMs) where speed is critical
  - Applications with unknown internals (no subject matter expert)
  - Short-term migration deadlines (data center lease expiring)
```

```bash
# AWS Application Migration Service (MGN) - Rehost
aws mgn initialize-service
aws mgn create-replication-configuration-template \
  --staging-area-subnet-id subnet-xxx \
  --staging-area-tags '{"MigratedBy":"MGN"}'

# Azure Migrate - Rehost
az migrate project create --name migration-project --resource-group myRG
az migrate assessment create --project-name migration-project
```

### R4: Relocate

```
Relocate (VMware Cloud on AWS, Azure VMware Solution, GCP VMware Engine):
  - Move VMware VMs to cloud-native VMware without conversion
  - No OS/application changes
  - Preserves VMware tooling (vSphere, vSAN, NSX)
  - Good for large VMware environments needing quick exit from data centers
```

### R5: Replatform (Lift, Tinker & Shift)

```
Optimizations without architectural change:
  Database:   Self-managed Oracle on EC2 → Amazon RDS for Oracle
  Containers: VM running Docker → ECS/EKS Fargate
  Storage:    NFS file server → EFS/Azure Files
  Identity:   Local AD → AWS Managed AD + SSO
  Monitoring: Nagios → CloudWatch/Azure Monitor
  Backups:    Cron + rsync → AWS Backup
```

### R6: Repurchase (SaaS)

```
Migration through procurement:
  On-premise CRM → Salesforce
  On-premise HR → Workday
  On-premise Wiki → Confluence Cloud
  On-premise Git → GitHub Enterprise Cloud
  On-premise Monitoring → Datadog/New Relic
  
Challenges: Data migration, user training, integration with remaining apps
```

### R7: Refactor / Rearchitect

```
The most effort-intensive, highest-value strategy:

Monolith → Microservices:
  1. Identify bounded contexts (Domain-Driven Design)
  2. Extract services one by one (Strangler Fig pattern)
  3. Migrate data ownership per service
  4. Implement API gateway, service mesh
  5. Decommission monolith piece by piece

Legacy .NET Framework → .NET Core on Linux:
  1. Assess dependencies (Windows-only APIs, COM components)
  2. Port to .NET Standard / .NET Core incrementally
  3. Containerize each service
  4. Deploy to ECS/EKS/AKS

Trade-offs:
  ✓ Best cloud optimization (elasticity, cost, resilience)
  ✓ Improves development velocity long-term
  ✗ Highest migration cost and risk
  ✗ Longest timeline (months to years)
```

### Migration Wave Planning

```
Wave 1 (Quick Wins): Rehost simple apps → build cloud muscle
  - Internal tools, dev/test environments
  - No customer impact if issues arise
  - Timeline: 2-4 weeks

Wave 2 (Core Apps): Replatform moderate complexity
  - Business applications with documented architecture
  - Managed database migration (self-managed → RDS)
  - Timeline: 1-3 months

Wave 3 (Complex): Refactor critical systems
  - Customer-facing, revenue-generating applications
  - Microservices extraction
  - Timeline: 3-12 months

Wave 4 (Holdouts): Retain, Retire, or Special Cases
  - Mainframe migration projects
  - Regulatory compliance systems
  - Timeline: 6-24 months
```

## Layman's Explanation

### The House Move Analogy
You're moving from your old house (on-premises data center) to a new smart home (cloud). Each item in your house is an application:

- **Retire**: That broken exercise bike in the garage you haven't touched in 3 years. Don't move it. Throw it away.
- **Retain**: Your grand piano. It weighs 800 lbs, requires special climate control, and the new house's vibration would affect tuning. It stays with the old house owner.
- **Rehost**: Your couch. It fits fine. Pick it up, put it on the truck, put it in the new house. Same couch, new location.
- **Replatform**: Your TV. The new house has smart mounts. You add a VESA adapter (minor modification). You plug it into the smart home system. It's still your TV, but now it works with Alexa.
- **Repurchase**: Your landline phone. Instead of moving it, you cancel it and buy a smartphone with a data plan. Better functionality, zero moving effort.
- **Refactor**: Your filing cabinet. You don't move it. Instead, you scan every document, organize them in a digital system, implement search, and set up access controls. The system is completely reimagined but serves the same purpose.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Migration Strategy Per Application**: Every application gets its own R classification. The ERP might be replatformed while the employee portal is retired and replaced with SaaS. One-size-fits-all migrations waste effort on wrong-fit applications.
- **Sequencing and Dependencies**: Applications with dependencies must migrate in dependency order. If App B depends on App A, migrate App A first (or migrate both in the same wave). Dependency mapping is the most critical and often neglected migration planning activity.
- **Cutover Strategy**: One-time cutover (downtime), incremental sync (minimal downtime), or parallel run (both environments active). The choice depends on RTO tolerance and data volume.
- **Cost Estimation**: The 7 R's each have different cost profiles. Rehost is cheap to execute but may have higher ongoing cloud costs (over-provisioned instances). Refactor is expensive to execute but cheaper long-term (serverless, auto-scaling, right-sized).

### Business Impact
- **Portfolio Rationalization**: The discovery phase typically identifies 10-20% of applications for retirement. For a 1,000-server data center migration, retiring 150 servers saves $500K-$1M/year in cloud costs plus migration effort.
- **Risk Management**: Moving the most complex, fragile application first (as many organizations do, driven by importance) maximizes risk. Moving simple, low-risk apps first builds confidence and operational capability.
- **Time to Value**: Rehost delivers ROI in weeks (data center savings). Refactor delivers ROI in years (development agility, elasticity). Both are valid parts of a complete migration strategy.

## On-Premises Examples

### VMware → Cloud Rehosting
```bash
# Carbonite / Double-Take migration (VM replication)
# 1. Install agent on source VM
# 2. Continuous replication to cloud target
# 3. Cutover with minimal downtime (< 15 minutes)

# AWS MGN Agent Installation (Linux)
wget -O ./aws-replication-installer-init.py \
  https://aws-application-migration-service-us-east-1.s3.amazonaws.com/latest/linux/aws-replication-installer-init.py
sudo python3 aws-replication-installer-init.py \
  --region us-east-1 \
  --aws-access-key-id $AWS_ACCESS_KEY \
  --aws-secret-access-key $AWS_SECRET_KEY
```

### Database Replatform (Self-Managed → Managed)
```bash
# MySQL on-prem → RDS MySQL
# 1. Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier migrated-db \
  --engine mysql \
  --engine-version 8.0.35

# 2. Dump source
mysqldump --single-transaction --routines --triggers \
  --databases mydb > dump.sql

# 3. Restore to RDS
mysql -h migrated-db.xxx.rds.amazonaws.com -u admin -p < dump.sql

# 4. Switch application config
```

## AWS Examples

### AWS Migration Hub (Discovery + Planning)
```bash
# Deploy discovery agent
aws discovery start-data-collection-by-agent-ids \
  --agent-ids agent-xxx

# Portfolio assessment
aws migrationhub-strategy list-assessment-templates
aws migrationhub-strategy get-portfolio-summary
```

### AWS MGN (Rehost)
```bash
# Initial sync
aws mgn start-replication --source-server-id s-xxx

# Launch test instance
aws mgn start-test --source-server-id s-xxx

# Cutover
aws mgn start-cutover --source-server-id s-xxx
```

## GCP Examples

### Migrate for Compute Engine (Rehost)
```bash
# Deploy Migrate Connector in source environment
gcloud compute instances create migrate-connector \
  --image-family=migrate-for-compute-engine \
  --image-project=migration-solutions

# Register source VMs in Migrate for Compute Engine
# Run replication, test clone, then cutover
```

## Azure Examples

### Azure Migrate (Assessment + Migration)
```bash
# Create assessment
az migrate project create --name migration-project --resource-group myRG
az migrate assessment create --project-name migration-project

# Migrate VM
az migrate vmware create-replication --vm-id /subscriptions/.../virtualMachines/app-vm

# Test failover
az migrate test-failover --vm-id app-vm

# Migrate
az migrate migrate --vm-id app-vm
```

## Summary

| Strategy | Effort | Timeline | Cloud Benefit | Best For |
|----------|--------|----------|--------------|----------|
| Retire | None | Days | Cost elimination | Unused/duplicate apps |
| Retain | None | N/A | None (stays on-prem) | Mainframe, regulatory |
| Rehost | Low | Weeks | Quick exit from DC | Large-scale, fast timeline |
| Relocate | Low | Weeks | Quick exit + VMware | VMware environments |
| Replatform | Medium | Weeks-Months | Managed services | DBs, containers |
| Repurchase | Medium | Months | Modern features | CRM, email, HR |
| Refactor | High | Months-Years | Full cloud-native | Strategic, competitive |

The 7 R's is not a one-time exercise. As cloud capabilities evolve and business priorities shift, applications that were rehosted 3 years ago may be candidates for replatform or refactor today. The migration journey is continuous—the first move to the cloud is just the beginning of cloud optimization.
