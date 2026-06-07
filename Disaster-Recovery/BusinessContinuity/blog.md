# Business Continuity Planning

## Introduction
Business Continuity Planning (BCP) is the strategic framework that ensures critical business functions can continue during and after a disaster. While Disaster Recovery focuses specifically on IT systems, BCP encompasses the broader organizational response—people, processes, facilities, communications, and third-party dependencies. A technically flawless DR plan is worthless if employees can't access the failover site, customers don't know what's happening, or critical suppliers can't fulfill their obligations.

BCP emerged from the Y2K preparation era and evolved through 9/11, Hurricane Katrina, and COVID-19. Each event taught organizations that continuity requires more than technology—it requires documented procedures, trained personnel, tested communication channels, and executive-level commitment. For solution architects, BCP defines the business requirements (RTO, RPO, critical function prioritization) that DR architecture must meet.

## Definition

**Business Continuity Planning (BCP)** is the process of creating systems of prevention and recovery to deal with potential threats to an organization. It defines:

- **Business Impact Analysis (BIA)**: Identifies critical business functions and quantifies the impact of disruption
- **Recovery Strategies**: Defines how each critical function will be recovered within its RTO
- **Plan Development**: Documents procedures, roles, and responsibilities
- **Testing & Exercises**: Validates the plan through regular drills and updates

**Key BCP Roles**:
- **Incident Commander**: Senior leader who declares disaster and activates BCP
- **Crisis Communications Lead**: Manages internal and external communications
- **IT Recovery Lead**: Executes technical DR procedures
- **Business Unit Leads**: Represent their function's continuity needs
- **Legal/Compliance**: Ensures regulatory obligations are met during the incident

## Concept Explanation

### Business Impact Analysis (BIA)

```
BIA Process:
  1. Identify critical business functions
  2. Determine maximum tolerable downtime (MTD) for each
  3. Define RTO and RPO based on MTD
  4. Quantify financial impact per hour/day of downtime
  5. Map dependencies: people, technology, vendors, facilities
  6. Prioritize recovery sequence
```

```python
class BusinessImpactAnalysis:
    def __init__(self):
        self.functions = {}
    
    def assess_function(self, name, revenue_impact_per_hour, 
                         regulatory_impact, reputation_impact):
        score = 0
        score += revenue_impact_per_hour / 1000  # Normalize
        score += regulatory_impact * 5       # Regulatory = high weight
        score += reputation_impact * 3       # Reputation = medium-high weight
        
        if score > 80:
            return {'tier': 'TIER_1', 'rto_minutes': 15, 'rpo_minutes': 0}
        elif score > 50:
            return {'tier': 'TIER_2', 'rto_minutes': 240, 'rpo_minutes': 15}
        else:
            return {'tier': 'TIER_3', 'rto_hours': 24, 'rpo_hours': 4}

# Example BIA for an e-commerce platform
bia = BusinessImpactAnalysis()
bia.assess_function('Order Processing', 
    revenue_impact_per_hour=50000,
    regulatory_impact=3,  # PCI DSS
    reputation_impact=8)
# Result: TIER_1, RTO=15min, RPO=0

bia.assess_function('Marketing Website', 
    revenue_impact_per_hour=5000,
    regulatory_impact=0,
    reputation_impact=3)
# Result: TIER_3, RTO=24hrs, RPO=4hrs

bia.assess_function('Analytics Dashboard', 
    revenue_impact_per_hour=1000,
    regulatory_impact=0,
    reputation_impact=1)
# Result: TIER_3, RTO=48hrs, RPO=24hrs
```

### RTO/RPO Mapping to Architecture

```
TIER 1 (Critical - RTO ≤ 15 min, RPO ≤ 0):
  Architecture: Active/Active multi-region, synchronous replication
  Examples: Order processing, payment gateway, authentication service
  
TIER 2 (Important - RTO ≤ 4 hrs, RPO ≤ 15 min):
  Architecture: Warm standby, cross-region read replicas
  Examples: Inventory management, customer support portal
  
TIER 3 (Non-critical - RTO ≤ 48 hrs, RPO ≤ 24 hrs):
  Architecture: Backup & Restore
  Examples: Analytics dashboard, reporting, internal tools
```

