# Authentication & Authorization

## Introduction
Authentication and Authorization are the two foundational pillars of application security. While often lumped together as "Auth," they serve fundamentally different purposes. Authentication answers "Who are you?" and Authorization answers "What are you allowed to do?" The distinction became critical with the rise of web applications, APIs, and federated identity, where verifying identity and controlling access are handled by separate systems.

The modern landscape is dominated by token-based authentication (JWT, OAuth 2.0, OpenID Connect), moving away from server-side sessions to stateless, scalable identity architectures. Understanding the full spectrum—from basic password hashing to federation protocols like SAML—is essential for every solution architect designing multi-service, multi-client systems.

## Definition

**Authentication (AuthN)** is the process of verifying the identity of a user, service, or device. It confirms that the entity is who it claims to be, typically through one or more factors:
- **Something you know**: Password, PIN, security question
- **Something you have**: Hardware token, phone (SMS OTP), smart card, hardware security key (FIDO2/WebAuthn)
- **Something you are**: Fingerprint, face, retina scan, voice pattern (biometrics)

**Authorization (AuthZ)** is the process of determining what an authenticated entity is permitted to do. It enforces access control policies:
- **RBAC (Role-Based Access Control)**: Permissions grouped into roles, users assigned to roles
- **ABAC (Attribute-Based Access Control)**: Policies evaluate user attributes, resource attributes, and environment context
- **PBAC (Policy-Based Access Control)**: Formal policies evaluated by a policy engine
- **ReBAC (Relationship-Based Access Control)**: Access based on relationships between entities (e.g., Google Zanzibar)

## Concept Explanation

### Authentication Protocols and Standards

#### OAuth 2.0 (Authorization Framework)
OAuth 2.0 is NOT an authentication protocol—it's a delegated authorization framework. It allows a user to grant a third-party application limited access to their resources without sharing their password.

**Core Roles**:
- **Resource Owner**: The user who owns the data
- **Client**: The application requesting access
- **Authorization Server**: Issues tokens after user consent
- **Resource Server**: Hosts the protected resources

**Grant Types (Flows)**:

```
1. Authorization Code Flow (with PKCE) - Web and mobile apps
   Client → Auth Server: /authorize?response_type=code&code_challenge=XYZ
   User authenticates and consents
   Auth Server → Client: authorization code
   Client → Auth Server: /token with code + code_verifier
   Auth Server → Client: access_token + refresh_token

2. Client Credentials Flow - Machine-to-machine
   Client → Auth Server: /token with client_id + client_secret
   Auth Server → Client: access_token

3. Device Code Flow - TV, IoT, CLI
   Client → Auth Server: /device/code
   Auth Server → Client: device_code + user_code + verification_uri
   User visits URL, enters code, authenticates
   Client polls Auth Server until token is ready

4. Refresh Token Flow - Renew expired access tokens
   Client → Auth Server: /token with refresh_token
   Auth Server → Client: new access_token (+ optional new refresh_token)
```

#### OpenID Connect (OIDC) - Authentication Layer on OAuth 2.0
OIDC adds identity verification on top of OAuth 2.0. It introduces an **ID Token** (JWT) alongside the access token:

```json
{
  "iss": "https://auth.example.com",       // Issuer
  "sub": "user-789",                       // Subject (user identifier)
  "aud": "my-app-client-id",               // Audience (intended client)
  "exp": 1700000000,                       // Expiration
  "iat": 1699996400,                       // Issued at
  "email": "alice@example.com",
  "email_verified": true,
  "name": "Alice Johnson"
}
```

The ID Token proves the user authenticated and provides their identity claims. The access token authorizes API access.

#### SAML 2.0 (Security Assertion Markup Language)
XML-based federation protocol, dominant in enterprise SSO:

