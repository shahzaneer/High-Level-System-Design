# Security Monitoring & Incident Response

## Introduction
Security Monitoring and Incident Response (IR) are the processes of continuously observing systems for security threats, detecting malicious activity, and responding to contain and remediate incidents. The famous cybersecurity adage captures it perfectly: "There are two types of companies—those that have been breached and those that don't know it yet." IBM's 2023 Cost of a Data Breach report found the average breach takes 207 days to identify and 73 days to contain. Organizations with mature security monitoring reduce this to under 200 days total, saving an average of $1.76 million per breach.

Security monitoring has evolved from simple log review to sophisticated SIEM (Security Information & Event Management) systems, SOAR (Security Orchestration, Automation, and Response) platforms, and cloud-native detection pipelines. For solution architects, building observable, auditable systems is as critical as building functional ones—you cannot protect what you cannot see.

## Definition

**Security Monitoring** is the continuous collection, analysis, and alerting on security-relevant events across infrastructure, applications, and user activity to detect threats, policy violations, and anomalous behavior.

**Incident Response** is the structured approach to addressing and managing the aftermath of a security breach or attack. It encompasses preparation, detection, containment, eradication, recovery, and post-incident analysis.

### Key Components

- **SIEM**: Aggregates logs from diverse sources, correlates events, and generates alerts (Splunk, Azure Sentinel, Chronicle)
- **SOAR**: Automates repetitive IR tasks, orchestrates tools, and provides playbook-driven response (Splunk Phantom, Cortex XSOAR, Azure Logic Apps)
- **EDR/XDR**: Endpoint/Extended Detection and Response for host-level visibility (CrowdStrike, Microsoft Defender, SentinelOne)
- **Cloud-Native Detection**: Cloud provider-specific detection and response (AWS GuardDuty, GCP Security Command Center, Azure Security Center)
- **UEBA**: User and Entity Behavior Analytics—ML-based anomaly detection on user and entity behavior

## Concept Explanation

### The Security Monitoring Pipeline

```
┌──────────────────────────────────────────────────────────┐
│                    DATA SOURCES                           │
│  CloudTrail │ VPC Flow Logs │ App Logs │ OSQuery │ EDR  │
│  WAF Logs   │ ALB Logs      │ DB Logs  │ DNS Logs │ IAM  │
└────────────┬─────────────────────────────────────────────┘
             │
    ┌────────▼─────────┐
    │  LOG AGGREGATION  │  (S3 + Athena, Cloud Logging, Log Analytics)
    └────────┬─────────┘
             │
    ┌────────▼─────────┐
    │  NORMALIZATION    │  (Parquet/OCSF schema, structured)
    └────────┬─────────┘
             │
    ┌────────▼─────────┐
    │  DETECTION        │  (Rules, ML, threat intel)
    │  • Signature-based│  Known IoCs, CVEs
    │  • Anomaly-based  │  Deviation from baseline
    │  • Behavioral     │  Unusual patterns
    └────────┬─────────┘
             │
    ┌────────▼─────────┐
    │  ALERTING         │  (PagerDuty, Slack, Opsgenie)
    └────────┬─────────┘
             │
    ┌────────▼─────────┐
    │  RESPONSE         │  (Auto-remediation, manual investigation)
    └──────────────────┘
```

### Critical Log Sources

#### 1. Audit Trail (API Activity)
Every API call, console action, and SDK invocation:

```json
{
  "eventVersion": "1.09",
  "userIdentity": {
    "type": "IAMUser",
    "arn": "arn:aws:iam::123456789:user/alice",
    "accessKeyId": "AKIAIOSFODNN7EXAMPLE"
  },
  "eventTime": "2024-01-15T09:30:00Z",
  "eventSource": "ec2.amazonaws.com",
  "eventName": "RunInstances",
  "sourceIPAddress": "203.0.113.42",
  "requestParameters": {
    "instanceType": "p4d.24xlarge",  // $32/hr GPU instance—suspicious!
    "maxCount": 100  // Crypto mining?
  }
}
```

