# Data Privacy & Protection

## Introduction
Data Privacy is the discipline of protecting personal information from unauthorized access, use, or disclosure while respecting individuals' rights over their data. It has transformed from a niche legal concern into a central architectural constraint driven by the explosive growth of data collection, the rise of data breaches, and the global proliferation of privacy regulations.

Unlike general data security (which protects ALL data equally), data privacy specifically addresses personal data: names, email addresses, device identifiers, location data, biometric data, and behavioral profiles. Modern architectures must treat personal data differently from other data—applying data minimization, purpose limitation, and individual rights controls at every layer of the stack. Solutions like Apple's differential privacy for analytics and Google's federated learning demonstrate that privacy-preserving architecture is technically achievable and commercially viable.

## Definition

**Data Privacy** is the right of individuals to control how their personal information is collected, used, shared, and retained. In systems architecture, it encompasses:

- **Data Minimization**: Collect only the data you need; retain it only as long as necessary
- **Purpose Limitation**: Use data only for the purpose for which it was collected
- **Data Subject Rights**: Enable individuals to access, correct, delete, and export their data
- **Consent Management**: Track and respect user consent for data collection and processing
- **Privacy by Design**: Embed privacy considerations into the architecture from inception, not as an afterthought
- **Data Protection**: Encrypt, anonymize, and pseudonymize personal data throughout its lifecycle

**Data Protection** is the technical implementation of privacy controls—the encryption, access controls, anonymization, and monitoring that protect personal data from breaches and misuse.

### Personal Data vs Sensitive Personal Data

| Category | Examples | Protection Level |
|----------|----------|-----------------|
| Personal Data (PII) | Name, email, phone, IP address | Encryption, access controls, audit |
| Sensitive Personal Data | Race, religion, health data, biometrics, sexual orientation, political opinions | Strongest protection; explicit consent required |
| Special Categories (GDPR Art 9) | Same as sensitive | Processing prohibited unless specific exemption applies |
| De-identified / Anonymized | Data that cannot be re-identified | Lower protection (outside scope of privacy laws if truly anonymous) |

## Concept Explanation

### Privacy by Design Principles (Ann Cavoukian)

1. **Proactive not Reactive; Preventative not Remedial**: Anticipate privacy issues before they happen
2. **Privacy as the Default Setting**: Maximum privacy by default; user shouldn't have to take action to protect privacy
3. **Privacy Embedded into Design**: Privacy is an integral component, not a bolt-on
4. **Full Functionality — Positive-Sum, not Zero-Sum**: Privacy AND security, not privacy VS security
5. **End-to-End Security — Full Lifecycle Protection**: Protect data from collection to destruction
6. **Visibility and Transparency**: Be open about data practices
7. **Respect for User Privacy**: Keep it user-centric

### Data Privacy Engineering Patterns

#### Data Minimization at Collection
```python
# BAD: Collect everything "just in case"
@app.route('/api/signup', methods=['POST'])
def signup():
    user = User(
        name=request.json['name'],
        email=request.json['email'],
        phone=request.json.get('phone'),
        address=request.json.get('address'),
        birth_date=request.json.get('birth_date'),
        ssn=request.json.get('ssn'),  # WHY?
        income=request.json.get('income')  # WHY?
    )

# GOOD: Collect only what's needed, document the purpose
# Purpose: Create account for order processing
# Legal Basis: Contract performance (GDPR Art 6.1.b)
# Retention: Account lifetime + 2 years after account deletion
REQUIRED_SIGNUP_FIELDS = {
    'name': 'Account identification - required for order delivery',
    'email': 'Account identification + order notifications - contract performance',
}

@app.route('/api/signup', methods=['POST'])
def signup():
    data = {}
    for field, purpose in REQUIRED_SIGNUP_FIELDS.items():
        data[field] = request.json[field]
    
    user = User(**data)
    user.data_purposes = REQUIRED_SIGNUP_FIELDS  # Store purpose mapping
```