```xml
<saml:Assertion>
  <saml:Issuer>https://idp.company.com</saml:Issuer>
  <saml:Subject>
    <saml:NameID>alice@company.com</saml:NameID>
  </saml:Subject>
  <saml:AttributeStatement>
    <saml:Attribute Name="email">alice@company.com</saml:Attribute>
    <saml:Attribute Name="department">Engineering</saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

SAML is still widely used in enterprise environments (Azure AD, Okta, PingFederate) for SSO across corporate applications.

#### JWT (JSON Web Token)
Self-contained token format used by OAuth 2.0 and OIDC:

```
Structure: header.payload.signature
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI3ODkifQ.SIG

Header:  {"alg": "RS256", "typ": "JWT"}
Payload: {"sub": "789", "role": "admin", "exp": 1700000000}
Signature: RSA_SHA256(base64Url(header) + "." + base64Url(payload), private_key)
```

```python
import jwt
import datetime

# Create JWT
payload = {
    'sub': 'user-789',
    'role': 'admin',
    'exp': datetime.datetime.utcnow() + datetime.timedelta(hours=1),
    'iat': datetime.datetime.utcnow()
}
token = jwt.encode(payload, PRIVATE_KEY, algorithm='RS256')

# Verify JWT
decoded = jwt.decode(token, PUBLIC_KEY, algorithms=['RS256'], audience='my-api')
```

### Password Storage (Critical for AuthN)

Never store passwords in plaintext. Use adaptive hashing:

```python
import hashlib
import os

# BAD: Plain text or simple hash
# password_hash = md5(password)  # NEVER DO THIS

# GOOD: bcrypt with salt
import bcrypt
password = b"user_password"
salt = bcrypt.gensalt(rounds=12)  # work factor
hashed = bcrypt.hashpw(password, salt)

# Verify
bcrypt.checkpw(password, hashed)

# BETTER: Argon2id (winner of Password Hashing Competition)
from argon2 import PasswordHasher
ph = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)
hashed = ph.hash("user_password")
ph.verify(hashed, "user_password")
```

### Authorization Models Deep Dive

#### RBAC Example
```python
# Role definitions
roles = {
    'admin': ['read:all', 'write:all', 'delete:all', 'manage:users'],
    'editor': ['read:all', 'write:own', 'delete:own'],
    'viewer': ['read:public']
}

def check_permission(user_role, required_permission):
    return required_permission in roles.get(user_role, [])

# Usage
if not check_permission(current_user.role, 'write:all'):
    raise HTTPException(status_code=403, detail="Insufficient permissions")
```

#### ABAC Example (AWS IAM-style)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/classification": "public"
        },
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2024-12-31T23:59:59Z"
        }
      }
    }
  ]
}
```

### Multi-Factor Authentication (MFA)

Layered authentication security:

```python
# TOTP (Time-based One-Time Password) verification
import pyotp

# Setup
secret = pyotp.random_base32()
totp = pyotp.TOTP(secret)
uri = totp.provisioning_uri("alice@example.com", issuer_name="MyApp")
# User scans QR code with Google Authenticator

# Verification
token = request.form['totp_code']
if totp.verify(token):
    # Second factor verified
    issue_session()
```

### Session Management

```python
# Stateless JWT session
import jwt
from datetime import datetime, timedelta

def create_session(user):
    return jwt.encode({
        'sub': user.id,
        'role': user.role,
        'exp': datetime.utcnow() + timedelta(hours=1),
        'jti': str(uuid.uuid4())  # unique token ID for revocation
    }, SECRET_KEY, algorithm='HS256')

# Token revocation via blocklist
def revoke_token(jti):
    redis.setex(f"revoked:{jti}", timedelta(hours=1), "1")

def verify_token(token):
    decoded = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
    if redis.exists(f"revoked:{decoded['jti']}"):
        raise InvalidTokenError("Token revoked")
    return decoded
```

## Layman's Explanation

### Authentication: The Bouncer Checking Your ID
You arrive at a private club (the application). The bouncer (authentication) asks for your ID. You show your driver's license (password), and they check it's real (verification). For high-security areas, they also ask for a second piece of ID (MFA)—maybe your membership card (TOTP code from phone).

