# Migration Assessment & Portfolio Discovery

## Introduction
Migration Assessment is the critical first phase of any cloud migration program. Before moving a single workload, organizations must understand what they have, what it depends on, what it costs, and what risks it carries. Gartner reports that through 2024, 60% of cloud migration projects will exceed timelines and budgets, primarily due to inadequate upfront assessment and discovery.

A thorough migration assessment answers: Which of our 876 servers actually matter? Which ones talk to each other? Which are compliant? Which serve zero users? Which would cost more in the cloud? The outputs—application inventory, dependency maps, TCO comparison, and prioritized migration waves—form the blueprint for the entire migration program.

## Definition

**Portfolio Discovery** is the automated and manual process of identifying, cataloging, and documenting all IT assets (servers, databases, applications, network devices) in an environment.

**Migration Assessment** is the analysis of discovered assets to determine cloud readiness, migration strategy (7 R's), total cost of ownership (TCO), and migration sequencing.

**Key outputs**:
- Complete application inventory with metadata
- Dependency maps (application-to-application, application-to-infrastructure)
- Cloud readiness scores per application
- TCO comparison (on-premises vs cloud) per application
- Prioritized migration wave plan
- Risk register and mitigation strategies

## Concept Explanation

### The Assessment Process

```
Phase 1: Discovery (2-4 weeks)
  ├── Automated scanning (agents, agentless collectors)
  ├── Network traffic analysis (NetFlow, VPC Flow Logs)
  ├── CMDB import (existing configuration management database)
  └── Stakeholder interviews (application owners)

Phase 2: Analysis (2-4 weeks)
  ├── Dependency mapping (visualize connections)
  ├── Utilization analysis (CPU, memory, disk, network)
  ├── Cost analysis (TCO per application)
  ├── Licensing assessment (bring-your-own-license)
  └── Compliance mapping (PCI, HIPAA, GDPR scope)

Phase 3: Planning (2-4 weeks)
  ├── Strategy assignment (7 R's per application)
  ├── Wave sequencing (dependency-driven)
  ├── Business case (ROI, NPV, payback period)
  └── Risk assessment and mitigation plan
```

### Data Collection Methods

#### Agent-Based Discovery
```bash
# Deploy discovery agents to each server
# AWS Application Discovery Agent
wget https://s3.amazonaws.com/application-discovery-agent/linux/latest/aws-discovery-agent.tar.gz
tar -xzf aws-discovery-agent.tar.gz
sudo ./install -r us-east-1 -k $AWS_KEY -s $AWS_SECRET

# Agent collects:
# - System configuration (OS, CPU, RAM, disk)
# - Running processes and their resource usage
# - Network connections (source/destination IP:port)
# - Installed applications and versions
```

#### Agentless Discovery
```bash
# VMware vCenter collector (no agents on VMs)
aws discovery start-export-task \
  --export-data-format CSV \
  --filters '[
    {"name": "vcenter-ip", "values": ["192.168.1.100"], "condition": "EQUALS"}
  ]'

# Collects: VM specs, performance metrics, network flows
# Without installing software on individual VMs
```

### Dependency Mapping

```python
class DependencyMapper:
    def __init__(self):
        self.nodes = {}  # server → metadata
        self.edges = []  # (source, dest, port, protocol, bytes)
    
    def analyze_connections(self):
        # Identify application groups
        groups = self._cluster_by_communication()
        
        # Identify critical dependencies
        for group in groups:
            group.dependencies = self._find_external_deps(group)
            group.data_flows = self._find_data_flows(group)
        
        return groups
    
    def _cluster_by_communication(self):
        """
        Group servers that heavily communicate with each other.
        These are likely the same application or tightly coupled services.
        For migration, they should move together (same wave).
        """
        graph = nx.Graph()
        for node in self.nodes:
            graph.add_node(node.id)
        for edge in self.edges:
            if edge.bytes > 1_000_000:  # > 1MB communication
                graph.add_edge(edge.source, edge.dest, weight=edge.bytes)
        
        # Community detection for application grouping
        from networkx.algorithms import community
        return community.louvain_communities(graph)
```

### TCO Analysis

```python
class TCOAnalyzer:
    def compare_costs(self, app, migration_strategy):
        on_prem = self._calculate_on_prem_cost(app)
        cloud = self._calculate_cloud_cost(app, migration_strategy)
        
        return {
            'app_name': app.name,
            'strategy': migration_strategy,
            'on_prem_monthly': on_prem['total'],
            'cloud_monthly': cloud['total'],
            'monthly_savings': on_prem['total'] - cloud['total'],
            'annual_savings': (on_prem['total'] - cloud['total']) * 12,
            'migration_cost': cloud['migration_effort_cost'],
            'payback_months': cloud['migration_effort_cost'] / (on_prem['total'] - cloud['total']),
            'details': {
                'on_prem': on_prem,
                'cloud': cloud
            }
        }
    
    def _calculate_on_prem_cost(self, app):
        return {
            'hardware_depreciation': app.server_count * 500,
            'power_cooling': app.server_count * 200,
            'data_center_space': app.server_count * 150,
            'licensing': app.license_monthly_cost,
            'administration': app.admin_hours_monthly * 75,
            'network_bandwidth': app.bandwidth_cost_monthly,
            'total': sum_of_above
        }
    
    def _calculate_cloud_cost(self, app, strategy):
        if strategy == 'REHOST':
            # Lift & shift: EC2 instances similar to on-prem specs
            instance_cost = estimate_ec2_cost(app.cpu, app.ram, app.storage)
            return {
                'compute': instance_cost,
                'storage': estimate_ebs_cost(app.storage_gb),
                'data_transfer': app.monthly_transfer_gb * 0.09,
                'support': total * 0.10,  # Business support
                'migration_effort_cost': app.server_count * 2000,  # Per-server migration
                'total': sum_of_above
            }
        elif strategy == 'REFACTOR':
            # Serverless/containerized: pay-per-use, lower baseline
            return {
                'compute': estimate_serverless_cost(app.requests, app.duration),
                'storage': estimate_managed_db_cost(app.data_size),
                'data_transfer': app.monthly_transfer_gb * 0.09,
                'migration_effort_cost': app.function_points * 5000,  # Refactoring
                'total': sum_of_above
            }
```

### Cloud Readiness Assessment

```python
def assess_cloud_readiness(app):
    score = 0
    max_score = 100
    findings = []
    
    # Architecture factors
    if app.stateless:
        score += 20
    else:
        findings.append("Stateful: may need session management redesign")
    
    if app.os in ['Linux', 'Windows Server 2016+']:
        score += 15
    else:
        findings.append(f"OS {app.os}: limited cloud support or EOL")
    
    if app.database in ['PostgreSQL', 'MySQL', 'MSSQL', 'Oracle 12c+']:
        score += 15
    else:
        findings.append(f"Database {app.database}: may need migration or replatform")
    
    # Operational factors
    if app.ci_cd_enabled:
        score += 15
    
    if app.infrastructure_as_code:
        score += 10
    
    if app.automated_backups:
        score += 10
    
    # Risk factors (negative scoring)
    if app.contains_pii:
        score -= 5
        findings.append("Contains PII: requires compliance controls")
    
    if app.last_modified_date < datetime(2019, 1, 1):
        score -= 10
        findings.append("Not modified in 5+ years: unknown internals")
    
    if app.single_point_of_failure:
        score -= 10
        findings.append("SPOF: requires HA redesign for cloud resilience")
    
    readiness = 'HIGH' if score > 70 else 'MEDIUM' if score > 40 else 'LOW'
    
    return {
        'score': max(0, score),
        'readiness': readiness,
        'findings': findings,
        'recommendation': get_migration_recommendation(score, findings)
    }
```

### Wave Planning

```python
class WavePlanner:
    def plan_waves(self, applications, dependency_graph):
        waves = []
        
        # WAVE 1: Quick wins (high readiness, low risk, no dependencies)
        wave1 = [a for a in applications 
                 if a.readiness == 'HIGH' 
                 and not has_unresolved_deps(a, dependency_graph)]
        waves.append({'name': 'Wave 1 - Quick Wins', 'apps': wave1})
        
        # WAVE 2: Core applications (medium readiness, manageable deps)
        remaining = [a for a in applications if a not in wave1]
        wave2 = [a for a in remaining if a.tier <= 2 and a.readiness >= 'MEDIUM']
        waves.append({'name': 'Wave 2 - Core Applications', 'apps': wave2})
        
        # WAVE 3: Complex migrations (low readiness, high value)
        remaining2 = [a for a in remaining if a not in wave2]
        wave3 = [a for a in remaining2 if a.tier <= 2]
        waves.append({'name': 'Wave 3 - Complex Applications', 'apps': wave3})
        
        # WAVE 4: Everything else
        wave4 = [a for a in remaining2 if a not in wave3]
        waves.append({'name': 'Wave 4 - Remaining', 'apps': wave4})
        
        return waves
```

## Layman's Explanation

### The Moving Survey
Before a moving company gives you a quote, they send a surveyor (discovery agent) to walk through your house:

- **Inventory**: "You have 3 couches, 12 chairs, 1 grand piano, 8 bookshelves with 400 books each." (Server inventory)
- **Dependencies**: "The master bedroom furniture can't be moved until the hallway renovation is complete." (Application dependencies)
- **Special Handling**: "The wine collection needs climate-controlled transport." (Compliance/security requirements)
- **What Stays**: "That pool table? It was built in the basement. It can't be moved." (Retain)
- **What Goes to Storage**: "These boxes you haven't opened since 2010?" (Retire - or archive data)
- **Cost Comparison**: "Moving your current setup vs. buying new furniture at the destination." (TCO)
- **Sequencing**: "We'll move the kitchen on Monday (Wave 1), the living room on Tuesday (Wave 2), the garage on Wednesday." (Wave planning)

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Assessment Accuracy Drives Everything**: A 10% error in assessing how many servers exist or how they're connected ripples through every subsequent phase—wrong instance types, missed dependencies, failed cutovers. Discovery must be comprehensive.
- **Dependency Resolution**: The #1 cause of migration failures is undiscovered dependencies. Application A gets migrated, and suddenly Application B stops working because nobody knew B called an API on A's old IP address. Thorough network traffic analysis over weeks (not hours) is essential.
- **Licensing Complexity**: Oracle licensing in the cloud is fundamentally different from on-premises. BYOL (Bring Your Own License) vs license-included instances. Windows Server licensing on shared tenancy vs dedicated hosts. License optimization can save or cost millions.
- **TCO vs Total Value**: TCO comparisons often favor on-premises for steady-state workloads (cloud is per-hour; on-prem is sunk cost). But TCO misses: elasticity value (scale down on weekends), innovation velocity (new cloud services), and operational agility (provision in minutes not months).

### Business Impact
- **Portfolio Rationalization Savings**: Discovery typically finds 15-25% of servers that can be decommissioned immediately (dev/test environments left running, zombie servers, duplicate systems). For a 1,000-server environment, that's $500K-$1M/year immediate savings.
- **Avoided Migration Failures**: Comprehensive dependency mapping prevents the most expensive type of migration failure—when a migrated application breaks another application that wasn't known to depend on it. These failures cost $50K-$500K in emergency remediation.
- **Informed Business Case**: Without a real TCO analysis, the business case for cloud migration is a guess. With it, the CFO can compare the 18-month payback period of rehosting vs the 5-year ROI of refactoring.

## On-Premises Examples

### Manual Discovery Script
```bash
#!/bin/bash
# Quick manual discovery for Linux servers

echo "=== SERVER INVENTORY ===" > discovery-$(hostname).txt
echo "Hostname: $(hostname)" >> discovery-$(hostname).txt
echo "OS: $(cat /etc/os-release | grep PRETTY_NAME)" >> discovery-$(hostname).txt
echo "CPU: $(nproc) cores" >> discovery-$(hostname).txt
echo "RAM: $(free -h | grep Mem | awk '{print $2}')" >> discovery-$(hostname).txt
echo "Disk: $(df -h / | tail -1 | awk '{print $2}')" >> discovery-$(hostname).txt

echo "" >> discovery-$(hostname).txt
echo "=== LISTENING PORTS ===" >> discovery-$(hostname).txt
ss -tlnp >> discovery-$(hostname).txt

echo "" >> discovery-$(hostname).txt
echo "=== ACTIVE CONNECTIONS ===" >> discovery-$(hostname).txt
ss -tn state established >> discovery-$(hostname).txt

echo "" >> discovery-$(hostname).txt
echo "=== INSTALLED PACKAGES ===" >> discovery-$(hostname).txt
dpkg -l | grep -E 'nginx|apache|mysql|postgres|redis|java|python|node' >> discovery-$(hostname).txt

echo "" >> discovery-$(hostname).txt
echo "=== RUNNING SERVICES ===" >> discovery-$(hostname).txt
systemctl list-units --type=service --state=running >> discovery-$(hostname).txt
```

### nmap Network Scanning
```bash
nmap -sV -p 1-65535 10.0.0.0/16 -oX subnet-discovery.xml

# Discovered:
# 10.0.1.5:22 OpenSSH 7.4
# 10.0.1.10:443 nginx 1.18
# 10.0.2.20:5432 PostgreSQL 12
# 10.0.2.30:3306 MySQL 5.7
```

## AWS Examples

### AWS Application Discovery Service
```bash
# Agent-based discovery
aws discovery start-data-collection-by-agent-ids \
  --agent-ids agent-001 agent-002

# Agentless discovery (VMware)
aws discovery start-export-task \
  --export-data-format CSV

# View discovered servers
aws discovery describe-agents
aws discovery list-server-neighbors  # Dependency visualization
```

### AWS Migration Hub
```hcl
# Central tracking of migration progress
resource "aws_migration_hub_config" "main" {
  home_region = "us-east-1"
}

# Import discovered servers into Migration Hub
resource "aws_migration_hub_refactor_spaces" "app" {
  name = "order-processing"
}

# Track migration status per application
# Migration Hub shows:
# - Discovery progress
# - Server grouping into applications
# - Migration status per server
# - Aggregated progress dashboard
```

## GCP Examples

### Migrate for Compute Engine Discovery
```bash
# Deploy discovery client in vCenter
gcloud compute instances create migrate-connector \
  --image-family=migrate-for-compute-engine \
  --image-project=migration-solutions

# Migrate for Compute Engine discovers:
# - VM inventory (CPU, RAM, disk, OS)
# - Performance metrics (max/avg CPU, network IO)
# - Network dependencies (source/dest tracking)
# - Application groupings (VMs that communicate)
```

### StratoZone (Portfolio Assessment)
```bash
# Cloud-native assessment platform acquired by Google
# Analyzes on-premises infrastructure and recommends:
# - Cloud readiness scores
# - TCO comparison
# - Optimal cloud configuration
# - Migration priority
```

## Azure Examples

### Azure Migrate Discovery
```bash
# Deploy Azure Migrate appliance (VMware/Hyper-V)
az migrate project create --name myMigrationProject --resource-group myRG

# Discovery appliance collects:
# - VM configurations and performance data
# - Installed applications and SQL Server instances
# - Network dependencies (agentless)
# - Web app configurations (IIS, Tomcat)

# Dependency visualization
az migrate dependency visualize --group-name app-group-1

# Assessment
az migrate assessment create \
  --project-name myMigrationProject \
  --group-name app-group-1 \
  --assessment-name app-assessment
```

### Azure Migrate TCO Calculator
```bash
# Total Cost of Ownership analysis
az migrate assessment create \
  --reserved-instance RI_3year \
  --azure-hybrid-benefit true \
  --azure-offer Pay-As-You-Go

# Output:
# - Azure compute cost (monthly/yearly)
# - Azure storage cost
# - Networking cost
# - Comparison with on-premises cost
```

## Summary

| Assessment Phase | Duration | Key Artifacts | Risks if Skipped |
|-----------------|----------|--------------|-----------------|
| Discovery | 2-4 weeks | Server inventory, app catalog | Missing servers, failed migrations |
| Dependency Mapping | 2-4 weeks | Dependency graph, app groups | Broken integrations, cascading failures |
| TCO Analysis | 1-2 weeks | Cost comparison, business case | Budget overruns, poor ROI |
| Readiness Assessment | 1-2 weeks | Strategy per app (7 R's) | Wrong migration approach |
| Wave Planning | 1-2 weeks | Prioritized migration schedule | Chaos, resource conflicts |

Migration assessment is the cheapest phase of migration and the most expensive to skip. Every hour spent on discovery and planning prevents 10+ hours of emergency remediation during migration. The architect's role is to ensure the assessment is thorough, data-driven, and directly informs the migration strategy for every application in the portfolio.