**What to detect**:
- API calls from unusual IPs or locations
- Resource creation outside approved regions
- Privilege escalation (AttachRolePolicy, CreateUser, PutUserPolicy)
- Disabling security controls (StopLogging, DeleteFlowLogs, DisableGuardDuty)
- Data exfiltration indicators (massive S3 GetObject, unusual data transfer patterns)

#### 2. VPC Flow Logs (Network Traffic)
```
srcaddr dstaddr srcport dstport protocol packets bytes action
10.0.1.5 10.0.3.10 45678 5432 6 50 45000 ACCEPT
10.0.1.5 8.8.8.8 45679 443 6 5 500 ACCEPT
10.0.1.5 169.254.169.254 45680 80 6 2000 1500000 REJECT  ← Blocked IMDSv2?
```

**What to detect**:
- Port scanning within VPC (1 source → many destinations on many ports)
- Data exfiltration (large byte transfers to unknown external IPs)
- Direct internet access from database subnets (policy violation)
- Suspicious protocols (outbound SSH from web tier)
- Blocked requests to IMDS (potential instance metadata service attack)

#### 3. Application & Database Logs
```json
{
  "timestamp": "2024-01-15T09:31:00Z",
  "service": "order-api",
  "user": "alice",
  "action": "query",
  "endpoint": "/api/orders/search",
  "query_params": {"q": "' OR 1=1 --"},
  "status_code": 403,
  "waf_action": "BLOCK",
  "source_ip": "45.33.32.156"
}
```

#### 4. Auth Logs
```
Successful login: alice, San Francisco, Chrome 120, MacOS
Failed login: alice, Moscow, Firefox, Windows (possible credential stuffing)
Successful login: alice, San Francisco (10 mins after Moscow attempt—MFA prompt?)
Password change: alice (from unknown device)
MFA device removed: alice (account takeover in progress?)
API key created: alice (unusual time: 3:15 AM)
```

### Detection Engineering

#### Signature-Based Detection (Known Threats)
```sql
-- Detect GuardDuty/CSPM disabling
SELECT * FROM cloudtrail_logs
WHERE eventName IN (
  'StopLogging', 'DeleteTrail', 'DeleteFlowLogs',
  'DisableGuardDuty', 'DeleteDetector',
  'DisableSecurityHub', 'DisableConfig'
)
AND eventTime > now() - interval '1' hour
```

#### Anomaly-Based Detection (Deviations from Baseline)
```python
import numpy as np
from collections import defaultdict

class AnomalyDetector:
    def __init__(self, baseline_window=168):  # 1 week hourly
        self.baselines = {}
    
    def build_baseline(self, historical_data):
        # Build per-hour baselines for each metric
        for hour in range(24):
            hourly_data = [d for d in historical_data if d['hour'] == hour]
            self.baselines[hour] = {
                'mean': np.mean([d['rps'] for d in hourly_data]),
                'std': np.std([d['rps'] for d in hourly_data])
            }
    
    def is_anomaly(self, current_hour, current_value, std_dev_threshold=3):
        baseline = self.baselines.get(current_hour)
        if not baseline:
            return False
        return abs(current_value - baseline['mean']) > std_dev_threshold * baseline['std']
```

#### Behavioral Detection (UEBA)
```python
# Detect impossible travel: user logged in from two distant locations
# within time less than physically possible

def detect_impossible_travel(login_events):
    for i in range(len(login_events) - 1):
        event1 = login_events[i]
        event2 = login_events[i + 1]
        
        if event1['user'] != event2['user']:
            continue
        
        distance_km = haversine(event1['lat'], event1['lon'],
                                event2['lat'], event2['lon'])
        time_diff_hours = (event2['timestamp'] - event1['timestamp']) / 3600
        
        # Impossible if travel would require > 900 km/h (commercial flight speed)
        if time_diff_hours > 0 and (distance_km / time_diff_hours) > 900:
            alert(f"Impossible travel: {event1['user']} from "
                  f"{event1['location']} to {event2['location']} "
                  f"in {time_diff_hours:.1f} hours")
```

