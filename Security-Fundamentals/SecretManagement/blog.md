# Secret Management

## Introduction
Secret Management is the practice of securely storing, distributing, rotating, and auditing secrets—passwords, API keys, database credentials, TLS certificates, encryption keys, and access tokens—that systems and applications need to function. In traditional IT, secrets were often stored in configuration files, environment variables, or hardcoded in source code. The 2017 Uber breach (57M records) happened because AWS credentials were accidentally committed to a public GitHub repository. The 2022 Toyota breach (300K customers) was caused by an exposed access key in public source code.

These incidents are not rare—GitHub detected over 12 million new secrets leaked in public repositories in 2023 alone. Secret management has evolved from a security hygiene practice into a critical architectural requirement: every secret must have a known lifecycle, be retrievable only by authorized workloads, rotated automatically, and never appear in plaintext in logs, code, or config files.

## Definition
**Secret Management** is the set of tools, processes, and policies for securely handling digital authentication credentials throughout their lifecycle:

- **Creation**: Generating cryptographically strong secrets (random passwords, API keys, certificates)
- **Storage**: Encrypting secrets at rest with strong encryption, storing in tamper-evident vaults
- **Distribution**: Delivering secrets to authorized workloads without exposing them in transit or in intermediate systems
- **Rotation**: Periodically replacing secrets to limit exposure window if they are compromised
- **Revocation**: Immediately invalidating secrets that are known or suspected to be compromised
- **Audit**: Logging every access, rotation, and lifecycle event for compliance and forensics

**Key principles**:
- Never hardcode secrets in source code, configuration files, or infrastructure-as-code
- Never pass secrets through environment variables without encryption
- Every secret has an expiration date—infinite-lived credentials are a security anti-pattern
- Workloads should authenticate via identity (IAM, SPIFFE), not via persistent secrets, wherever possible

## Concept Explanation

### Secret Lifecycle

```
Create → Store → Distribute → Use → Rotate → Revoke
  │                                          │
  └──────────── Audit (every step) ──────────┘
```

#### 1. Dynamic Secrets (Ephemeral Credentials)
Instead of distributing long-lived passwords, the secret manager generates ephemeral credentials on demand, scoped to the requesting workload:

```python
# HashiCorp Vault: dynamic database credentials
import hvac

client = hvac.Client(url='https://vault.internal.com')

# Application requests temporary database credentials
response = client.secrets.database.generate_credentials(
    name='order-reader-role',
    mount_point='database'
)
# Returns: {"username": "v-user-order-abc123", "password": "temp-pwd-xyz",
#           "lease_duration": 3600, "lease_id": "..."}

# Vault automatically:
# 1. Creates database user with limited permissions
# 2. Sets password to expire in 1 hour
# 3. After lease expires, Vault revokes the user
# Application never stores long-lived credentials
```

#### 2. Envelope Encryption
The foundational pattern for storing secrets securely:

```
Encrypting a secret:
  1. Vault generates Data Encryption Key (DEK)
  2. Encrypts secret with DEK → ciphertext
  3. Encrypts DEK with Master Key (in HSM) → encrypted DEK
  4. Stores: ciphertext + encrypted DEK
  5. Master Key never leaves HSM

Decrypting a secret:
  1. Send encrypted DEK to HSM → receives decrypted DEK
  2. Decrypt ciphertext with DEK → plaintext secret
```

#### 3. Secret Rotation

```python
import boto3
import datetime

class SecretRotator:
    def __init__(self, secretsmanager_client, secret_id):
        self.sm = secretsmanager_client
        self.secret_id = secret_id
    
    def rotate(self):
        # 1. Create new secret version
        new_password = self._generate_password()
        self.sm.put_secret_value(
            SecretId=self.secret_id,
            SecretString=json.dumps({'password': new_password}),
            VersionStages=['AWSPENDING']
        )
        
        # 2. Update database with new password
        self._update_database_password(new_password)
        
        # 3. Test new password works
        if self._test_credentials(new_password):
            # 4. Mark new version as current
            self.sm.update_secret_version_stage(
                SecretId=self.secret_id,
                VersionStage='AWSCURRENT',
                MoveToVersionId=NEW_VERSION_ID
            )
        else:
            # Rollback
            self._abandon_rotation()
```

#### 4. Workload Identity (No Secrets at All)

The ultimate secret management is having no secrets to manage:

```hcl
# AWS: EC2 instance with IAM role (no credentials in instance)
resource "aws_iam_role" "app_role" {
  name = "app-instance-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

# Application uses AWS SDK which automatically gets temporary credentials
# from the instance metadata service—NO secret keys in code or config
```

