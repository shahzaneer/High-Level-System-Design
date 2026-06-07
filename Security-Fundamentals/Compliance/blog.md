# Compliance, Governance & Data Privacy

## Introduction
Compliance, governance, and data privacy form the regulatory and operational framework that constrains and guides system architecture. Unlike other security domains that focus on preventing specific attacks, compliance ensures the organization meets externally imposed standards (PCI DSS, SOC 2, HIPAA, GDPR, FedRAMP) and internal governance policies. Data privacy specifically addresses the rights of individuals over their personal data, driven by regulations like GDPR (Europe), CCPA/CPRA (California), and LGPD (Brazil).

For solution architects, compliance is not a checkbox exercise at the end of a project—it is a design constraint that shapes data architecture, encryption strategy, access controls, logging, and operational procedures from day one. Architecting for compliance from the start is 10x cheaper than retrofitting a non-compliant system.

## Definition

**Compliance** is the state of conforming to rules, standards, policies, and laws that apply to an organization and its systems. It encompasses:
- **External Regulations**: PCI DSS, HIPAA, GDPR, CCPA, FedRAMP, SOC 2, ISO 27001
- **Industry Standards**: NIST Cybersecurity Framework, CIS Benchmarks, CSA Cloud Controls Matrix
- **Internal Policies**: Organizational security policies, data classification standards, change management procedures

**Governance** is the framework of policies, processes, and controls that ensures IT resources are used appropriately, securely, and efficiently. It includes:
- Resource provisioning standards (approved instance types, regions, services)
- Tagging and cost allocation policies
- Access review and certification processes
- Change management and approval workflows

**Data Privacy** is the aspect of data governance that deals with the proper handling of personal data, focusing on individuals' rights:
- Right to access (know what data is collected)
- Right to rectification (correct inaccurate data)
- Right to erasure (be forgotten)
- Right to data portability (receive data in machine-readable format)
- Right to object (opt out of processing)

## Concept Explanation

### Major Compliance Frameworks

#### PCI DSS (Payment Card Industry Data Security Standard)
The most prescriptive framework—explicit requirements for handling cardholder data:

```
12 Requirements:
1.  Install and maintain network security controls
2.  Apply secure configurations
3.  Protect stored account data
4.  Encrypt cardholder data in transit
5.  Protect against malware
6.  Develop and maintain secure systems
7.  Restrict access to cardholder data
8.  Identify and authenticate access
9.  Restrict physical access
10. Log and monitor all access
11. Test security regularly
12. Maintain security policies
```

**Architecture Impact**: CDE (Cardholder Data Environment) must be segmented from the rest of the network. Encryption everywhere. Strict logging. Annual penetration testing. QSA (Qualified Security Assessor) audit.

#### SOC 2 (Service Organization Control 2)
Trust Services Criteria focusing on service provider controls:

```
Five Trust Services Criteria:
  Security (Common Criteria - required):
    CC1: Control environment
    CC2: Communication and information
    CC3: Risk assessment
    CC4: Monitoring activities
    CC5: Control activities
    CC6: Logical and physical access controls
    CC7: System operations
    CC8: Change management
    CC9: Risk mitigation

  Availability (optional):
    A1: Availability monitoring, incident response, disaster recovery

  Confidentiality (optional):
    C1: Confidential information identification and protection

  Processing Integrity (optional):
    PI1: Processing integrity controls

  Privacy (optional):
    P1-P8: Privacy practices (aligned with GDPR)
```

**Architecture Impact**: Documented controls for access management, change management, monitoring, and incident response. Evidence of control effectiveness over a period (Type I = point in time, Type II = over 6-12 months).

#### HIPAA (Health Insurance Portability and Accountability Act)
Protects Protected Health Information (PHI/ePHI):

```
Key Rules:
  Privacy Rule: Patient rights over health data
  Security Rule: Administrative, physical, and technical safeguards
  Breach Notification Rule: Notify patients and HHS of breaches
```

**Architecture Impact**: BAA (Business Associate Agreement) required with all vendors. ePHI must be encrypted at rest and in transit. Access controls, audit controls, integrity controls, and authentication all mandated. PHI must never appear in logs, error messages, or analytics without de-identification.