### Incident Response Lifecycle (NIST SP 800-61)

```
Preparation → Detection & Analysis → Containment → Eradication → Recovery → Post-Incident
    │               │                    │            │            │           │
    ├─ IR plan      ├─ SIEM alerts       ├─ Isolate   ├─ Remove    ├─ Restore  ├─ Lessons
    ├─ Playbooks    ├─ User reports      │  systems   │  malware   │  services │  learned
    ├─ Tools        ├─ Threat intel      ├─ Revoke    ├─ Patch     ├─ Verify   ├─ Update
    └─ Training     └─ Anomaly detection │  credentials│           │  integrity│  playbooks
                                         └─ Block IPs └─ Rebuild  └─ Monitor  └─ Report
```

#### Automated Response Playbooks

```python
# Example SOAR playbook: automated response to GuardDuty finding
def guardduty_finding_response(finding):
    severity = finding['Severity']
    finding_type = finding['Type']
    
    if finding_type == 'UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration':
        # Immediate: revoke credentials
        revoke_iam_credentials(finding['Resource']['AccessKeyDetails'])
        
        # Isolate the EC2 instance
        isolate_instance(finding['Resource']['InstanceDetails']['InstanceId'])
        
        # Take forensic snapshot
        take_ebs_snapshot(finding['Resource']['InstanceDetails']['InstanceId'])
        
        # Notify
        notify_security_team(finding)
    
    elif finding_type == 'CryptoCurrency:EC2/BitcoinTool.B!DNS':
        # Crypto mining detected
        terminate_instance(finding['Resource']['InstanceDetails']['InstanceId'])
        notify_security_team(finding)
    
    elif finding_type == 'Recon:EC2/PortProbe':
        # Port probing - moderate severity
        if severity >= 7:
            block_ip_in_nacl(finding['Service']['Action']['NetworkConnectionAction']['RemoteIpDetails']['IpAddressV4'])
        notify_security_team(finding)
```

### Cloud-Native Investigation Queries

```sql
-- AWS CloudTrail: find all actions by a compromised user
SELECT eventTime, eventName, eventSource, sourceIPAddress, requestParameters
FROM cloudtrail_logs
WHERE useridentity.arn = 'arn:aws:iam::123456789:user/alice'
  AND eventTime BETWEEN '2024-01-15T00:00:00Z' AND '2024-01-15T23:59:59Z'
ORDER BY eventTime DESC

-- Find all public S3 buckets created in last 24 hours
SELECT eventTime, requestParameters->>'bucketName' as bucket
FROM cloudtrail_logs
WHERE eventName = 'CreateBucket'
  AND json_extract(requestParameters, '$.x-amz-acl') LIKE '%public%'
  AND eventTime > now() - interval '24' hour

-- Detect privilege escalation attempts
SELECT eventTime, useridentity.arn, eventName, sourceIPAddress
FROM cloudtrail_logs
WHERE eventName IN ('AttachRolePolicy', 'PutUserPolicy', 'CreateUser',
                    'CreateAccessKey', 'AddUserToGroup')
  AND errorCode IS NULL  -- successful
  AND eventTime > now() - interval '24' hour
```

## Layman's Explanation

### The Bank Security Operations Center
A bank has a security operations center (SOC):

**Security Monitoring**: The bank has cameras (logs) covering every entrance, teller station, vault, and hallway. Security analysts watch the feeds 24/7 (SIEM). The cameras don't just record—they have smart analytics:
- Someone running through the lobby → alert
- Someone at the vault at 3 AM → alert
- Same person at the New York branch and the London branch 30 minutes apart → alert (impossible travel)
- Someone trying 100 different key cards on the same door → alert (brute force)