### Crisis Communication Plan

```
Communication Timeline:

T+0 minutes:   Incident detection
T+5 minutes:   Incident Commander notification
T+15 minutes:  War room activation (virtual/physical)
T+30 minutes:  Initial internal communication (Slack, email)
                - "We are investigating an incident affecting [services]"
T+60 minutes:  Status page update (external)
                - First public communication
                - Acknowledge, don't speculate
T+2 hours:     Regular updates begin (every 30-60 minutes)
T+Resolved:    Post-incident summary
T+24 hours:    Preliminary post-mortem
T+5 days:      Full post-mortem published

Principles:
  - Communicate early, even if you don't have all the answers
  - Never lie or minimize; customers can tell
  - Provide a timeline for the next update ("We'll update in 30 minutes")
  - Have pre-approved message templates for common scenarios
```

### Runbook Design

```python
class DisasterRunbook:
    """
    Step-by-step, executable instructions.
    Written so that a sleep-deprived engineer at 3 AM can follow them.
    """
    
    def __init__(self):
        self.runbook = {
            'title': 'Regional Disaster - us-east-1 Failure',
            'owner': 'SRE Team',
            'last_tested': '2024-06-15',
            'approval_required': 'Incident Commander or SRE Director',
            'estimated_duration': '45 minutes',
            'backout_plan': 'Reverse steps in section 7'
        }
    
    def step_1_assess(self):
        """
        1. Verify the outage is real (not a monitoring false alarm)
           - Check https://health.aws.amazon.com
           - Check #sre-alerts Slack channel
           - Check statuspage.io for provider status
        2. If confirmed: ESCALATE to Incident Commander
        """
        pass
    
    def step_2_notify(self):
        """
        1. Send initial notification:
           Template: "DR_RUNBOOK_REGIONAL_FAILURE_ACTIVATED"
           Recipients: incident-command@company.com, sre@company.com
        2. Post in #incident-response Slack
        """
        pass
    
    def step_3_failover_dns(self):
        """
        1. Login to Route 53 console (or use break-glass credentials)
        2. Navigate to Hosted Zone: example.com
        3. Manually override failover: 
           aws route53 update-record-set → force secondary
        4. Confirm propagation: dig api.example.com from multiple locations
        """
        pass
    
    def step_4_promote_database(self):
        """
        1. Promote Aurora replica in eu-west-1:
           aws rds promote-read-replica --db-instance-identifier orders-db-dr
        2. Wait for promotion: aws rds wait db-instance-available
        3. Update SSM Parameter /prod/database/host to new endpoint
        4. Restart application services to pick up new config
        """
        pass
    
    def step_5_validate(self):
        """
        1. Run smoke tests against DR endpoint
        2. Verify: login → search → add to cart → checkout (test card)
        3. Check monitoring dashboards in DR region
        4. Verify order processing flow end-to-end
        """
        pass
    
    def step_6_communicate(self):
        """
        1. Update status page: "Failover to DR region complete"
        2. Notify customer support (they can inform customers)
        3. Update #incident-response with current status
        """
        pass
```

### Dependency Mapping

```
CRITICAL PATH ANALYSIS:
  Order Processing depends on:
    └── Authentication Service (Tier 1)
    └── Payment Gateway (Tier 1, third-party: Stripe)
    └── Inventory Service (Tier 2)
    └── Shipping Calculator (Tier 2, third-party: EasyPost)
    └── Customer Notification (Tier 3)
    
  If PAYMENT GATEWAY is down:
    → Order Processing cannot complete
    → Accept orders but defer payment capture (risk-based decision)
    → Requires pre-approved business decision and legal review
```

### Vendor/Supplier Continuity