```python
# Application code: no credentials needed
import boto3

# SDK automatically fetches temporary credentials from instance metadata
# or ECS task role, or Lambda execution role
secrets_client = boto3.client('secretsmanager')
db_password = secrets_client.get_secret_value(SecretId='prod/database/password')
```

### Secret Detection and Prevention

```bash
# git-secrets: prevent committing secrets to git
git secrets --install
git secrets --register-aws
git secrets --add 'AIza[0-9A-Za-z\\-_]{35}'  # GCP API key pattern

# detect-secrets: scan for secrets in codebase
detect-secrets scan --all-files > .secrets.baseline
detect-secrets audit .secrets.baseline

# truffleHog: deep git history scan for secrets
trufflehog git https://github.com/org/repo
```

### Pre-Commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
        exclude: package.lock.json
  
  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

## Layman's Explanation

### The Hotel Safe Deposit Box System
Imagine a luxury hotel with thousands of guests, each needing to access various amenities:

**Hardcoded secrets**: Writing your safe combination on a Post-it note and sticking it on the safe. Anyone who sees it can open it. (Credentials in source code)

**Environment variables**: Writing your safe combination on the hotel registration card. Every staff member who handles the card sees it. (Secrets in environment variables)

**Secret Management (Vault/KMS)**: The hotel uses a central safe deposit box system:
- You don't know ANY combinations. When you need to open your deposit box, you present your ID (workload identity).
- The clerk (secret manager) checks your authorization and opens the box for you.
- The box's combination changes every day (rotation). Yesterday's combination is useless today.
- Every time someone accesses a box, it's recorded with their ID and timestamp (audit).
- If you lose your ID, the clerk can immediately disable it (revocation) without changing anyone else's access.
- VIP guests get temporary assistants (dynamic secrets) who can only open specific boxes for the next 2 hours, after which their access expires automatically.

### Why It Matters: The Stolen Laptop
An employee's laptop is stolen. The thief finds:
- **Bad**: Plaintext passwords in a config file → thief has all access
- **Better**: Encrypted config; thief must crack encryption
- **Best**: No secrets on the laptop at all. When the laptop tries to connect, the secret manager detects the unusual location (impossible travel) and denies access. Even the legitimate employee can't get secrets from an untrusted device on an untrusted network.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Centralized vs Decentralized Secret Store**: One vault for the entire organization (simpler governance, single audit point) vs per-application vaults (smaller blast radius, less coupling). Most organizations use a hybrid: centralized secret backend per environment (dev/staging/prod) with application-specific access policies.
- **Secret Injection Pattern**: How do workloads receive secrets? Init containers that fetch secrets before the application starts (Kubernetes Secrets CSI Driver), sidecars that proxy secret requests (Vault Agent), or SDK-level integration (application calls Vault API directly). Each has different security, complexity, and startup latency profiles.
- **Secret Rotation Strategy**: Automated vs manual. Automated rotation requires the secret manager to have the ability to update the target system (database, API gateway). Multi-step rotation (create new, test new, promote new, delete old) is more reliable than single-step replacement.
- **Zero Secret Goal**: The architectural North Star: work toward eliminating persistent credentials entirely. Use workload identity (IAM roles, SPIFFE, managed identities) for cloud-native workloads. Only legacy or external systems should need persistent secrets.

### Business Impact
- **Breach Prevention**: The single largest cause of cloud security incidents is leaked credentials. Proper secret management with rotation, revocation, and audit directly prevents the most common breach pattern.
- **Compliance**: SOC 2 CC6.1 requires logical access controls for credentials. PCI DSS Requirement 3.6 requires key management procedures including rotation. GDPR requires access controls for personal data. Secret management is front-and-center in every audit.
- **Operational Safety**: When an employee with access to production leaves, can you rotate all their credentials in minutes, not weeks? Without centralized secret management, offboarding is a multi-week scavenger hunt through config files, hardcoded values, and tribal knowledge.
- **Developer Productivity**: Developers waste time debugging "why doesn't my connection work?" when credentials change. Automated secret management with dynamic credentials eliminates this friction and enables self-service access.

## On-Premises Examples

### HashiCorp Vault (The Gold Standard)
```bash
# Start Vault dev server
vault server -dev

# Enable secrets engines
vault secrets enable -path=database database
vault secrets enable -path=kv kv-v2
vault secrets enable pki
vault secrets enable transit

# Configure database credentials
vault write database/config/postgres-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles="order-reader" \
  connection_url="postgresql://{{username}}:{{password}}@db.internal.com:5432/orders"

vault write database/roles/order-reader \
  db_name=postgres-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' \
    VALID UNTIL '{{expiration}}'; GRANT SELECT ON orders TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Application reads dynamic database credentials
vault read database/creds/order-reader

# Store static secrets (K/V v2 with versioning)
vault kv put secret/prod/database config='{"host":"db.internal.com","port":5432}'
vault kv get secret/prod/database

# PKI - issue TLS certificates
vault write pki/issue/example-dot-com \
  common_name=app.example.com \
  ttl=720h  # 30 days
```