#### Purpose Limitation
```python
class DataPurposeTracker:
    """Track why each piece of data was collected and prevent off-purpose use"""
    
    def __init__(self):
        self.purpose_database = {
            'email': {
                'purposes': ['account_auth', 'order_notifications'],
                'legal_basis': 'contract_performance',
                'retention': 'account_lifetime_plus_2_years',
                'shareable': False
            },
            'browsing_history': {
                'purposes': ['product_recommendations'],
                'legal_basis': 'consent',
                'retention': '90_days',
                'shareable': False
            },
            'anonymized_analytics': {
                'purposes': ['business_analytics', 'product_improvement'],
                'legal_basis': 'legitimate_interest',
                'retention': '38_months',
                'shareable': True  # Anonymized and aggregated only
            }
        }
    
    def validate_use(self, data_field, requested_purpose):
        field_config = self.purpose_database.get(data_field)
        if not field_config:
            raise PrivacyViolation(f"No tracking for field: {data_field}")
        if requested_purpose not in field_config['purposes']:
            raise PrivacyViolation(
                f"Cannot use {data_field} for {requested_purpose}. "
                f"Only allowed for: {field_config['purposes']}"
            )
        return field_config
```

#### Anonymization Techniques

**k-Anonymity**: Each record is indistinguishable from at least k-1 other records:
```
Original:          k=2 anonymized:
Age: 34, ZIP: 12345  → Age: 30-40, ZIP: 1234*
Age: 36, ZIP: 12348  → Age: 30-40, ZIP: 1234*
```

**Differential Privacy**: Add calibrated noise to query results. Guarantees that the presence or absence of any individual in the dataset has a mathematically bounded impact on outputs:

```python
import numpy as np

class DifferentialPrivacy:
    def __init__(self, epsilon=1.0):
        """
        epsilon (ε): Privacy budget. Lower = more privacy, more noise.
        ε = 0.1: Very private (high noise)
        ε = 1.0: Standard privacy
        ε = 10: Weak privacy (low noise)
        """
        self.epsilon = epsilon
    
    def private_count(self, true_count, sensitivity=1):
        """Add Laplace noise to a count query"""
        scale = sensitivity / self.epsilon
        noise = np.random.laplace(0, scale)
        return max(0, int(true_count + noise))
    
    def private_average(self, values, lower_bound, upper_bound):
        """Differentially private average with clipping"""
        # Clip values to bounds (limits sensitivity)
        clipped = [min(max(v, lower_bound), upper_bound) for v in values]
        sensitivity = (upper_bound - lower_bound) / len(clipped)
        
        true_avg = sum(clipped) / len(clipped)
        noise = np.random.laplace(0, sensitivity / self.epsilon)
        return true_avg + noise

# Usage: Analytics without exposing individual data
dp = DifferentialPrivacy(epsilon=1.0)

total_users = 1000000
private_total = dp.private_count(total_users)
# Returns: 1000007 (close, but privacy-preserving)

# Track cumulative privacy budget
# When budget exhausted, stop answering queries
# Prevents reconstruction attacks from multiple queries
```

**Tokenization**: Replace sensitive data with non-sensitive tokens:
```python
import hashlib
import secrets

class TokenizationService:
    def __init__(self):
        self.token_store = {}  # In production: encrypted KV store
    
    def tokenize(self, sensitive_value):
        token = secrets.token_urlsafe(32)
        self.token_store[token] = {
            'value': sensitive_value,
            'created_at': datetime.utcnow()
        }
        return token
    
    def detokenize(self, token):
        # Requires authentication + authorization + audit
        audit_log.record(f"Detokenization requested for {token}")
        return self.token_store[token]['value']

# Usage
token = tokenization_service.tokenize("4111-1111-1111-1111")
db.execute("INSERT INTO payments (card_token) VALUES (?)", [token])
# Card number never stored in application database
```

### Consent Management Architecture

```python
class ConsentManager:
    def __init__(self):
        self.consent_store = {}  # In production: versioned consent storage
    
    def record_consent(self, user_id, purpose, version, granted):
        consent_record = {
            'user_id': user_id,
            'purpose': purpose,
            'version': version,  # Privacy policy version
            'granted': granted,
            'timestamp': datetime.utcnow(),
            'ip_address': request.remote_addr,
            'user_agent': request.user_agent.string
        }
        self.consent_store[f"{user_id}:{purpose}"] = consent_record
        audit_log.record('CONSENT_RECORDED', consent_record)
    
    def has_valid_consent(self, user_id, purpose):
        record = self.consent_store.get(f"{user_id}:{purpose}")
        if not record or not record['granted']:
            return False
        
        # Check if consent has expired (privacy policy update)
        current_policy_version = "3.2"
        if record['version'] != current_policy_version:
            return False  # Require re-consent
        
        return True

# API endpoint that requires consent
@app.route('/api/recommendations')
def get_recommendations():
    user_id = g.current_user['sub']
    if not consent_manager.has_valid_consent(user_id, 'personalized_ads'):
        return jsonify({'error': 'Consent required for personalized recommendations'}), 403
    return recommendation_engine.generate(user_id)
```