```python
class VendorBCPAssessment:
    """Assess if critical vendors can meet your RTO"""
    
    def assess_vendor(self, vendor_name, your_rto, sla):
        risk = {
            'vendor': vendor_name,
            'your_rto_minutes': your_rto,
            'vendor_sla_minutes': sla.get('response_time'),
            'vendor_rto_minutes': sla.get('recovery_time'),
            'last_bcp_test': sla.get('last_test_date'),
            'single_point_of_failure': True
        }
        
        # Vendor can't meet our RTO
        if sla.get('recovery_time', 9999) > your_rto:
            risk['gap'] = f"Vendor RTO ({sla['recovery_time']}min) > Our RTO ({your_rto}min)"
            risk['mitigation'] = "Implement multi-vendor or graceful degradation"
        
        # Vendor hasn't tested recently
        if sla.get('last_test_date'):
            days_since_test = (datetime.now() - sla['last_test_date']).days
            if days_since_test > 180:
                risk['gap'] = f"Vendor BCP not tested in {days_since_test} days"
                risk['mitigation'] = "Request vendor BCP test results or run joint test"
        
        return risk
```

## Layman's Explanation

### The Wedding Planner's Emergency Kit
A wedding planner (Business Continuity Planner) plans for a perfect wedding day but prepares for disaster:

- **BIA**: The cake and the vows are Tier 1 (critical). The photo booth is Tier 3 (nice to have). Budget reflects this.
- **RTO/RPO**: If the flowers don't arrive (outage), the wedding can proceed with 0 minutes RTO (substitute flowers from the garden). If the officiant is late, RTO is up to 30 minutes (guests can wait).
- **Runbook**: The planner has a printed checklist: "If caterer doesn't show → Call backup caterer (contact in phone), move cocktail hour up by 30 minutes, notify venue coordinator."
- **Communications**: The planner tells the bride "We have a flower situation. It's handled. I'll update you in 15 minutes." No panic. No oversharing.
- **Vendor Continuity**: The photographer has a backup photographer on call. The cake baker has an assistant who knows all the recipes.
- **Testing**: The rehearsal dinner (DR test) caught that the sound system cables were incompatible with the venue's speakers. Fixed before the real day.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Architecture Matches BIA**: The BIA defines the tier for each business function. The architecture must deliver the RTO/RPO for each tier. Tier 1 functions on Backup & Restore architecture violates the BIA. The architect bridges the business requirement to the technical implementation.
- **Dependency Chain Analysis**: A Tier 3 service that a Tier 1 service depends on MUST be treated as Tier 1. Dependency analysis often reveals "hidden" critical services that are architected as non-critical.
- **People Dimension**: The most resilient architecture fails if the only person who knows the DR procedure is on vacation. BCP addresses this through documentation, cross-training, and automation.
- **Communication Architecture**: Status pages, internal comms, and customer notification systems must survive the disaster. Hosting the status page on the same infrastructure it monitors is a common anti-pattern.

### Business Impact
- **Insurance & Liability**: Many business interruption insurance policies require documented BCP. Directors and officers can be personally liable for failing to prepare for foreseeable business disruptions.
- **Customer Retention**: Customers tolerate honest communication during outages. They leave when there's radio silence. A BCP that includes a communication plan directly impacts customer retention.
- **Regulatory Survival**: Financial services (OCC, FINRA) require BCP. Healthcare (HIPAA) requires contingency plans. Public companies (SOX) must disclose material risks to business continuity. Non-compliance threatens the business license itself.

## On-Premises Examples

### BCP Template (Document Structure)
```markdown
# Business Continuity Plan - [Organization Name]
## Version: 2.0 | Last Tested: 2024-06-15

### 1. Plan Activation Criteria
- Major incident declared by Incident Commander
- System outage exceeding Tier 1 RTO
- Facility unavailability (fire, flood, power loss)
- Third-party critical service failure > 2 hours

### 2. Roles and Responsibilities
| Role | Primary | Secondary |
|------|---------|-----------|
| Incident Commander | Jane (CTO) | Bob (VP Eng) |
| IT Recovery Lead | Mike (SRE Lead) | Sarah (DevOps) |
| Communications | Lisa (PR) | Tom (Marketing) |

### 3. Critical Functions Recovery Sequence
Priority order:
1. Authentication (RTO: 15 min) - Unblocks all other services
2. Order Processing (RTO: 15 min) - Revenue-generating
3. Payment Gateway (RTO: 30 min) - Depends on Auth
4. Customer Support Portal (RTO: 4 hrs)
5. Analytics Dashboard (RTO: 48 hrs)

### 4. Communication Templates
[Pre-approved messages for: initial alert, status updates, resolution]
```