### Authorization: The Wristband System
Once inside, you get a wristband (access token). The wristband color indicates your access level (role). Blue wristband = general area. Red wristband = VIP lounge + general. Black wristband = everywhere including backstage. When you try to enter the VIP lounge, the staff (authorization middleware) checks your wristband color. No wristband, no entry. Wrong color, denied.

### OAuth: The Valet Key
You have a fancy car (your data). You don't want to give the valet (third-party app) your actual car key (password)—it has access to your trunk, glovebox, everything. Instead, you give them a valet key (access token) that only starts the car and opens the door, nothing else. The valet key expires after you leave the restaurant.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Auth Protocol Selection**: OAuth 2.0 + OIDC for customer-facing apps and APIs. SAML for enterprise SSO integration with legacy systems. WebAuthn/FIDO2 for passwordless, phishing-resistant authentication. The choice impacts every client SDK and integration.
- **Token Strategy**: Access token lifetime (5 minutes to 1 hour), refresh token rotation, token revocation architecture. Short-lived tokens limit damage from leaks. Refresh token rotation detects stolen refresh tokens.
- **Centralized vs Decentralized Auth**: A centralized identity provider (Auth0, Okta, Cognito, Azure AD) simplifies management but creates a single point of failure and potential latency bottleneck. Decentralized (per-service auth) provides resilience at the cost of consistency and user experience.
- **Session Architecture**: Stateful (server-side sessions with Redis/DB) vs stateless (JWT). Stateful enables instant revocation but requires shared session storage and doesn't scale as easily across regions. Stateless scales better but requires blocklist/revocation infrastructure.

### Business Impact
- **Breach Prevention**: 81% of hacking-related breaches involve compromised passwords (Verizon DBIR). MFA blocks 99.9% of automated attacks (Microsoft). Strong authentication is the single most effective security control.
- **User Experience**: Passwordless authentication (WebAuthn, passkeys, magic links) increases conversion rates by reducing login friction. Modern consumers expect social login (Google, Apple, Facebook) as an option.
- **Compliance**: SOC 2, PCI DSS, HIPAA, and GDPR all mandate specific authentication controls—MFA, password policies, access review, and session timeout requirements. Non-compliance means fines, lost certifications, and lost business.
- **B2B Integration**: Enterprise customers require SAML/OIDC SSO integration. Without it, you cannot sell to Fortune 500 companies. This is a hard requirement for B2B SaaS.

## On-Premises Examples

### Keycloak (Open Source Identity Provider)
```bash
# Run Keycloak
docker run -p 8080:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:24.0

# Configure realm, clients, users via Admin Console at http://localhost:8080
```

```javascript
// OIDC client configuration
const keycloak = new Keycloak({
    url: 'https://auth.internal.com',
    realm: 'my-realm',
    clientId: 'my-app'
});

await keycloak.init({ onLoad: 'login-required' });

// Use access token for API calls
fetch('/api/orders', {
    headers: { 'Authorization': `Bearer ${keycloak.token}` }
});
```

### LDAP + FreeRADIUS for Enterprise Auth
```bash
# OpenLDAP for directory services
slapd -h ldap:/// ldaps:///

# LDAP user authentication
ldapsearch -x -H ldap://ldap.internal.com \
  -D "uid=alice,ou=users,dc=company,dc=com" \
  -W -b "dc=company,dc=com"

# FreeRADIUS for network-level authentication (VPN, WiFi)
radiusd -X
```

### Django Authentication Framework
```python
from django.contrib.auth import authenticate, login
from django.contrib.auth.decorators import login_required, permission_required

# Authentication
def login_view(request):
    user = authenticate(request, username='alice', password='secure_password')
    if user is not None:
        login(request, user)

# Authorization with decorators
@login_required
@permission_required('orders.view_order', raise_exception=True)
def view_order(request, order_id):
    ...

# Custom permissions
from django.contrib.auth.models import Permission
view_order_perm = Permission.objects.get(codename='view_order')
user.user_permissions.add(view_order_perm)
```

## AWS Examples

### Amazon Cognito
Managed identity provider with user pools and federated identities:

