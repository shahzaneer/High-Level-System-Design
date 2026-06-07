# Encryption (At-Rest & In-Transit)

## Introduction
Encryption is the mathematical backbone of digital security. It transforms readable data (plaintext) into unreadable ciphertext that can only be reversed with the correct key. In system architecture, encryption is deployed in two critical contexts: **at-rest** (protecting stored data from unauthorized access if storage media is compromised) and **in-transit** (protecting data as it moves across networks from eavesdropping and tampering).

The distinction matters because different threats, performance characteristics, key management strategies, and compliance requirements apply to each. A system with robust encryption in-transit but unencrypted databases fails compliance audits just as surely as one with encrypted storage but plaintext HTTP communication.

## Definition

**Encryption at-Rest** protects data stored on persistent media (databases, file systems, object storage, backups, logs) by encrypting it with cryptographic keys. When an attacker steals a hard drive or gains access to storage, the data is unreadable without the encryption keys.

**Encryption in-Transit** protects data moving across a network between clients and servers, between microservices, or between regions. It prevents eavesdropping (reading data), tampering (modifying data), and impersonation (man-in-the-middle attacks). TLS (Transport Layer Security) is the standard protocol.

**End-to-End Encryption (E2EE)**: Data is encrypted on the sender's device and only decrypted on the recipient's device. No intermediary (server, cloud provider) can read the plaintext. Used by Signal, WhatsApp, and iMessage.

## Concept Explanation

### Encryption Fundamentals

#### Symmetric Encryption
Same key for encryption and decryption. Fast and efficient for bulk data.

```
Plaintext ──[AES-256-GCM + Key]──→ Ciphertext
Ciphertext ──[AES-256-GCM + Key]──→ Plaintext
```

**Algorithms**:
- **AES-256-GCM**: Standard for data at rest. GCM mode provides authenticated encryption (confidentiality + integrity). Used by AWS KMS, GCP Cloud KMS.
- **ChaCha20-Poly1305**: Faster on mobile/IoT devices without AES hardware acceleration. Used by TLS 1.3, WireGuard.

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

key = AESGCM.generate_key(bit_length=256)  # 32 bytes
aesgcm = AESGCM(key)

# Encryption
nonce = os.urandom(12)  # 96-bit nonce, MUST be unique per encryption
plaintext = b"Sensitive data to encrypt"
ciphertext = aesgcm.encrypt(nonce, plaintext, associated_data=None)
# ciphertext includes authentication tag appended