### Automated Notification System
```python
import smtplib
from twilio.rest import Client

class EmergencyNotificationSystem:
    def __init__(self):
        self.call_tree = {
            'Incident Commander': {'phone': '+1...', 'email': 'ic@company.com'},
            'SRE Lead': {'phone': '+1...', 'email': 'sre@company.com'},
            'SRE Secondary': {'phone': '+1...', 'email': 'sre2@company.com'},
        }
    
    def notify(self, incident_type, severity):
        for role, contact in self.call_tree.items():
            # SMS to primary phone
            self.send_sms(contact['phone'], 
                f"BCP ACTIVATED: {incident_type} Severity: {severity}. "
                f"Join war room: https://meet.google.com/xxx")
            
            # Email with full details
            self.send_email(contact['email'],
                subject=f"[BCP-ACTIVATED] {incident_type}",
                body=self._generate_initial_report(incident_type))
    
    def escalate_if_no_response(self, notified_role, timeout_minutes=15):
        # If primary doesn't acknowledge, notify secondary
        pass
```

## AWS Examples

### AWS Elastic Disaster Recovery (DRS)
```hcl
resource "aws_drs_replication_configuration_template" "main" {
  configuration_template_id = "default"

  replication_server_instance_type = "t3.medium"
  
  create_public_ip                 = false
  use_dedicated_replication_server = false

  default_large_staging_disk_type = "gp3"
  ebs_encryption                  = "DEFAULT"

  pit_policy {
    enabled            = true
    interval           = 4  # Point-in-time snapshot every 4 hours
    retention_duration = 7  # Retain for 7 days
    units              = "HOURS"
  }

  tags = {
    Environment = "BCP"
  }
}
```

### AWS Resilience Hub (BCP Assessment)
```bash
aws resiliencehub create-app \
  --name "OrderProcessing" \
  --description "E-commerce order processing system" \
  --resiliency-policy-arn "arn:aws:resiliencehub:..."

# Assess against RTO/RPO policy
aws resiliencehub start-app-assessment \
  --app-arn "arn:aws:resiliencehub:..."

# Generates:
# - Resilience score
# - RTO/RPO gap analysis
# - Recommendations (add multi-AZ, enable backups, etc.)
# - Estimated cost of recommendations
```

## GCP Examples

### Cloud Monitoring Uptime Checks
```bash
gcloud monitoring uptime create app-uptime-check \
  --resource-type=uptime_url \
  --resource-labels=host=api.example.com \
  --checker-type=STATIC_IP_CHECKERS \
  --period=60s \
  --timeout=10s \
  --alert-policy=bcp-critical-alert
```

## Azure Examples

### Azure Site Recovery (ASR) BCP Integration
```bash
az site-recovery protection-container mapping create \
  --name "primary-to-dr-mapping" \
  --vault-name myRecoveryVault \
  --primary-protection-container "primary-container" \
  --recovery-protection-container "dr-container"
```

## Summary

| BCP Component | Frequency | Owner | Artifact |
|--------------|-----------|-------|----------|
| Business Impact Analysis | Annual | Business + IT | BIA document with tier assignments |
| Plan Review & Update | Semi-annual | BCP Coordinator | Updated BCP document |
| Tabletop Exercise | Quarterly | Incident Commander | Exercise minutes + action items |
| Technical DR Test | Semi-annual | IT Recovery Lead | Test report + gap analysis |
| Full Failover Test | Annual | CTO | Pass/fail report, lessons learned |

BCP is the bridge between business requirements and technical architecture. The solution architect translates "we can't be down for more than 4 hours" into a specific DR pattern, failover architecture, and recovery runbook. Without BCP, DR is technology without purpose. Without DR, BCP is strategy without execution. Together, they ensure the business survives whatever disaster strikes.