#### GDPR (General Data Protection Regulation)
European regulation with extraterritorial reach (applies to any organization processing EU residents' data):

```
Key Principles:
  Lawfulness, fairness, and transparency
  Purpose limitation
  Data minimization
  Accuracy
  Storage limitation
  Integrity and confidentiality (security)
  Accountability

Data Subject Rights:
  Right to be informed
  Right of access (DSAR - Data Subject Access Request)
  Right to rectification
  Right to erasure ("right to be forgotten")
  Right to restrict processing
  Right to data portability
  Right to object
  Rights related to automated decision-making and profiling
```

**Architecture Impact**: Must be able to locate ALL data about a specific user (data inventory). Must be able to delete ALL copies, including backups and replicas (hard with eventually consistent systems). Must obtain explicit consent for data collection. Must notify regulators within 72 hours of discovering a breach.

### Compliance as Code

Modern cloud governance is implemented through policy-as-code, not manual review:

```rego
# OPA policy: enforce PCI DSS tagging requirements
package compliance.pci

# Require specific tags on all resources
deny[msg] {
    resource := input.resources[_]
    required_tags := {"Environment", "DataClassification", "CostCenter"}
    missing := required_tags - {tag | resource.tags[tag]}
    count(missing) > 0
    msg = sprintf("Resource %s missing tags: %v", 
                  [resource.id, missing])
}

# Block public S3 buckets for CDE
deny[msg] {
    resource := input.resources[_]
    resource.type == "aws_s3_bucket"
    resource.tags.DataClassification == "cardholder"
    resource.acl == "public-read"
    msg = sprintf("CDE bucket %s cannot be public", [resource.id])
}
```

```hcl
# AWS SCP: prevent disabling security controls
resource "aws_organizations_policy" "protect_security" {
  name = "protect-security-controls"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyDisablingSecurityServices"
        Effect = "Deny"
        Action = [
          "guardduty:DeleteDetector",
          "guardduty:Disassociate*",
          "cloudtrail:StopLogging",
          "cloudtrail:DeleteTrail",
          "config:DeleteConfigRule",
          "securityhub:DisableSecurityHub"
        ]
        Resource = "*"
      }
    ]
  })
}
```

### Data Privacy Architecture Patterns

#### Data Classification
```
Public:     Blog posts, documentation, marketing materials
            → No access restrictions
Internal:   Employee directory, internal wikis, non-sensitive metrics
            → Auth required, no external sharing
Confidential: Customer PII, financial data, source code
            → Auth + MFA, access logged, encryption required
Restricted: Cardholder data, PHI, credentials, secrets
            → Strict access controls, encryption, audit, DLP
```

#### Data Minimization
```python
# BAD: Return entire user object with sensitive fields
@app.route('/api/users/<user_id>')
def get_user(user_id):
    user = User.query.get(user_id)
    return jsonify(user.to_dict())
    # Returns: password_hash, ssn, credit_card_number, ...

# GOOD: Return only necessary fields
@app.route('/api/users/<user_id>')
def get_user(user_id):
    user = User.query.get(user_id)
    return jsonify({
        'id': user.id,
        'name': user.name,
        'email': user.email,
        'department': user.department
        # Only the fields needed by the frontend
    })
```

#### Data Deletion / Right to Erasure
```python
def delete_user_data(user_id):
    # 1. Anonymize in primary database
    db.execute(
        "UPDATE users SET email=NULL, name='DELETED_USER', phone=NULL "
        "WHERE id = ?", [user_id]
    )
    
    # 2. Delete from caches
    cache.delete(f"user:{user_id}")
    cache.delete(f"sessions:{user_id}")
    
    # 3. Delete from search indexes
    search_index.delete(f"user-{user_id}")
    
    # 4. Delete from analytics (or anonymize)
    analytics_db.execute(
        "UPDATE events SET user_id='ANONYMIZED' WHERE user_id = ?", [user_id]
    )
    
    # 5. Log deletion for audit
    audit_log.record({
        'action': 'DATA_DELETION',
        'user_id': user_id,
        'requested_by': g.current_user,
        'timestamp': datetime.utcnow(),
        'type': 'GDPR_RIGHT_TO_ERASURE'
    })
    
    # 6. Backup caveat: data in immutable backups may persist
    # Document this in privacy policy: "deleted from live systems within 30 days,
    # removed from backups within 90 days per backup rotation"
```

#### Data Residency / Sovereignty
```hcl
# Control where data is stored
resource "aws_s3_bucket" "eu_data" {
  bucket = "customer-data-eu"
}

resource "aws_s3_bucket_policy" "eu_only" {
  bucket = aws_s3_bucket.eu_data.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Deny"
      Principal = "*"
      Action    = "*"
      Resource  = "${aws_s3_bucket.eu_data.arn}/*"
      Condition = {
        StringNotEquals = {
          "s3:RequestObjectTag/region": "eu"
        }
      }
    }]
  })
}
```

### Audit Readiness

```
For every compliance framework, be prepared to demonstrate:

  POLICIES:     "Here is our written policy on data encryption"
  EVIDENCE:     "Here are the AWS Config rules proving all buckets are encrypted"
  LOGS:         "Here are the CloudTrail logs showing who accessed S3 buckets"
  TIMELINE:     "Here is evidence from the last 12 months"
  INDEPENDENCE: "Here is the auditor's separate read-only account access"
```

## Layman's Explanation

### Compliance: The Building Code Inspector
Building a house without compliance is like building without permits and inspections:

- **Building Code (PCI DSS, HIPAA)**: The minimum safety requirements. Your electrical wiring must meet code. Your staircases need handrails. You can't skip these just because they're "inconvenient."
- **Building Inspector (Auditor)**: Visits periodically to verify you followed the code. Shows up at random (surprise audit). Writes up violations that you must fix by a deadline.
- **Permits (Evidence)**: You can't just say "trust me, the wiring is safe." You must show permits, inspection reports, and contractor licenses. In compliance, that's CloudTrail logs, Config rules, penetration test reports.
- **GDPR (Right to Be Forgotten)**: Like a law that says any homeowner can demand the builder erase all records of them from the blueprints. The builder must find every copy, every backup, every archived plan and remove that person's data.

### Data Privacy: The Medical File Analogy
Your medical file at a doctor's office is private data:
- Only your doctor and nurses can see it (access control)
- The file cabinet is locked when the office is closed (encryption at rest)
- You can request a copy of everything in your file (right of access / DSAR)
- You can ask them to delete your records when you switch doctors (right to erasure)
- They can't sell your medical history to advertisers (purpose limitation)
- If someone breaks in and steals files, the doctor must tell you (breach notification)

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Multi-Account Strategy for Compliance**: Separate AWS/GCP/Azure accounts for different compliance scopes. PCI/CDE in a dedicated account with stricter controls. This prevents a misconfiguration in the dev environment from becoming a compliance finding.
- **Data Residency Architecture**: GDPR requires EU residents' data to stay in the EU unless adequate protections exist. Architect must choose: single-region deployment (simpler but non-global) or multi-region with data residency controls (complex but global).
- **Encryption Everywhere**: PCI DSS requires encryption everywhere. HIPAA requires encryption everywhere. GDPR Article 32 requires appropriate technical measures (encryption is the industry standard). Architect for encryption-on-by-default at every layer: disks, databases, backups, logs, and transit.
- **Immutable Logging**: Every framework requires tamper-proof audit trails. Use S3 Object Lock, Cloud Logging log buckets with retention policies, or Azure immutable blobs. Logs must survive even root account compromise.

### Business Impact
- **Revenue Enablement**: SOC 2 is a prerequisite for selling to enterprise customers. PCI DSS is required to process credit cards. FedRAMP is required to sell to the U.S. federal government. HIPAA is required for healthcare. Compliance opens markets.
- **Fine Avoidance**: GDPR fines up to 4% of global annual revenue or €20M (whichever is greater). Amazon was fined €746M (2021) for cookie consent violations. Meta was fined €1.2B (2023) for data transfers. These are existential amounts.
- **Customer Trust**: Privacy-conscious consumers (increasingly the majority) choose services with transparent data practices. Apple has made privacy a core product differentiator. Privacy-preserving architecture is a competitive advantage.
- **Operational Discipline**: The discipline of compliance—documented procedures, automated controls, regular testing—produces more resilient operations even beyond security. Change management reduces outages. Monitoring catches incidents faster.

### Designing for Compliance (Not Retrofitting)
1. Identify required frameworks before writing any code
2. Map data flows—where does data originate, transit, and rest?
3. Classify data at creation (not months later)
4. Automate controls via policy-as-code (CloudFormation Guard, OPA, Checkov, tfsec)
5. Design deletion workflows during data modeling, not during a DSAR panic
6. Assume an auditor will examine every access control, every log, every change

## On-Premises Examples

### OpenSCAP (Compliance Scanning)
```bash
# Scan system against PCI DSS profile
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_pci-dss \
  --results results.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Generate compliance report
oscap xccdf generate report results.xml > compliance-report.html
```

### PGP/GPG for Data Privacy
```bash
# Encrypt sensitive data at field level
gpg --encrypt --recipient compliance@company.com sensitive_data.csv

# Pseudonymization: replace identifiers with tokens
echo "user-123" | openssl dgst -sha256 -hmac "secret-key"
# Returns pseudonymized identifier that can't be reversed without key
```

### Data Retention Policy Automation
```bash
#!/bin/bash
# Purge data older than retention period
# Logs: 90 days, Analytics: 2 years, Backups: 7 years

find /var/log/app -type f -mtime +90 -delete -exec echo "Deleted: {}" \;
find /data/analytics -type f -mtime +730 -delete
```

## AWS Examples

### AWS Artifact (Compliance Reports)
```bash
# Download SOC 2, PCI DSS, HIPAA, ISO 27001 reports
aws artifact get-report \
  --report-id "arn:aws:artifact:::report/..." \
  --term-token "$TERM_TOKEN"
```

### AWS Config Conformance Packs
```hcl
# Deploy PCI DSS conformance pack
resource "aws_config_conformance_pack" "pci" {
  name = "pci-dss-4.0"

  template_body = <<EOT
Resources:
  S3BucketPublicReadProhibited:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: S3BucketPublicReadProhibited
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED
  EncryptedVolumes:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: EncryptedVolumes
      Source:
        Owner: AWS
        SourceIdentifier: ENCRYPTED_VOLUMES
EOT
}
```

### AWS Macie (Data Classification & Privacy)
```hcl
resource "aws_macie2_account" "main" {}

resource "aws_macie2_classification_job" "pii_scan" {
  name        = "pii-discovery-scan"
  job_type    = "ONE_TIME"
  s3_job_definition {
    bucket_definitions {
      account_id = data.aws_caller_identity.current.account_id
      buckets    = ["customer-data-bucket"]
    }
  }

  sampling_percentage = 100
}
```
Macie automatically discovers and classifies sensitive data: PII (names, SSNs, emails), financial data (credit cards, bank accounts), PHI, and credentials in S3 buckets.

### AWS Audit Manager
```hcl
resource "aws_auditmanager_assessment" "soc2" {
  name        = "soc2-assessment-2024"
  description = "SOC 2 Type II assessment evidence collection"

  framework_id = data.aws_auditmanager_framework.soc2.id

  roles {
    role_arn = aws_iam_role.audit_manager.arn
    role_type = "PROCESS_OWNER"
  }

  scope {
    aws_accounts {
      id = data.aws_caller_identity.current.account_id
    }
  }
}
```

### Data Deletion in AWS
```python
import boto3

def gdpr_delete_user(user_id):
    # 1. DynamoDB: delete item
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Users')
    table.delete_item(Key={'UserId': user_id})
    
    # 2. S3: delete user's objects
    s3 = boto3.client('s3')
    objects = s3.list_objects_v2(
        Bucket='user-uploads',
        Prefix=f'users/{user_id}/'
    )
    if 'Contents' in objects:
        s3.delete_objects(
            Bucket='user-uploads',
            Delete={'Objects': [{'Key': o['Key']} for o in objects['Contents']]}
        )
    
    # 3. CloudWatch Logs: delete log groups (if user-specific)
    logs = boto3.client('logs')
    logs.delete_log_group(logGroupName=f'/aws/lambda/user-{user_id}-processor')
    
    # 4. DynamoDB Streams / Kinesis: data is immutable for 24h-7d
    # Document: "Data removed from live systems within 24 hours.
    #  Stream data expires per retention policy (max 7 days)."
    
    # 5. S3 Backup: using S3 Lifecycle policies to expire old versions
    # Data in backups expires per backup lifecycle (30-90 days)
```

## GCP Examples

### Security Command Center (Compliance Dashboard)
```bash
# View compliance with CIS benchmarks
gcloud scc findings list organizations/ORG_ID/sources/- \
  --filter="category=\"CIS*\""

# Enable compliance reporting
gcloud scc assets list organizations/ORG_ID \
  --filter="securityCenterProperties.resourceType=\"google.cloud.storage.Bucket\""
```

### Cloud DLP (Data Loss Prevention)
```python
from google.cloud import dlp_v2

dlp_client = dlp_v2.DlpServiceClient()

# Inspect data for PII
response = dlp_client.inspect_content(
    request={
        "parent": "projects/my-project",
        "inspect_config": {
            "info_types": [
                {"name": "EMAIL_ADDRESS"},
                {"name": "CREDIT_CARD_NUMBER"},
                {"name": "US_SOCIAL_SECURITY_NUMBER"},
                {"name": "PHONE_NUMBER"}
            ],
            "min_likelihood": "POSSIBLE"
        },
        "item": {"value": "Customer email: alice@example.com, SSN: 123-45-6789"}
    }
)

# De-identify data
response = dlp_client.deidentify_content(
    request={
        "parent": "projects/my-project",
        "deidentify_config": {
            "info_type_transformations": {
                "transformations": [
                    {
                        "info_types": [{"name": "EMAIL_ADDRESS"}],
                        "primitive_transformation": {
                            "replace_with_info_type_config": {}
                        }
                        # Output: "Customer email: [EMAIL_ADDRESS], SSN: 123-45-6789"
                    }
                ]
            }
        },
        "item": {"value": "Customer email: alice@example.com"}
    }
)
```

### Organization Policy Service (Guardrails)
```bash
# Restrict resource locations (data residency)
gcloud org-policies set-policy restrict-locations.yaml

# restrict-locations.yaml:
# constraint: constraints/gcp.resourceLocations
# allowedValues:
#   - in:eu-locations

# Require OS Login (no SSH keys)
gcloud org-policies enable-enforce constraints/compute.requireOsLogin

# Disable service account key creation
gcloud org-policies enable-enforce constraints/iam.disableServiceAccountKeyCreation
```

## Azure Examples

### Microsoft Purview (Compliance & Data Governance)
```bash
# Purview catalog: automatically discover and classify data
az purview account create \
  --name myPurviewAccount \
  --resource-group myResourceGroup \
  --location eastus

# Data classification rules
az purview classification-rule create \
  --account-name myPurviewAccount \
  --name "CreditCardDetection" \
  --rule-type "regex" \
  --pattern "\b(?:\d[ -]*?){13,16}\b"
```

### Azure Policy (Compliance-as-Code)
```json
{
  "properties": {
    "displayName": "Require encryption on all storage accounts",
    "policyType": "BuiltIn",
    "mode": "All",
    "parameters": {},
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Storage/storageAccounts"
          },
          {
            "field": "Microsoft.Storage/storageAccounts/enableHttpsTrafficOnly",
            "equals": "false"
          }
        ]
      },
      "then": {
        "effect": "Deny"
      }
    }
  }
}
```

```bash
# Assign policy to subscription
az policy assignment create \
  --name require-encryption \
  --policy "require-storage-encryption" \
  --scope "/subscriptions/SUBSCRIPTION_ID"
```

### Data Subject Requests (DSR) Automation
```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import ResourceManagementClient
import requests

def process_dsar(user_email):
    results = {
        'user_email': user_email,
        'data_found': [],
        'data_deleted': []
    }
    
    # 1. Search Azure Cognitive Search index
    search_results = search_client.search(
        search_text=user_email,
        filter="data_classification eq 'PII'"
    )
    results['data_found'].extend([r['id'] for r in search_results])
    
    # 2. Retrieve from Cosmos DB
    container = cosmos_db.get_container_client('users')
    items = container.query_items(
        query="SELECT * FROM c WHERE c.email = @email",
        parameters=[{'name': '@email', 'value': user_email}]
    )
    
    # 3. Export data (Right to Access / Portability)
    user_data = list(items)
    export_json = json.dumps(user_data, indent=2)
    
    # 4. Delete data (Right to Erasure)
    for item in user_data:
        container.delete_item(item['id'], partition_key=item['partitionKey'])
        results['data_deleted'].append(item['id'])
    
    return results, export_json
```

### Compliance Manager
```bash
# View compliance score and recommendations
az security regulatory-compliance-standards list

# Assess against specific framework
az security regulatory-compliance-assessments list \
  --standard-name "PCI-DSS-3.2.1"
```

## Summary Decision Matrix

| Framework | Scope | Key Architecture Impact | Certification Method |
|-----------|-------|------------------------|---------------------|
| PCI DSS | Cardholder data | CDE segmentation, encryption everywhere, WAF required | QSA audit (annual) |
| SOC 2 | Service provider controls | Access controls, change mgmt, monitoring | CPA audit (annual) |
| HIPAA | Healthcare (PHI) | BAA with providers, encryption, audit controls | Self-assessment or HHS audit |
| GDPR | EU personal data | Data residency, right to erasure, 72hr breach notification | Self-compliance (fines for violations) |
| FedRAMP | US federal cloud | NIST 800-53 controls, continuous monitoring | 3PAO assessment |
| ISO 27001 | Information security | ISMS framework, risk management, 114 controls in Annex A | Accredited certification body |

Compliance, governance, and data privacy are architectural constraints that must be designed in from the start. The architect who treats them as post-hoc "check the box" activities creates systems that fail audits, incur fines, and lose customer trust. The architect who designs for compliance from day one—data classification at creation, encryption by default, immutable audit logs, automated policy enforcement, and data deletion workflows—creates systems that pass audits smoothly and earn customer confidence. In regulated industries, compliance architecture is not separate from system architecture—it IS system architecture.