### Data Lifecycle Automation

```python
from apscheduler.schedulers.background import BackgroundScheduler

class DataLifecycleManager:
    def __init__(self):
        self.scheduler = BackgroundScheduler()
        
        # Schedule: Anonymous analytics > 38 months → delete
        self.scheduler.add_job(
            self.purge_old_analytics,
            'cron',
            hour=3,
            minute=0
        )
        
        # Schedule: Soft-deleted accounts > 30 days → hard delete
        self.scheduler.add_job(
            self.hard_delete_accounts,
            'cron',
            hour=4,
            minute=0
        )
    
    def purge_old_analytics(self):
        cutoff = datetime.utcnow() - timedelta(days=38*30)
        count = analytics_db.execute(
            "DELETE FROM events WHERE created_at < ? AND user_id != 'ANONYMIZED'",
            [cutoff]
        )
        audit_log.record(f"Purged {count} old analytics records")
    
    def hard_delete_accounts(self):
        cutoff = datetime.utcnow() - timedelta(days=30)
        accounts = db.execute(
            "SELECT id FROM users WHERE deleted_at < ? AND deletion_stage = 'SOFT'",
            [cutoff]
        ).fetchall()
        
        for account in accounts:
            self.hard_delete_user_data(account['id'])
```

## Layman's Explanation

### Data Privacy: The Diary Analogy
Your diary is personal data. You keep it in your room. Privacy means:

- You decide who reads it (consent)
- Your parents don't read it without permission (access control)
- If you give someone a key to your room (data sharing), you know exactly what they can see
- When you throw away old diaries, you shred them (secure deletion)
- If you write about someone else, they have a right to know (data subject rights)
- You don't keep diaries from when you were 5 years old if you don't need them (data minimization)

### Differential Privacy: The Classroom Survey
A teacher wants to know how many students play video games but promises not to reveal individual answers.

Without differential privacy: The teacher collects exact data. If someone hacks the teacher's computer, they know exactly who plays video games.

With differential privacy: Each student flips a coin. If heads, they answer truthfully. If tails, they flip another coin and answer "yes" on heads and "no" on tails. The teacher can calculate an accurate aggregate (adjusting for the 25% random noise) but can NEVER know any individual student's true answer. Even if the database is breached, individual privacy is mathematically protected.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Data Inventory and Classification**: You cannot protect data you don't know exists. Every system must have a data catalog: what data is collected, where it's stored, how it flows, who accesses it, and when it's deleted. This is the foundational prerequisite for all privacy controls.
- **Anonymization vs Pseudonymization**: Anonymized data (irreversibly de-identified) is outside the scope of most privacy laws. Pseudonymized data (can be re-identified with additional information) is still personal data. The architectural choice between them has profound legal and functional implications.
- **Data Residency**: Where data is stored determines which laws apply. EU data in a US data center potentially violates GDPR unless Standard Contractual Clauses (SCCs) or an Adequacy Decision exists. Multi-region architectures must account for data sovereignty.
- **Deletion Architecture**: True deletion in distributed systems is hard. Data replicates across regions, exists in backups, caches, log files, analytics databases, and message queues. Architect for comprehensive data lifecycle management from day one—deletion plans for every copy in every system.

### Business Impact
- **Revenue & Trust**: Apple's privacy stance is a core product differentiator worth billions. Consumers increasingly choose privacy-respecting services. GDPR compliance enables access to the entire EU market—447 million people.
- **Fine Avoidance**: GDPR fines are up to 4% of global annual turnover. CCPA provides for statutory damages of $100-$750 per consumer per incident. These fines compound across millions of affected individuals.
- **Data Breach Cost Amplification**: A breach that exposes encrypted and tokenized data costs dramatically less than one exposing raw PII. Privacy-preserving architecture directly reduces breach impact.
- **M&A Diligence**: Acquiring companies heavily scrutinize privacy practices. Poor privacy posture can kill deals or significantly reduce valuation. Strong privacy architecture increases company value.