### SOPS (Secrets Operations) - Git-Friendly Encrypted Secrets
```yaml
# .sops.yaml - use KMS to encrypt, store encrypted files in git
creation_rules:
  - path_regex: config/.*\.yaml$
    kms: arn:aws:kms:us-east-1:123456789:key/abc-123
  - path_regex: staging/.*\.yaml$
    kms: arn:aws:kms:us-east-1:123456789:key/def-456

# Encrypt a secrets file
sops -e config/production.yaml > config/production.enc.yaml

# Decrypt at deploy time
sops -d config/production.enc.yaml
```

### Sealed Secrets (Kubernetes)
```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# Encrypt a Kubernetes Secret into a SealedSecret (safe for git)
kubectl create secret generic db-credentials \
  --from-literal=password=SuperSecret123 \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > db-credentials-sealed.yaml

# Apply SealedSecret (controller decrypts and creates Kubernetes Secret)
kubectl apply -f db-credentials-sealed.yaml
```

```yaml
# db-credentials-sealed.yaml (safe to commit to git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
spec:
  encryptedData:
    password: AgBy3i4JDqZ8encryptedbase64stringhere...
```

## AWS Examples

### AWS Secrets Manager
```hcl
resource "aws_secretsmanager_secret" "database" {
  name        = "prod/database/credentials"
  description = "Production database credentials"
  
  # Enable automatic rotation
  rotation_rules {
    automatically_after_days = 30
  }
  
  # Lambda function that handles rotation
  rotation_lambda_arn = aws_lambda_function.db_rotator.arn
}

resource "aws_secretsmanager_secret_version" "database" {
  secret_id = aws_secretsmanager_secret.database.id
  secret_string = jsonencode({
    username = "app_user"
    password = random_password.db_password.result
    host     = aws_db_instance.main.address
    port     = 5432
    dbname   = "orders"
  })
}
```

```python
import boto3
import json
from botocore.exceptions import ClientError

def get_db_credentials(secret_name, region_name="us-east-1"):
    session = boto3.session.Session()
    client = session.client(service_name='secretsmanager', region_name=region_name)
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return json.loads(response['SecretString'])
    except ClientError as e:
        raise e

# Application code
creds = get_db_credentials('prod/database/credentials')
connection = psycopg2.connect(
    host=creds['host'],
    user=creds['username'],
    password=creds['password'],
    dbname=creds['dbname']
)
```

### AWS Systems Manager Parameter Store
```hcl
# For lower-cost, simpler secrets (not auto-rotated)
resource "aws_ssm_parameter" "api_key" {
  name  = "/prod/api/external-service-key"
  type  = "SecureString"
  value = var.external_api_key
  
  # KMS key for encryption
  key_id = aws_kms_key.ssm.id
}
```

```python
import boto3

ssm = boto3.client('ssm')

# Retrieve single parameter
response = ssm.get_parameter(
    Name='/prod/api/external-service-key',
    WithDecryption=True  # Decrypt SecureString
)
api_key = response['Parameter']['Value']

# Batch retrieve
response = ssm.get_parameters(
    Names=['/prod/db/host', '/prod/db/password'],
    WithDecryption=True
)
```

### AWS KMS + Envelope Encryption
```python
import boto3
from cryptography.fernet import Fernet
import base64

kms = boto3.client('kms')

def encrypt_with_kms(plaintext, kms_key_id):
    # Generate data key (plaintext + encrypted versions)
    response = kms.generate_data_key(
        KeyId=kms_key_id,
        KeySpec='AES_256'
    )
    plaintext_key = response['Plaintext']
    encrypted_key = response['CiphertextBlob']
    
    # Encrypt data with plaintext key
    fernet_key = base64.b64encode(plaintext_key)
    fernet = Fernet(fernet_key)
    ciphertext = fernet.encrypt(plaintext.encode())
    
    return {
        'ciphertext': base64.b64encode(ciphertext).decode(),
        'encrypted_key': base64.b64encode(encrypted_key).decode()
    }

def decrypt_with_kms(encrypted_data, encrypted_key):
    # Decrypt the data key
    response = kms.decrypt(
        CiphertextBlob=base64.b64decode(encrypted_key)
    )
    plaintext_key = response['Plaintext']
    
    # Decrypt data with plaintext key
    fernet_key = base64.b64encode(plaintext_key)
    fernet = Fernet(fernet_key)
    plaintext = fernet.decrypt(base64.b64decode(encrypted_data))
    
    return plaintext.decode()
```