```hcl
resource "aws_cognito_user_pool" "main" {
  name = "app-users"

  password_policy {
    minimum_length                   = 12
    require_lowercase                = true
    require_numbers                  = true
    require_symbols                  = true
    require_uppercase                = true
    temporary_password_validity_days = 1
  }

  mfa_configuration = "ON"

  software_token_mfa_configuration {
    enabled = true
  }

  schema {
    name     = "email"
    required = true
    attribute_data_type = "String"
    mutable  = true
  }
}

resource "aws_cognito_user_pool_client" "app" {
  name         = "app-client"
  user_pool_id = aws_cognito_user_pool.main.id

  generate_secret       = true
  refresh_token_validity = 30  # days
  
  explicit_auth_flows = [
    "ALLOW_REFRESH_TOKEN_AUTH",
    "ALLOW_USER_SRP_AUTH"
  ]
  
  token_validity_units {
    access_token  = "hours"
    id_token      = "hours"
    refresh_token = "days"
  }
  access_token_validity  = 1
  id_token_validity      = 1
}

# Identity pool for federated access to AWS services
resource "aws_cognito_identity_pool" "main" {
  identity_pool_name               = "app-identity-pool"
  allow_unauthenticated_identities = false

  cognito_identity_providers {
    client_id               = aws_cognito_user_pool_client.app.id
    provider_name           = "cognito-idp.us-east-1.amazonaws.com/${aws_cognito_user_pool.main.id}"
  }
}
```

```python
import boto3

# Cognito authentication
cognito = boto3.client('cognito-idp')

response = cognito.initiate_auth(
    ClientId='app-client-id',
    AuthFlow='USER_PASSWORD_AUTH',
    AuthParameters={
        'USERNAME': 'alice@example.com',
        'PASSWORD': 'SecureP@ss123!'
    }
)

# Response includes ID token, access token, refresh token
id_token = response['AuthenticationResult']['IdToken']
access_token = response['AuthenticationResult']['AccessToken']
refresh_token = response['AuthenticationResult']['RefreshToken']

# Verify JWT locally without calling Cognito
# Use jwks_client to fetch and cache JWKS from Cognito
```

### AWS IAM (Authorization)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/Orders",
      "Condition": {
        "ForAnyValue:StringEquals": {
          "dynamodb:LeadingKeys": "${cognito-identity.amazonaws.com:sub}"
        }
      }
    }
  ]
}
```

### AWS Verified Access (Zero Trust)
```hcl
resource "aws_verifiedaccess_instance" "main" {
  description = "Corporate zero trust access"
}

resource "aws_verifiedaccess_trust_provider" "cognito" {
  description       = "Identity verification"
  policy_reference_name = "cognito"
  trust_provider_type   = "user"

  device_options {
    tenant_id = aws_cognito_user_pool.main.id
  }
}
```

## GCP Examples

### Firebase Authentication (Google's Identity Platform)
```javascript
import { getAuth, signInWithEmailAndPassword, onAuthStateChanged } from "firebase/auth";

const auth = getAuth();

// Email/password authentication
signInWithEmailAndPassword(auth, 'alice@example.com', 'password')
  .then((userCredential) => {
    const user = userCredential.user;
    const idToken = user.getIdToken(); // JWT for backend
  });

// Auth state observer
onAuthStateChanged(auth, (user) => {
  if (user) {
    // User signed in
  }
});

// Social login: Google, Facebook, Apple, Twitter, GitHub
import { GoogleAuthProvider, signInWithPopup } from "firebase/auth";
signInWithPopup(auth, new GoogleAuthProvider());
```

### Identity Platform (Enterprise Identity)
```bash
gcloud identity platform configs update \
  --project=my-project \
  --mfa-enabled \
  --sms-region-config region=us-central1 \
  --blocked-functions email-enumeration-disable

# Multi-tenancy for B2B
gcloud identity platform tenants create my-enterprise-tenant \
  --allow-password-signup \
  --enable-email-link-signin
```

### Cloud IAM (Authorization)
```python
from google.cloud import resourcemanager