# Decryption
decrypted = aesgcm.decrypt(nonce, ciphertext, associated_data=None)
assert decrypted == plaintext
```

#### Asymmetric Encryption (Public Key Cryptography)
Separate keys for encryption (public key) and decryption (private key). Slower, used for key exchange and digital signatures.

```
Sender:   Plaintext ──[Recipient's Public Key]──→ Ciphertext
Recipient: Ciphertext ──[Recipient's Private Key]──→ Plaintext
```

**Algorithms**:
- **RSA-2048/4096**: Traditional, widely supported. Key size must increase with computing power.
- **Elliptic Curve (ECDH, ECDSA)**: Smaller keys, faster, equivalent security with less computation. ECDHE for TLS key exchange, Ed25519 for signatures.
- **Kyber (Post-Quantum)**: NIST-standardized post-quantum key encapsulation. Preparing for quantum computers breaking RSA/EC.

```python
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes

# Key generation
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

# Encryption (small data only—RSA limited by key size)
ciphertext = public_key.encrypt(
    b"Symmetric key here",
    padding.OAEP(mgf=padding.MGF1(algorithm=hashes.SHA256()), algorithm=hashes.SHA256(), label=None)
)

# Decryption
plaintext = private_key.decrypt(ciphertext, padding.OAEP(...))
```

#### Hybrid Encryption (How TLS Actually Works)
Combines both: asymmetric for key exchange, symmetric for bulk data.

```
Client ──[TLS Handshake]──→ Server
  1. Client: "Hello, I support TLS 1.3 + these cipher suites"
  2. Server: "Let's use AES-256-GCM. Here's my certificate."
  3. Client verifies certificate against trusted CA
  4. ECDHE key exchange: both derive shared session key
  5. All subsequent data encrypted with AES-256-GCM session key
```

### TLS Deep Dive

```
TLS 1.3 Handshake (1-RTT):
Client → Server: ClientHello (supported ciphers, key share)
Server → Client: ServerHello (chosen cipher, key share) + Certificate + Finished
Client → Server: Finished
                  ─── Application Data (encrypted) ───→
```

**Certificate Validation Chain**:
```
Root CA (trusted, in browser/OS trust store)
  └── Intermediate CA (cross-signed by Root CA)
       └── Server Certificate (signed by Intermediate CA)
            Subject: api.example.com
            SAN: api.example.com, *.api.example.com
            Validity: 90 days (industry best practice, from CA/B Forum)
```

### Encryption at-Rest Patterns

#### Application-Level Encryption
Application encrypts data before sending to the database. The database never sees plaintext.

```python
from cryptography.fernet import Fernet

class ApplicationEncryption:
    def __init__(self, master_key):
        self.fernet = Fernet(master_key)
    
    def encrypt_and_store(self, user_id, sensitive_data):
        encrypted = self.fernet.encrypt(sensitive_data.encode())
        db.execute(
            "INSERT INTO user_data (user_id, encrypted_ssn) VALUES (?, ?)",
            (user_id, encrypted)
        )
    
    def retrieve_and_decrypt(self, user_id):
        row = db.execute(
            "SELECT encrypted_ssn FROM user_data WHERE user_id = ?",
            (user_id,)
        ).fetchone()
        return self.fernet.decrypt(row['encrypted_ssn']).decode()
```

**Pros**: Database breaches don't expose data. Database admins can't see plaintext.
**Cons**: Can't search/index encrypted columns. Key management complexity.

#### Transparent Data Encryption (TDE)
Database engine encrypts data at storage level, transparent to application. SQL Server, Oracle, MySQL, PostgreSQL support TDE.

```sql
-- SQL Server TDE
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongMasterKeyP@ss!';
CREATE CERTIFICATE TDECert WITH SUBJECT = 'TDE Certificate';
CREATE DATABASE ENCRYPTION KEY
  WITH ALGORITHM = AES_256
  ENCRYPTION BY SERVER CERTIFICATE TDECert;

ALTER DATABASE my_database SET ENCRYPTION ON;
```

#### Full Disk Encryption (FDE)
Encrypts entire storage volume. LUKS (Linux), BitLocker (Windows), FileVault (macOS).

```bash
# LUKS disk encryption
cryptsetup luksFormat /dev/sdb
cryptsetup luksOpen /dev/sdb encrypted_volume
mkfs.ext4 /dev/mapper/encrypted_volume
mount /dev/mapper/encrypted_volume /mnt/secure
```

#### Envelope Encryption
The pattern used by all cloud KMS services. Data is encrypted with a **Data Encryption Key (DEK)**. The DEK is encrypted with a **Key Encryption Key (KEK)** stored in a Hardware Security Module (HSM). The encrypted DEK is stored alongside the ciphertext.

```
Encrypt:
  1. Generate DEK (AES-256)
  2. Encrypt data with DEK → ciphertext
  3. Send DEK to KMS → receive encrypted DEK (wrapped by KEK)
  4. Store: ciphertext + encrypted DEK

Decrypt:
  1. Send encrypted DEK to KMS → receive decrypted DEK
  2. Decrypt ciphertext with DEK → plaintext
```

```python
import boto3

kms = boto3.client('kms')
KEY_ID = 'arn:aws:kms:us-east-1:123456789:key/abc-123'

# Generate and encrypt DEK
response = kms.generate_data_key(KeyId=KEY_ID, KeySpec='AES_256')
plaintext_dek = response['Plaintext']          # Use this, then discard
encrypted_dek = response['CiphertextBlob']     # Store this with data

# Encrypt data with DEK
aesgcm = AESGCM(plaintext_dek)
ciphertext = aesgcm.encrypt(nonce, data, None)

# Store: ciphertext + encrypted_dek + nonce
# ...

# Decrypt: first decrypt DEK, then data
response = kms.decrypt(CiphertextBlob=encrypted_dek)
dek = response['Plaintext']
aesgcm = AESGCM(dek)
plaintext = aesgcm.decrypt(nonce, ciphertext, None)
```

### Key Management

**Key Rotation**: Regularly replacing encryption keys limits data exposed if a key is compromised:

```python
# Envelope encryption with key versioning
class KeyRotatingEncryptor:
    def __init__(self, kms_client, key_id):
        self.kms = kms_client
        self.key_id = key_id
        self.current_key = self._get_current_dek()
        self.rotation_interval = timedelta(hours=24)
        self.last_rotation = datetime.utcnow()
    
    def encrypt(self, data):
        if datetime.utcnow() - self.last_rotation > self.rotation_interval:
            self.current_key = self._get_current_dek()
            self.last_rotation = datetime.utcnow()
        # Encrypt with current DEK, store with encrypted DEK + version
        pass
```

**Hardware Security Modules (HSMs)**: Tamper-resistant hardware that stores and manages keys. Private keys never leave the HSM. AWS CloudHSM, Azure Dedicated HSM, Google Cloud HSM.

### Certificate Management

```bash
# Let's Encrypt (ACME protocol) automated certificate issuance
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /etc/cloudflare.ini \
  -d example.com -d *.example.com

# Manual CSR (Certificate Signing Request)
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server.key -out server.csr \
  -subj "/C=US/ST=California/L=San Francisco/O=My Company/CN=api.example.com"

# Self-signed (development only)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -nodes -subj "/CN=localhost"
```

## Layman's Explanation

### Encryption At-Rest: The Locked Safe
Your house has a locked safe (encrypted database). Even if a burglar breaks in (server breach) and steals the safe (copies the database files), they can't open it without the combination (encryption key). You also store the combination in a different, ultra-secure vault (KMS/HSM). The burglar would need to steal both the safe AND break into the vault to get anything useful.

### Encryption In-Transit: The Armored Truck
When you move valuables (data) between buildings (servers), you don't just carry them in the open. You use an armored truck (TLS/HTTPS) with:
- Opaque walls: nobody can see what's inside (encryption)
- Tamper-proof locks: if someone opens the doors en route, you know (authentication/integrity)
- Verified driver badges: the truck belongs to who it claims (certificate validation)

### End-to-End Encryption: The Sealed Envelope
You write a letter (message) and seal it in an envelope (encrypt) before giving it to the postal service (internet). The postal service delivers the sealed envelope to the recipient, who opens it. The postal service never reads the contents. Even if someone at the sorting facility opens it, all they see is gibberish. Only the recipient has the letter opener (private key).

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Where to Terminate TLS**: At the CDN edge? At the load balancer? At the application server? End-to-end TLS (client → CDN → LB → application)? Each hop has trade-offs. PCI DSS requires end-to-end encryption for cardholder data.
- **Encryption Key Hierarchy**: DEKs, KEKs, Master Keys. How often are keys rotated? Who has access? What happens when an employee with key access leaves the company? Key management is harder than encryption itself.
- **TLS Version and Cipher Selection**: TLS 1.0 and 1.1 are deprecated (PCI DSS deadline was 2018). TLS 1.2 minimum, TLS 1.3 preferred. Weak ciphers (RC4, 3DES, CBC mode without encrypt-then-MAC) must be disabled. This requires configuration at every hop.
- **Certificate Lifecycle**: 90-day certificates (industry standard since Apple's 2020 announcement) mean automation is mandatory. Manual certificate renewal is operationally unsustainable. ACME protocol (Let's Encrypt) or cloud-managed certificates with auto-renewal is non-negotiable.

### Business Impact
- **Compliance**: Encryption at-rest and in-transit are explicitly required by PCI DSS (Requirement 3 & 4), HIPAA Security Rule (164.312(a)(2)(iv) & 164.312(e)(2)(ii)), SOC 2 CC6.1, GDPR Article 32. Non-compliance means audit failure, fines, and breach notification requirements.
- **Breach Liability**: Encrypted data that is stolen is typically not considered a "breach" under most data protection laws if the encryption keys were not also compromised. This can save millions in breach notification costs and reputational damage.
- **Customer Trust**: Public breaches destroy customer confidence. Encryption demonstrates security maturity and is increasingly a sales requirement for B2B SaaS products (security questionnaires, vendor risk assessments).
- **Data Sovereignty**: Organizations dealing with classified or export-controlled data may require specific encryption standards (FIPS 140-2/3 validated modules, CNSA suite algorithms).

## On-Premises Examples

### TLS Configuration (Nginx)
```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;

    # Strong TLS configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;  # Let client choose (TLS 1.3 best practice)
    
    ssl_certificate     /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1h;
    ssl_session_tickets off;  # Disable for forward secrecy
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    
    # HSTS (31536000 seconds = 1 year)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

### LUKS Full Disk Encryption
```bash
# Encrypt disk
cryptsetup luksFormat --type luks2 /dev/sdb \
  --cipher aes-xts-plain64 --key-size 512 --hash sha512

# Mount encrypted volume
cryptsetup luksOpen /dev/sdb encrypted_data
mkfs.ext4 /dev/mapper/encrypted_data
mount /dev/mapper/encrypted_data /data

# Auto-mount with key file (for servers)
dd if=/dev/urandom of=/root/keyfile bs=4096 count=1
chmod 0400 /root/keyfile
cryptsetup luksAddKey /dev/sdb /root/keyfile
```

### HashiCorp Vault (Key Management)
```bash
vault secrets enable transit

# Create encryption key with auto-rotation
vault write -f transit/keys/orders-key \
  type=aes256-gcm96 \
  auto_rotate_period=720h  # 30 days

# Encrypt data via Vault API
curl -X POST https://vault.internal.com/v1/transit/encrypt/orders-key \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -d '{"plaintext": "'$(echo -n "sensitive data" | base64)'"}'

# Decrypt
curl -X POST https://vault.internal.com/v1/transit/decrypt/orders-key \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -d '{"ciphertext": "vault:v1:..."}'
```

### Mutual TLS (mTLS) Between Services
```bash
# Generate CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt

# Generate service certificate signed by CA
openssl genrsa -out service.key 2048
openssl req -new -key service.key -out service.csr
openssl x509 -req -in service.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out service.crt -days 365
```

```nginx
# mTLS: require client certificate
server {
    listen 443 ssl;
    ssl_certificate     /etc/ssl/service.crt;
    ssl_certificate_key /etc/ssl/service.key;
    ssl_client_certificate /etc/ssl/ca.crt;
    ssl_verify_client on;
    ssl_verify_depth 2;
    
    location / {
        proxy_pass http://backend;
    }
}
```

## AWS Examples

### AWS Certificate Manager (ACM)
Managed TLS certificates with automatic renewal:

```hcl
resource "aws_acm_certificate" "app" {
  domain_name       = "api.example.com"
  validation_method = "DNS"

  subject_alternative_names = [
    "*.api.example.com",
    "example.com"
  ]

  options {
    certificate_transparency_logging_preference = "ENABLED"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# DNS validation (automatic renewal)
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.app.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  zone_id = aws_route53_zone.main.zone_id
  ttl     = 60
}

resource "aws_acm_certificate_validation" "app" {
  certificate_arn         = aws_acm_certificate.app.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}
```

### AWS KMS (Key Management Service)
```hcl
resource "aws_kms_key" "app" {
  description             = "Application encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true       # Automatic annual rotation
  rotation_period_in_days = 90
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow use by application"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.app_role.arn
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:GenerateDataKey*"
        ]
        Resource = "*"
      }
    ]
  })
}
```

```python
# Envelope encryption with KMS
import boto3
import json

kms = boto3.client('kms')

def encrypt_record(record, kms_key_id):
    dek_response = kms.generate_data_key(
        KeyId=kms_key_id,
        KeySpec='AES_256'
    )
    
    from cryptography.hazmat.primitives.ciphers.aead import AESGCM
    import os
    
    aesgcm = AESGCM(dek_response['Plaintext'])
    nonce = os.urandom(12)
    ciphertext = aesgcm.encrypt(nonce, json.dumps(record).encode(), None)
    
    return {
        'data': base64.b64encode(ciphertext).decode(),
        'nonce': base64.b64encode(nonce).decode(),
        'encrypted_key': base64.b64encode(dek_response['CiphertextBlob']).decode()
    }
```

### AWS CloudHSM
Dedicated hardware security module for FIPS 140-2 Level 3 compliance:

```hcl
resource "aws_cloudhsm_v2_cluster" "main" {
  hsm_type   = "hsm1.medium"
  subnet_ids = aws_subnet.private[*].id

  tags = {
    Name = "app-hsm-cluster"
  }
}

resource "aws_cloudhsm_v2_hsm" "hsm" {
  count        = 2  # minimum 2 for HA
  cluster_id   = aws_cloudhsm_v2_cluster.main.cluster_id
  subnet_id    = aws_subnet.private[count.index].id
}
```

### S3 Server-Side Encryption
```python
import boto3

s3 = boto3.client('s3')

# SSE-KMS: Encrypt with KMS customer managed key
s3.put_object(
    Bucket='my-bucket',
    Key='sensitive-data.json',
    Body=json.dumps(data),
    ServerSideEncryption='aws:kms',
    SSEKMSKeyId='arn:aws:kms:us-east-1:123456789:key/abc-123'
)

# SSE-S3: AES-256 managed by S3 (simplest)
s3.put_object(
    Bucket='my-bucket',
    Key='data.json',
    Body=json.dumps(data),
    ServerSideEncryption='AES256'
)

# SSE-C: Customer-provided key (you manage)
s3.put_object(
    Bucket='my-bucket',
    Key='data.json',
    Body=json.dumps(data),
    SSECustomerAlgorithm='AES256',
    SSECustomerKey='base64-encoded-256-bit-key'
)
```

## GCP Examples

### Cloud KMS
```bash
# Create key ring and key
gcloud kms keyrings create app-keyring --location=global

gcloud kms keys create app-encryption-key \
  --location=global \
  --keyring=app-keyring \
  --purpose=encryption \
  --rotation-period=90d \
  --next-rotation-time=2026-01-01T00:00:00Z \
  --protection-level=hsm  # FIPS 140-2 Level 3
```

```python
from google.cloud import kms

client = kms.KeyManagementServiceClient()
key_name = client.crypto_key_path(
    'my-project', 'global', 'app-keyring', 'app-encryption-key'
)

# Encrypt
response = client.encrypt(
    request={'name': key_name, 'plaintext': b'sensitive data'}
)
ciphertext = response.ciphertext

# Decrypt
response = client.decrypt(
    request={'name': key_name, 'ciphertext': ciphertext}
)
plaintext = response.plaintext
```

### Cloud HSM
```bash
gcloud kms keyrings create app-keyring --location=us-central1

gcloud kms keys create hsm-key \
  --location=us-central1 \
  --keyring=app-keyring \
  --purpose=encryption \
  --protection-level=hsm \
  --default-algorithm=google-symmetric-encryption
```

### Managed SSL Certificates
```bash
# Google-managed SSL certificate for load balancer
gcloud compute ssl-certificates create app-cert \
  --domains=api.example.com,*.api.example.com \
  --managed

# Self-managed certificate
gcloud compute ssl-certificates create app-cert \
  --certificate=fullchain.pem \
  --private-key=privkey.pem
```

### TLS Policy (Enforce Minimum TLS Version)
```bash
gcloud compute ssl-policies create modern-tls \
  --profile=MODERN \
  --min-tls-version=1.2

gcloud compute target-https-proxies update app-proxy \
  --ssl-policy=modern-tls
```

## Azure Examples

### Azure Key Vault
```bash
az keyvault create \
  --name myAppKeyVault \
  --resource-group myResourceGroup \
  --location eastus \
  --enable-soft-delete true \
  --enable-purge-protection true \
  --sku premium

# Create key with auto-rotation
az keyvault key create \
  --vault-name myAppKeyVault \
  --name app-encryption-key \
  --protection hsm \
  --ops encrypt decrypt \
  --kty RSA-HSM

# Enable key auto-rotation
az keyvault key rotation-policy update \
  --vault-name myAppKeyVault \
  --name app-encryption-key \
  --value rotation-policy.json
```

```python
from azure.keyvault.keys import KeyClient
from azure.keyvault.keys.crypto import CryptographyClient, EncryptionAlgorithm
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
key_client = KeyClient(vault_url="https://myAppKeyVault.vault.azure.net", credential=credential)
key = key_client.get_key("app-encryption-key")

crypto_client = CryptographyClient(key, credential)

# Encrypt
result = crypto_client.encrypt(
    EncryptionAlgorithm.rsa_oaep_256,
    b"Sensitive data"
)
ciphertext = result.ciphertext

# Decrypt
result = crypto_client.decrypt(
    EncryptionAlgorithm.rsa_oaep_256,
    ciphertext
)
```

### Azure Encryption at-Rest (Storage Service Encryption)
```bash
# Storage account with Microsoft-managed keys (default)
az storage account create \
  --name mystorageaccount \
  --resource-group myResourceGroup \
  --encryption-services blob file table queue

# Storage account with customer-managed keys
az storage account update \
  --name mystorageaccount \
  --resource-group myResourceGroup \
  --encryption-key-source Microsoft.KeyVault \
  --encryption-key-vault https://myAppKeyVault.vault.azure.net/ \
  --encryption-key-name app-encryption-key
```

### Azure SQL TDE
```sql
-- Transparent Data Encryption with customer-managed key
ALTER SERVER CONFIGURATION 
  SET TDE_MASTER_KEY_ENCRYPTION_TYPE = SERVICE_MANAGED;

-- With Azure Key Vault key
ALTER SERVER CONFIGURATION 
  SET TDE_MASTER_KEY_ENCRYPTION_TYPE = CUSTOMER_MANAGED;
```

### Azure Application Gateway (TLS Termination + End-to-End)
```bash
az network application-gateway ssl-cert create \
  --gateway-name appGateway \
  --resource-group myResourceGroup \
  --name app-cert \
  --cert-file fullchain.pem \
  --cert-password ""

# End-to-end TLS: re-encrypt at gateway
az network application-gateway http-settings update \
  --gateway-name appGateway \
  --resource-group myResourceGroup \
  --name backend-http-settings \
  --protocol Https \
  --port 443
```

## Summary Decision Matrix

| Encryption Context | Recommended Algorithm | Key Management | Performance Impact |
|-------------------|-----------------------|----------------|-------------------|
| Data at-rest (DB/Storage) | AES-256-GCM | KMS with envelope encryption | 3-5% overhead with hardware AES-NI |
| TLS in-transit | TLS 1.3, AES-256-GCM, ECDHE | ACM/Managed certificates | ~1ms added latency for handshake |
| File/disk encryption | AES-256-XTS (LUKS 2) | Local key file or TPM | Minimal with hardware acceleration |
| Passwords (hashing) | Argon2id | Salt + high work factor | Configurable (tune for 100ms+) |
| Digital signatures | Ed25519 or ECDSA P-256 | HSM-protected private key | Sub-millisecond |
| End-to-end messaging | X3DH + Double Ratchet | Per-device key pairs | Minimal |

Encryption is not a feature—it's infrastructure. The difference between a secure system and a breached one is often as simple as whether the database files are encrypted, whether TLS is enforced on every connection, and whether encryption keys are managed separately from encrypted data. A solution architect must treat encryption as a first-class architectural concern, designing key hierarchies, TLS termination points, and key rotation policies from day one.