### ECS + Secrets Manager Injection
```json
{
  "family": "order-service",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecs-task-role",
  "containerDefinitions": [{
    "name": "order-service",
    "image": "order-service:latest",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/db-password-abc123"
      }
    ],
    "environment": [
      {
        "name": "DB_HOST",
        "value": "orders-db.internal.com"
      }
    ]
  }]
}
```

## GCP Examples

### Secret Manager
```bash
# Create secret
echo -n "SuperSecretPassword123!" | gcloud secrets create db-password \
  --data-file=- \
  --replication-policy=automatic

# Create secret version
gcloud secrets versions add db-password \
  --data-file=new-password.txt

# Grant access (IAM-based, not secret-level)
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:app-service@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Enable automatic rotation
gcloud secrets update db-password \
  --add-rotation=rotation-schedule.yaml
```

```python
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()

# Access secret
name = "projects/my-project/secrets/db-password/versions/latest"
response = client.access_secret_version(request={"name": name})
password = response.payload.data.decode("UTF-8")
```

### Workload Identity (No Secret Alternative)
```bash
# Map Kubernetes service account to GCP service account
gcloud iam service-accounts add-iam-policy-binding \
  app-service@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project.svc.id.goog[production/order-service]"
# Now the Kubernetes pod automatically gets GCP credentials
# NO secrets to manage in Kubernetes secrets or env vars
```

### Berglas (Secret Management for GCP)
```bash
# Store secret encrypted in Cloud Storage, decrypt via KMS
berglas create my-bucket/db-password "SuperSecret123" \
  --key projects/my-project/locations/global/keyRings/my-keyring/cryptoKeys/berglas-key

# Inject into Cloud Run / Cloud Function
berglas exec -- gcloud run deploy order-service \
  --image gcr.io/my-project/order-service \
  --set-secrets=DB_PASSWORD=berglas://my-bucket/db-password
```

## Azure Examples

### Azure Key Vault
```bash
az keyvault create \
  --name myAppKeyVault \
  --resource-group myResourceGroup \
  --location eastus \
  --enable-soft-delete true \
  --enable-purge-protection true

# Store secret
az keyvault secret set \
  --vault-name myAppKeyVault \
  --name db-password \
  --value "SuperSecretPassword123!"

# Set auto-rotation policy (90 days)
az keyvault secret rotation-policy update \
  --vault-name myAppKeyVault \
  --name db-password \
  --value rotation-policy.json

# Grant access via managed identity
az keyvault set-policy \
  --name myAppKeyVault \
  --object-id MANAGED_IDENTITY_OBJECT_ID \
  --secret-permissions get list
```

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

# Uses managed identity (no credentials in code)
credential = DefaultAzureCredential()
client = SecretClient(
    vault_url="https://myAppKeyVault.vault.azure.net",
    credential=credential
)

# Retrieve secret
secret = client.get_secret("db-password")
password = secret.value
```

### Azure Managed Identity (No Secrets)
```python
# App Service / Container App / VM automatically has a managed identity
# DefaultAzureCredential tries: Managed Identity → Environment → Azure CLI → VS Code
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

# ZERO credentials in code or config
credential = DefaultAzureCredential()
blob_service = BlobServiceClient(
    account_url="https://mystorage.blob.core.windows.net",
    credential=credential
)
```

### Azure App Configuration + Key Vault References
```json
{
  "Database": {
    "ConnectionString": "@Microsoft.KeyVault(SecretUri=https://myAppKeyVault.vault.azure.net/secrets/connection-string/)"
  }
}
```
The app configuration value is a reference to Key Vault, not the actual secret. App resolves it at runtime.

## Summary Decision Matrix

| Secret Management Approach | Security Level | Operational Complexity | Best For |
|---------------------------|---------------|------------------------|----------|
| Workload Identity (no secrets) | Highest | Low | Cloud-native workloads (EC2, ECS, Lambda, GKE, AKS) |
| Dynamic Secrets (ephemeral) | Very High | Medium (requires vault) | Database credentials, API keys for services |
| Encrypted Secret Store (Vault/KMS) | High | Medium | All static secrets with rotation |
| Encrypted in Git (SOPS, Sealed Secrets) | Medium-High | Low | GitOps workflows, declarative config |
| Environment Variables | Low-Medium | Very Low | Development only; not for production |
| Hardcoded in Code | None | "Easy" (disastrous) | NEVER |

Secret management is a journey. Start by eliminating hardcoded secrets—add pre-commit hooks and CI scanning today. Next, move static secrets to a managed secret store (Secrets Manager, Secret Manager, Key Vault). Then add rotation for critical credentials. The North Star is workload identity—where workloads authenticate via their inherent identity (IAM role, SPIFFE, managed identity) and never touch a persistent credential. Every step reduces the blast radius of a credential leak and moves your architecture toward security maturity.