client = resourcemanager.ProjectsClient()

# Grant role
policy = client.get_iam_policy(request={"resource": "projects/my-project"})
policy.bindings.append({
    "role": "roles/storage.objectViewer",
    "members": ["user:alice@example.com"],
    "condition": {
        "title": "Business hours only",
        "expression": "request.time.getHours('WET') >= 9 && request.time.getHours('WET') <= 17"
    }
})
client.set_iam_policy(request={"resource": "projects/my-project", "policy": policy})
```

### Identity-Aware Proxy (IAP)
```bash
gcloud iap web enable --resource-type=backend-services \
  --oauth2-client-id=CLIENT_ID \
  --oauth2-client-secret=CLIENT_SECRET \
  backend-service-app

# IAP enforces authentication at the edge
# Users must authenticate via Google before reaching the application
```

## Azure Examples

### Azure AD / Entra ID
```bash
az ad app create --display-name "My Application"

az ad sp create --id "APPLICATION_ID"

# OAuth 2.0 + OIDC configuration
az ad app update --id "APPLICATION_ID" \
  --reply-urls "https://myapp.com/auth/callback" \
  --oauth2-allow-implicit-flow false \
  --required-resource-accesses @manifest.json
```

```python
from msal import ConfidentialClientApplication

# Microsoft Authentication Library (MSAL)
app = ConfidentialClientApplication(
    client_id=CLIENT_ID,
    client_credential=CLIENT_SECRET,
    authority="https://login.microsoftonline.com/TENANT_ID"
)

# Authorization code flow
result = app.acquire_token_by_authorization_code(
    code=request.args['code'],
    scopes=["https://graph.microsoft.com/.default"],
    redirect_uri="https://myapp.com/auth/callback"
)

access_token = result['access_token']
id_token = result['id_token']
```

### Azure RBAC
```bash
az role assignment create \
  --assignee alice@company.com \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/SUBSCRIPTION_ID/resourceGroups/myResourceGroup"

# Custom role definition
az role definition create --role-definition '{
  "Name": "Order Processor",
  "Description": "Can read orders and update status",
  "Actions": ["Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read"],
  "DataActions": ["Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read"],
  "AssignableScopes": ["/subscriptions/SUBSCRIPTION_ID"]
}'
```

### Azure AD B2C (Customer Identity)
```xml
<!-- Custom policy for sign-up/sign-in with MFA -->
<UserJourney Id="SignUpOrSignIn">
  <OrchestrationSteps>
    <OrchestrationStep Order="1" Type="CombinedSignInAndSignUp" />
    <OrchestrationStep Order="2" Type="ClaimsExchange">
      <ClaimsExchanges>
        <ClaimsExchange Id="MFAExchange" 
          TechnicalProfileReferenceId="PhoneFactor-InputOrVerify" />
      </ClaimsExchanges>
    </OrchestrationStep>
  </OrchestrationSteps>
</UserJourney>
```

## Summary Decision Matrix

| Requirement | Recommended Solution | Protocol |
|------------|---------------------|----------|
| Customer-facing app auth | Cognito, Firebase Auth, Azure AD B2C | OAuth 2.0 + OIDC |
| Enterprise SSO | Azure AD, Okta, Keycloak | SAML 2.0 + OIDC |
| API-to-API (machine) | OAuth Client Credentials, mTLS | OAuth 2.0, x509 |
| Passwordless / Phishing-resistant | WebAuthn/FIDO2, Passkeys | FIDO2/CTAP |
| Fine-grained cloud authorization | AWS IAM, GCP IAM, Azure RBAC | JSON policies, ABAC |
| Mobile app auth | OAuth 2.0 with PKCE | OIDC with PKCE |
| IoT/CLI device auth | Device Code Flow | OAuth 2.0 Device Grant |

Authentication and Authorization are not features to bolt on at the end—they are architectural decisions that shape the entire system design. The right protocol, token strategy, and permission model make security transparent to users while protecting resources effectively. A solution architect must design the identity layer before designing the application layer.