## On-Premises Examples

### Data Masking for Non-Production
```python
class DataMaskingPipeline:
    """Mask sensitive data when refreshing dev/staging from production"""
    
    @staticmethod
    def mask_email(email):
        local, domain = email.split('@')
        return f"user{hash(local) % 10000}@{domain}"
    
    @staticmethod
    def mask_name(name):
        return f"Test User {hash(name) % 10000}"
    
    @staticmethod
    def mask_credit_card(card_number):
        return f"4111-{hash(card_number) % 10000:04d}-XXXX-XXXX"
    
    def process(self, production_record):
        return {
            'id': production_record['id'],
            'name': self.mask_name(production_record['name']),
            'email': self.mask_email(production_record['email']),
            'payment_token': 'TOKEN-' + secrets.token_hex(8),
            # Order data preserved for realistic testing
            'order_total': production_record['order_total']
        }
```

### Retention Policy Enforcement
```bash
#!/bin/bash
# Automated data purging based on retention policy

# Application logs: 90 days
find /var/log/app -type f -mtime +90 -delete

# Database backups: 30 days locally, 7 years in immutable archive
find /backups/pg_dump -type f -mtime +30 -delete

# User event streams in Kafka: 7 days (configured via topic retention)
kafka-configs --alter --topic user-events \
  --add-config retention.ms=604800000

# Analytics data in ClickHouse: 38 months
clickhouse-client --query "
  ALTER TABLE analytics.events 
  DELETE WHERE timestamp < now() - INTERVAL 38 MONTH
"
```

## AWS Examples

### Amazon Macie (Sensitive Data Discovery)
```python
import boto3

macie = boto3.client('macie2')

# Create classification job
macie.create_classification_job(
    name='pii-discovery',
    jobType='ONE_TIME',
    s3JobDefinition={
        'bucketDefinitions': [{
            'accountId': '123456789',
            'buckets': ['customer-data']
        }]
    },
    samplingPercentage=100
)
# Macie finds: PII, PHI, credentials, financial data
# Generates findings with severity levels
```

### S3 Object Lock for Immutable Compliance Data
```hcl
resource "aws_s3_bucket" "compliance_logs" {
  bucket = "compliance-logs"
}

resource "aws_s3_bucket_object_lock_configuration" "lock" {
  bucket = aws_s3_bucket.compliance_logs.bucket

  rule {
    default_retention {
      mode = "COMPLIANCE"  # No one—not even root—can delete before retention expires
      years = 7
    }
  }
}

resource "aws_s3_bucket_versioning" "logs" {
  bucket = aws_s3_bucket.compliance_logs.bucket
  versioning_configuration {
    status = "Enabled"
  }
}
```

### AWS Glue DataBrew (Data Masking/Anonymization)
```python
# Glue DataBrew recipe for PII masking
{
  "Name": "pii-masking-recipe",
  "Steps": [
    {
      "Action": {
        "Operation": "HASH",
        "Parameters": {
          "sourceColumn": "email",
          "destinationColumn": "email",
          "hashType": "SHA256"
        }
      }
    },
    {
      "Action": {
        "Operation": "REPLACE_WITH_RANDOM",
        "Parameters": {
          "sourceColumn": "ssn",
          "destinationColumn": "ssn"
        }
      }
    }
  ]
}
```

## GCP Examples