**Incident Response**: When an alert fires, the IR playbook kicks in:
1. **Containment**: Lock the affected area. If a teller's badge was stolen, revoke it immediately.
2. **Eradication**: Find how they got in. Was it a stolen badge? Change all badge protocols. Was it a hole in the wall? Patch it.
3. **Recovery**: Once the threat is removed, restore normal operations. Verify all systems are clean.
4. **Post-Incident**: What did we learn? Update the playbooks. Maybe we need cameras at that entrance (better logging).

### The Smoke Detector Analogy
Security monitoring is like smoke detectors in a building:
- You don't install ONE detector—you install them in every room (comprehensive logging)
- Detectors are tested regularly (log validation, rule testing)
- Some detectors are simple (signature-based: detects specific smoke particle signature)
- Some are smart (anomaly-based: detects unusual temperature rise)
- When one goes off, you don't just silence it—you investigate (every alert must be triaged)
- The fire department (IR team) has pre-planned response procedures for different building zones

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Log Architecture**: What gets logged? At what level? Where do logs go? How long are they retained? Logs have cost (storage, bandwidth, compute for analysis), but insufficient logging means you're blind during an incident. Balance: log all security-relevant events (API calls, auth, network flows) at INFO; application at WARN for production.
- **Immutable Log Storage**: Logs must be append-only and immutable. If an attacker can delete CloudTrail logs, they can cover their tracks. Use S3 Object Lock, Cloud Logging log buckets with retention locks, or Azure immutable storage.
- **Detection vs Prevention**: Detection complements prevention (you can't prevent everything). Detection requires: comprehensive logging, well-tuned rules, trained analysts (or managed service). Prevention without detection is a Maginot Line—once bypassed, you have no idea.
- **Multi-Account Log Aggregation**: In multi-account AWS/cloud architectures, aggregate all logs to a dedicated security account that nobody (not even admins) can modify logs in. The security account is read-only for everyone except an automated pipeline.

### Business Impact
- **Breach Cost Reduction**: Organizations with fully deployed security AI and automation saved $1.76M per breach (IBM 2023). The combination of faster detection and automated containment directly reduces financial impact.
- **Compliance**: SOC 2 CC4.0 (monitoring), CC7.0 (incident response). PCI DSS Requirement 10 (logging and monitoring). HIPAA Security Rule 164.308(a)(6) (security incident procedures). Every compliance framework requires monitoring and IR.
- **Cyber Insurance**: Insurers now require evidence of SIEM, 24/7 monitoring, and documented IR plans. Without them, you either can't get coverage or pay exponentially higher premiums.
- **Executive & Board Confidence**: When the board asks "Are we secure?" the architect must answer with data: "We process X million security events daily, have Y detection rules, detected Z threats last quarter, mean time to detect is T minutes." Security monitoring provides that data.

## On-Premises Examples

### ELK Stack (Elasticsearch, Logstash, Kibana)
```yaml
# Filebeat configuration for log shipping
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
      - /var/log/nginx/error.log
    fields:
      log_type: nginx

  - type: log
    paths:
      - /var/log/audit/audit.log
    fields:
      log_type: auditd

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
```

```json
// Elasticsearch detection rule (Kibana SIEM)
{
  "rule_id": "detect-brute-force",
  "name": "SSH Brute Force Detection",
  "description": "Detects multiple failed SSH login attempts",
  "risk_score": 47,
  "severity": "medium",
  "type": "threshold",
  "query": "event.type:authentication_failure AND system.auth.ssh.event:*",
  "threshold": {
    "field": "host.name",
    "value": 10,
    "cardinality": [{"field": "user.name", "value": 3}]
  },
  "interval": "5m",
  "from": "now-6m"
}
```

### Wazuh (Open-Source XDR + SIEM)
```xml
<!-- Wazuh agent configuration for file integrity monitoring -->
<syscheck>
  <directories check_all="yes" realtime="yes">/etc,/usr/bin,/usr/sbin</directories>
  <directories check_all="yes">/bin,/sbin,/boot</directories>
  
  <ignore>/etc/mtab</ignore>
  <ignore type="sregex">.log$</ignore>
</syscheck>
```

### OSQuery (Host Visibility)
```sql
-- OSQuery: query endpoints like a database
-- List all listening ports
SELECT pid, port, address, protocol FROM listening_ports;

-- Find all users with UID 0 (root equivalent)
SELECT uid, username FROM users WHERE uid = 0;

-- Find all crontab entries (persistence mechanism)
SELECT command, path FROM crontab;

-- Find processes with deleted binaries (often malware)
SELECT pid, name, path FROM processes WHERE on_disk = 0;
```

## AWS Examples

### AWS GuardDuty (Managed Threat Detection)
```hcl
resource "aws_guarddetector_detector" "main" {
  enable = true
  
  features {
    name = "EKS_AUDIT_LOG"
    enable = true
  }

  features {
    name = "RDS_LOGIN_EVENTS"
    enable = true
  }

  features {
    name = "S3_DATA_EVENTS"
    enable = true
  }

  features {
    name = "LAMBDA_NETWORK_LOGS"
    enable = true
  }

  datasources {
    s3_logs {
      enable = true
    }
  }
}
```

### AWS Security Hub (Centralized Security View)
```hcl
resource "aws_securityhub_account" "main" {}
resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:::standards/cis-aws-foundations-benchmark/v/1.4.0"
}

resource "aws_securityhub_standards_subscription" "pci" {
  standards_arn = "arn:aws:securityhub:us-east-1::standards/pci-dss/v/3.2.1"
}
```

### CloudTrail + GuardDuty Automated Response
```python
import boto3

def lambda_handler(event, context):
    finding = event['detail']['findings'][0]
    
    if finding['type'] == 'UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration':
        iam = boto3.client('iam')
        ec2 = boto3.client('ec2')
        
        # Revoke compromised credentials
        access_key_id = finding['resource']['accessKeyDetails']['accessKeyId']
        iam.update_access_key(
            UserName=finding['resource']['accessKeyDetails']['userName'],
            AccessKeyId=access_key_id,
            Status='Inactive'
        )
        
        # Isolate EC2 instance
        ec2.modify_instance_attribute(
            InstanceId=finding['resource']['instanceDetails']['instanceId'],
            Groups=[]  # Remove security groups → no network
        )
        
        # Create Jira ticket
        create_ticket(finding)
```

### AWS Config Rules (Continuous Compliance)
```hcl
resource "aws_config_config_rule" "s3_public_read" {
  name = "s3-bucket-public-read-prohibited"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }

  scope {
    compliance_resource_types = ["AWS::S3::Bucket"]
  }
}

resource "aws_config_config_rule" "restricted_ssh" {
  name = "restricted-ssh-in-security-groups"

  source {
    owner             = "AWS"
    source_identifier = "INCOMING_SSH_DISABLED"
  }
}
```

## GCP Examples

### Security Command Center (SCC)
```bash
# Enable Security Command Center Premium
gcloud services enable securitycenter.googleapis.com

# View findings
gcloud scc findings list organizations/ORG_ID/sources/- \
  --filter="severity=\"CRITICAL\" OR severity=\"HIGH\""

# Enable Event Threat Detection (GuardDuty equivalent)
gcloud services enable eventthreatdetection.googleapis.com

# Enable Security Health Analytics
gcloud services enable securityhealthanalytics.googleapis.com
```

### Cloud Audit Logs + Logging
```bash
# Create log sink for security events to BigQuery
gcloud logging sinks create security-sink \
  bigquery.googleapis.com/projects/my-project/datasets/security_logs \
  --log-filter='resource.type="audited_resource" OR severity>=WARNING'

# Create log-based alert
gcloud logging metrics create admin-activity \
  --description="Count of admin actions" \
  --log-filter='protoPayload.methodName=~"setIamPolicy|setIamPermissions"'
```

### Chronicle (Google's Cloud-Native SIEM)
```python
from google.cloud import chronicle

# Chronicle ingests logs from Cloud Logging, feeds into threat detection
# Rules written in YARA-L language

# Example YARA-L detection rule:
# rule detect_crypto_mining {
#   meta:
#     author = "security-team"
#     description = "Detect crypto mining activity"
#     severity = "High"
#   events:
#     $process.metadata.event_type = "PROCESS_LAUNCH"
#     $process.principal.process.command_line = /(xmrig|cpuminer|ccminer)/
#   condition:
#     $process
# }
```

## Azure Examples

### Microsoft Sentinel (Cloud-Native SIEM + SOAR)
```bash
# Enable Sentinel on Log Analytics workspace
az monitor log-analytics workspace create \
  --workspace-name security-workspace \
  --resource-group myResourceGroup

az sentinel onboarding-state create \
  --workspace-name security-workspace \
  --resource-group myResourceGroup \
  --name default

# Connect data connectors
# Azure AD, Azure Activity, Microsoft Defender for Cloud, Firewall
```

```kusto
// KQL: Detection query
// Detect impossible travel
SigninLogs
| where ResultType == 0  // Successful sign-in
| summarize
    Locations = make_set(Location),
    FirstSeen = min(TimeGenerated),
    LastSeen = max(TimeGenerated)
    by UserPrincipalName
| where array_length(Locations) > 1
| mv-expand Locations to typeof(string)
| order by FirstSeen asc
```

### Microsoft Defender for Cloud
```bash
# Enable enhanced security features
az security pricing create \
  --name "VirtualMachines" \
  --tier "Standard"

az security pricing create \
  --name "SqlServers" \
  --tier "Standard"

az security pricing create \
  --name "Containers" \
  --tier "Standard"
```

### Azure Logic Apps (SOAR Automation)
```json
{
  "definition": {
    "triggers": {
      "When_a_response_to_an_Azure_Sentinel_alert_is_triggered": {
        "type": "Microsoft.Sentinel/alert"
      }
    },
    "actions": {
      "Condition_-_Severity": {
        "type": "If",
        "expression": "@greater(triggerBody()?['Severity'], 'Medium')",
        "actions": {
          "Post_message_to_Slack": {
            "type": "Slack",
            "inputs": {
              "channel": "#security-alerts",
              "text": "@{triggerBody()?['DisplayName']} - Severity: @{triggerBody()?['Severity']}"
            }
          }
        }
      }
    }
  }
}
```

## Summary Decision Matrix

| Monitoring Capability | AWS | GCP | Azure |
|---------------------|-----|-----|-------|
| Threat Detection | GuardDuty | Event Threat Detection / SCC | Defender for Cloud |
| Centralized Security View | Security Hub | Security Command Center | Defender for Cloud |
| Audit Logs | CloudTrail | Cloud Audit Logs | Activity Log |
| Network Monitoring | VPC Flow Logs | VPC Flow Logs | NSG Flow Logs |
| SIEM (Native) | N/A (partner: Splunk, Datadog) | Chronicle | Microsoft Sentinel |
| SOAR Automation | Lambda + EventBridge | Cloud Functions + Eventarc | Logic Apps / Sentinel Playbooks |
| Compliance Checks | AWS Config | Security Health Analytics | Azure Policy |
| Data Classification | Macie | Cloud DLP | Purview / Information Protection |

Security monitoring is not optional for production systems. It is the institutional immune system—continuously scanning for threats, alerting when anomalies are detected, and triggering automated responses to contain damage. An architect must design systems to be not just functional but observable: every component emits security-relevant logs, logs are centralized and immutable, detection rules are tuned and tested, and response playbooks are practiced. The difference between a minor security incident and a business-crippling breach is often the quality of monitoring and the speed of response.