### Cloud DLP (Sensitive Data Protection)
```python
from google.cloud import dlp_v2

dlp = dlp_v2.DlpServiceClient()

# Create inspection template for PII
template = {
    "inspect_config": {
        "info_types": [
            {"name": "PERSON_NAME"},
            {"name": "EMAIL_ADDRESS"},
            {"name": "PHONE_NUMBER"},
            {"name": "CREDIT_CARD_NUMBER"},
            {"name": "US_SOCIAL_SECURITY_NUMBER"}
        ],
        "min_likelihood": dlp_v2.Likelihood.POSSIBLE,
        "include_quote": True
    }
}

# Inspect BigQuery table
response = dlp.create_dlp_job(
    request={
        "parent": "projects/my-project",
        "inspect_job": {
            "storage_config": {
                "big_query_options": {
                    "table_reference": {
                        "project_id": "my-project",
                        "dataset_id": "analytics",
                        "table_id": "events"
                    }
                }
            },
            "inspect_template_name": "projects/my-project/inspectTemplates/pii-template",
            "actions": [
                {
                    "save_findings": {
                        "output_config": {
                            "table": {
                                "project_id": "my-project",
                                "dataset_id": "dlp_findings",
                                "table_id": "pii_findings"
                            }
                        }
                    }
                }
            ]
        }
    }
)

# De-identify data
response = dlp.deidentify_content(
    request={
        "parent": "projects/my-project",
        "deidentify_config": {
            "info_type_transformations": {
                "transformations": [
                    {
                        "info_types": [{"name": "EMAIL_ADDRESS"}],
                        "primitive_transformation": {
                            "crypto_hash_config": {
                                "crypto_key": {
                                    "transient": {"name": "temp-hash-key"}
                                }
                            }
                        }
                    }
                ]
            }
        },
        "item": {"value": "Contact: alice@example.com"}
    }
)
```

## Azure Examples

### Microsoft Purview Information Protection
```bash
# Create sensitivity labels
az purview sensitivity-label create \
  --name "Confidential-PII" \
  --display-name "Confidential - PII" \
  --description "Contains personally identifiable information" \
  --priority 1

# Auto-labeling rules
az purview auto-labeling-rule create \
  --name "auto-label-pii" \
  --sensitive-info-types "CreditCardNumber,SSN,EmailAddress" \
  --sensitivity-label "Confidential-PII"
```

### Azure Information Protection (AIP)
```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.security import SecurityCenter

# Apply sensitivity label to blob
blob_client.set_blob_metadata(
    metadata={
        'sensitivity_label': 'Confidential-PII',
        'retention_period': '7_years',
        'data_classification': 'PII'
    }
)
```

### Data Deletion Automation in Azure
```python
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
from datetime import datetime, timedelta

def process_dsar_deletion(user_email):
    """Process Data Subject Access Request - deletion"""
    credential = DefaultAzureCredential()
    
    # 1. Cosmos DB: find and delete user data
    cosmos_client = CosmosClient(COSMOS_ENDPOINT, credential)
    database = cosmos_client.get_database_client('app-db')
    containers = ['users', 'orders', 'analytics']
    
    for container_name in containers:
        container = database.get_container_client(container_name)
        items = container.query_items(
            query="SELECT * FROM c WHERE c.email = @email",
            parameters=[{'name': '@email', 'value': user_email}]
        )
        for item in items:
            container.delete_item(item['id'], partition_key=item['partitionKey'])
    
    # 2. Blob Storage: delete user files
    blob_client = BlobServiceClient(BLOB_ENDPOINT, credential)
    container_client = blob_client.get_container_client('user-uploads')
    
    blobs = container_client.list_blobs(name_starts_with=f'users/{user_email}/')
    for blob in blobs:
        container_client.delete_blob(blob.name)
    
    # 3. Log the deletion
    audit_client.log_event({
        'event': 'DSAR_DELETION',
        'user_email': user_email,
        'timestamp': datetime.utcnow().isoformat()
    })
```

## Summary Decision Matrix

| Privacy Capability | Purpose | Implementation |
|-------------------|---------|---------------|
| Data Discovery | Find where PII exists | AWS Macie, Cloud DLP, Purview |
| Data Classification | Label data sensitivity | Sensitivity labels, custom classification rules |
| Anonymization | Irreversible de-identification | Differential privacy, k-anonymity, aggregation |
| Pseudonymization | Reversible de-identification | Tokenization, deterministic hashing with salt |
| Consent Management | Track user preferences | Purpose-based consent records, preference centers |
| Data Deletion | Right to erasure | Comprehensive deletion pipelines across all stores |
| Data Residency | Geographic data controls | Region-locked buckets, geo-restriction policies |
| Privacy-Preserving Analytics | Insights without exposing individuals | Differential privacy, federated learning, secure enclaves |

Data privacy and protection are not compliance burdens—they are architectural differentiators. Systems that embed privacy from day one (privacy by design) are simpler to operate, cheaper to maintain, and earn greater customer trust than systems that retrofit privacy after a breach or regulatory action. The solution architect's responsibility is to treat personal data as a toxic asset: minimize collection, limit retention, protect comprehensively, and enable individual rights throughout the data lifecycle.
