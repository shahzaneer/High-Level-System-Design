# Zero Trust Architecture

## Introduction
Zero Trust is the most significant paradigm shift in security architecture since the invention of the firewall. Coined by Forrester Research in 2010 and popularized by Google's BeyondCorp initiative (2014), Zero Trust rejects the traditional "castle-and-moat" security model that trusts everything inside the corporate network perimeter. Instead, it operates on a simple but radical principle: **never trust, always verify**.

The catalyst for Zero Trust adoption was the dissolution of the network perimeter. When employees work from coffee shops, applications run in public clouds, and third-party APIs integrate directly with backend services, there is no "inside" the network to trust. Every access request—from the CEO's laptop, from a microservice in Kubernetes, from an IoT sensor—must be authenticated, authorized, and encrypted, regardless of network location.

In 2021, the U.S. Executive Order on Improving the Nation's Cybersecurity mandated Zero Trust for federal agencies. In 2022, OMB Memorandum M-22-09 required agencies to meet specific Zero Trust goals by the end of FY 2024. This is now a compliance requirement, not just a best practice.

## Definition
**Zero Trust Architecture (ZTA)** is a security model based on the principle of maintaining strict access controls and not trusting any entity by default, even those already inside the network perimeter. Core tenets:

1. **Verify Explicitly**: Always authenticate and authorize based on all available data points—user identity, device health, location, service, data classification, and anomalies
2. **Use Least Privilege Access**: Grant just-in-time and just-enough access (JIT/JEA), risk-based adaptive policies, and data protection to secure both data and productivity
3. **Assume Breach**: Design with the assumption that the network is already compromised. Minimize blast radius with micro-segmentation, end-to-end encryption, and continuous monitoring

**Key Departure from Traditional Security**:
- Traditional: "Trust but verify" → once inside the VPN, you're trusted
- Zero Trust: "Never trust, always verify" → every request is treated as if it originates from an untrusted network

## Concept Explanation

### The Castle-and-Moat vs Zero Trust

```
TRADITIONAL (Castle-and-Moat):
┌─────────────────────────────────────┐
│         "INSIDE" (TRUSTED)          │
│  ┌──────┐ ┌──────┐ ┌────────┐      │
│  │ App  │ │  DB  │ │File Srv│      │
│  └──────┘ └──────┘ └────────┘      │
│                                     │
│     ┌────────────────────┐          │
│     │   VPN / Firewall   │  ←── Perimeter Security
│     └────────────────────┘          │
└─────────────────────────────────────┘
              ↑
    "OUTSIDE" (UNTRUSTED)
    Once inside → full access to everything


ZERO TRUST:
┌─────────────────────────────────────┐
│          NO "INSIDE" ZONE            │
│  ┌──────┐ ┌──────┐ ┌────────┐      │
│  │ App  │ │  DB  │ │File Srv│      │
│  │  ┌──┐│ │  ┌──┐│ │   ┌──┐ │      │
│  │  │PE││ │  │PE││ │   │PE│ │      │
│  │  └──┘│ │  └──┘│ │   └──┘ │      │
│  └──────┘ └──────┘ └────────┘      │
│                                     │
│  Every access → AuthN + AuthZ +     │
│  Device Health + Encryption         │
│  Every. Single. Time.               │
└─────────────────────────────────────┘
PE = Policy Enforcement Point
```

### Core Components of Zero Trust

#### Policy Enforcement Point (PEP)
The gate that every request must pass through. Intercepts requests, enforces policy, and allows or denies access. Examples: identity-aware proxy (Google IAP, Azure AD App Proxy), API gateway, service mesh sidecar (Envoy).

```python
# Conceptual PEP implementation
class PolicyEnforcementPoint:
    def handle_request(self, request):
        # 1. Authenticate (who is this?)
        identity = self.authenticate(request)
        if not identity:
            return self.deny("Authentication failed")
        
        # 2. Check device trust (is the device healthy?)
        device = self.assess_device(request)
        if not device.trusted:
            return self.deny("Device not compliant")
        
        # 3. Authorize (what can they do to this resource?)
        if not self.authorize(identity, device, request):
            return self.deny("Not authorized")
        
        # 4. Check risk signals (is this request anomalous?)
        risk = self.assess_risk(identity, device, request)
        if risk == RiskLevel.HIGH:
            return self.challenge(request)  # Require MFA
        if risk == RiskLevel.CRITICAL:
            return self.deny("Too risky")
        
        # 5. Grant access (just-in-time, limited scope)
        return self.allow(identity, request)
```

#### Policy Decision Point (PDP)
The brain that evaluates access requests against policies. May be a centralized service (e.g., OPA - Open Policy Agent) or distributed (sidecar proxies).

```rego
# Open Policy Agent (OPA) - Zero Trust policy
package app.authz

default allow = false

# Allow API access if:
allow {
    # Token is valid
    input.token.valid == true
    
    # User has required scope
    input.token.scope[_] == input.required_scope
    
    # Device is managed and compliant
    input.device.managed == true
    input.device.compliant == true
    
    # Request is from expected geo-location
    input.geo.country == "US"
    
    # Data classification allows this access level
    input.resource.classification == "internal"
}

# Allow only during business hours for sensitive operations
allow {
    input.operation == "sensitive_read"
    input.token.scope[_] == "read:all"
    time.now_ns() % 86400000000000 < 61200000000000  # Before 5 PM UTC
}
```

#### Policy Information Point (PIP)
Sources of truth that feed data into policy decisions: identity provider (Okta, Azure AD), device management (Intune, Jamf), threat intelligence feeds, data classification systems.

### Zero Trust Pillars

#### 1. Identity (Who)
- Multi-factor authentication for ALL users, every session
- Passwordless authentication (FIDO2/WebAuthn, passkeys) is the goal
- Device-bound credentials (TPM-stored certificates, hardware security keys)
- Dynamic risk assessment: impossible travel detection, unusual behavior

```
User logs in from San Francisco at 9:00 AM
Same user attempts login from Moscow at 9:05 AM
→ Risk score: CRITICAL → Block + alert
```

#### 2. Device (What)
- Device must be enrolled and managed (MDM: Intune, Jamf, Workspace ONE)
- Device health checks: OS patch level, disk encryption, firewall enabled, no known vulnerabilities
- Device trust is continuously evaluated, not just at initial access

```json
{
  "device_id": "laptop-abc123",
  "os_version": "macOS 14.2.1",
  "disk_encrypted": true,
  "firewall_enabled": true,
  "last_patch": "2024-01-05",
  "has_screen_lock": true,
  "jailbroken": false,
  "compliance_status": "compliant"
}
```

#### 3. Network (Where)
- Network location is NOT a trust factor (coffee shop == corporate office)
- All traffic encrypted (mTLS, WireGuard, IPSec)
- Micro-segmentation: service-to-service communication explicitly authorized
- Software-Defined Perimeter (SDP): resources are "dark" until authenticated

#### 4. Application/Workload (Which)
- Every workload has its own identity (SPIFFE/SPIRE, cloud IAM roles)
- mTLS between all services, regardless of network location
- Continuous authorization—token validity is checked on every request
- API gateways enforce per-endpoint policies

```yaml
# SPIFFE ID: uniquely identifies each workload
# spiffe://example.com/namespace/production/service/order-processor
# SPIRE issues short-lived X.509 certificates for mTLS
```

#### 5. Data (The Asset)
- Data classified by sensitivity (public, internal, confidential, restricted)
- Data loss prevention (DLP) monitors and blocks unauthorized exfiltration
- Encryption everywhere: at rest, in transit, and increasingly in use (confidential computing)
- Rights management: even if a file is downloaded, access can be revoked

### Implementing Zero Trust: Google BeyondCorp Model

```
┌────────────────────────────────────────────────────┐
│              ACCESS PROXY (PEP)                     │
│  ┌──────────────────────────────────────────────┐  │
│  │ 1. Is user authenticated? (SSO + MFA)        │  │
│  │ 2. Is device trusted? (inventory + certs)    │  │
│  │ 3. Is user authorized for this resource?     │  │
│  │ 4. Does the request pass risk assessment?    │  │
│  └──────────────────────────────────────────────┘  │
│                    │                               │
│        ┌───────────┴───────────┐                   │
│        │                       │                   │
│   [Allow]                  [Deny]                   │
│  Access granted         Access denied              │
│  (session-encrypted)   (logged + alerted)          │
└────────────────────────────────────────────────────┘
```

### Software-Defined Perimeter (SDP)

Resources are "dark"—not visible to anyone, not even by IP scan—until authenticated:

```
Traditional:   Attacker scans 10.0.0.0/16 → finds open ports → attacks
Zero Trust:    Attacker scans 10.0.0.0/16 → all ports appear closed
               Authenticated user requests access → gateway reveals resource
               Authenticated user gets 1-to-1 encrypted tunnel to resource
```

## Layman's Explanation

### The Hotel That Trusts No One
Traditional security is like a hotel that checks your ID at the front door (VPN login) and then lets you roam freely through all floors, open any door, access any room. Once you're "inside," nobody questions you. If someone steals a guest's key card, they have access to everything.

Zero Trust is like a hotel where:
- Every door (resource) has its own security guard who checks your ID, room key, and purpose of visit—even if you were just in the hallway
- Your room key changes every hour (short-lived credentials)
- The guard also checks your "health"—do you have a valid reservation? Is your key card reported stolen? (device trust)
- Even the CEO needs to show ID at every door (no exceptions)
- The cleaning staff can only access rooms they're assigned to today, and only during their shift (just-in-time access)
- If you suddenly appear in the Tokyo branch when you were in the New York branch 10 minutes ago, all your keys are immediately disabled (impossible travel detection)

### The VIP Concert Backstage Pass
At a music festival, having a ticket gets you into the grounds. A VIP pass gets you into the VIP area. A backstage pass gets you backstage. An "all access" pass gets you everywhere. But even with an all-access pass, security checks it EVERY TIME you go through a door. If the pass expired (token expiry), you need a new one. If you lent your pass to a friend, facial recognition (biometric verification) catches it.

Zero Trust applies this logic to every digital resource in an organization.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Proxy Architecture**: All user access routes through an identity-aware proxy. This is the single architectural decision that enables Zero Trust for user-facing applications. Without a proxy that sits between users and applications, you can't enforce device trust or continuous verification.
- **Workload Identity (SPIFFE/SPIRE)**: Every microservice, container, and function needs a cryptographic identity independent of network location. SPIFFE provides a universal identity framework. Cloud-native equivalents (AWS IAM roles, GCP service accounts, Azure Managed Identities) implement the same concept.
- **Policy-as-Code**: Access policies must be version-controlled, tested, and deployed like application code (OPA, Cedar, Kyverno). Manual policy management is the enemy of Zero Trust—policies change too frequently and need audit trails.
- **Micro-Segmentation**: Every service can only communicate with explicitly authorized services. Kubernetes NetworkPolicy, Istio AuthorizationPolicy, and cloud security groups must enforce this. The default posture: deny all, then add explicit allows.

### Business Impact
- **Breach Containment**: The SolarWinds attack (2020) spread across thousands of organizations because once attackers compromised the build system, they had broad network access. Zero Trust micro-segmentation would have limited lateral movement to a handful of services.
- **Remote Work Enablement**: Zero Trust makes VPNs obsolete. Employees get the same secure access whether they're in the office, at home, or at a coffee shop. This eliminates the VPN capacity crunch that plagued organizations during COVID-19.
- **Compliance**: U.S. Federal mandate (Executive Order 14028), CISA Zero Trust Maturity Model, NIST SP 800-207. For any organization doing business with the U.S. government, Zero Trust is a contractual requirement. Financial services (OCC, FDIC) and healthcare (HHS) are following suit.
- **M&A Integration**: When companies merge, network integration traditionally takes months (overlapping IP spaces, firewall rule reconciliation). Zero Trust networks connect via identity and policy, not IP convergence, dramatically accelerating integration.

### The Zero Trust Journey (Not a Destination)
Zero Trust is a maturity model, not a binary state:

```
Traditional → Identity-Aware Proxy → Device Trust → Micro-Segmentation → Full ZTA
    │                │                    │                │
    ├─ VPN           ├─ SSO + MFA        ├─ MDM enrolled  ├─ Kubernetes NetPol
    ├─ Perimeter FW ├─ BeyondCorp       ├─ Device health  ├─ Service mesh mTLS
    ├─ Implicit Trust├─ Context-aware     check            ├─ Data classification
                     │  access            └─ Conditional   └─ Automated threat
                     └─ OAuth/OIDC          access            response
```

## On-Premises Examples

### OpenZiti (Open Source Zero Trust Overlay)
```bash
# Deploy controller
ziti controller run controller.yaml

# Create edge router
ziti router run edge-router.yaml

# Create identity (service identity)
ziti edge create identity device order-service -o order-service.jwt
ziti edge enroll order-service.jwt

# Create service with zero trust policy
ziti edge create service order-api --role-attributes '#order-api'
ziti edge create service-policy order-dial-policy \
  --service-roles '#order-api' \
  --identity-roles '#order-service' \
  --semantic AnyOf

# Service is now "dark" - only enrolled identities can connect
# No open ports, no DNS visibility, no network scan detection
```

### Teleport (Modern Access Proxy)
```yaml
# teleport.yaml
auth_service:
  enabled: true
  authentication:
    type: local
    second_factor: otp
    
proxy_service:
  enabled: true
  public_addr: proxy.example.com:443
  https_keypairs:
    - key_file: /etc/teleport/privkey.pem
      cert_file: /etc/teleport/fullchain.pem

ssh_service:
  enabled: true
  
# Role with just-in-time access
kind: role
version: v5
metadata:
  name: developer
spec:
  allow:
    # Access to Kubernetes clusters
    kubernetes_groups: ["system:authenticated"]
    kubernetes_labels:
      environment: ["staging"]
    
    # Access to databases
    db_labels:
      environment: ["staging"]
    
    # Session recording for audit
    options:
      record_session:
        desktop: true
```

### OPA (Open Policy Agent) as PDP
```rego
# policy.rego
package system.authz

# Dynamic access policy
default allow = false

# API access policy
allow {
    input.method == "GET"
    input.path == ["api", "orders", order_id]
    input.user.department == data.order_owners[order_id]
}

# Data containing order-to-department mapping
order_owners = {
    "order-123": "retail",
    "order-456": "wholesale",
    "order-789": "retail"
}

# CLI: evaluate policy
# opa eval --input request.json --data policy.rego "data.system.authz.allow"
```

## AWS Examples

### AWS Verified Access (Zero Trust for Corporate Apps)
```hcl
resource "aws_verifiedaccess_instance" "main" {
  description = "Zero Trust access proxy"
}

resource "aws_verifiedaccess_trust_provider" "identity" {
  description          = "Identity verification"
  trust_provider_type  = "user"
  policy_reference_name = "identity"

  user_trust_provider_type = "iam-identity-center"

  device_options {
    tenant_id = data.aws_organizations_organization.main.id
  }
}

resource "aws_verifiedaccess_group" "developers" {
  verifiedaccess_instance_id = aws_verifiedaccess_instance.main.id
  
  policy_document = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { VerifiedAccess = { User = "*" } }
      Action = "VerifiedAccess:Allow"
      Condition = {
        StringEquals = {
          "verifiedaccess:userGroup": "developers"
          "verifiedaccess:deviceSecurity": "compliant"
        }
      }
    }]
  })
}

resource "aws_verifiedaccess_endpoint" "internal_app" {
  verifiedaccess_group_id = aws_verifiedaccess_group.developers.id
  
  application_domain    = "internal.example.com"
  endpoint_domain_prefix = "app"
  endpoint_type         = "load-balancer"
  load_balancer_options {
    load_balancer_arn = aws_lb.internal.arn
    port              = 443
    protocol          = "https"
  }
}
```

### AWS IAM Roles Anywhere (Workload Identity Outside AWS)
```hcl
resource "aws_rolesanywhere_profile" "onprem" {
  name      = "onprem-servers"
  role_arns = [aws_iam_role.onprem_access.arn]
}

resource "aws_rolesanywhere_trust_anchor" "onprem" {
  name = "onprem-ca"

  source {
    source_data {
      x509_certificate_data = file("ca-cert.pem")
    }
    source_type = "CERTIFICATE_BUNDLE"
  }
}
# On-premises servers use X.509 certificates to get temporary AWS credentials
# No long-term IAM access keys stored on-premises
```

### AWS Private CA + Certificates for mTLS
```hcl
resource "aws_acmpca_certificate_authority" "internal" {
  certificate_authority_configuration {
    key_algorithm     = "RSA_4096"
    signing_algorithm = "SHA512WITHRSA"

    subject {
      common_name = "internal.example.com"
    }
  }

  type = "ROOT"
}
# Issue short-lived certificates for every workload
# mTLS between all services, regardless of network location
```

## GCP Examples

### BeyondCorp Enterprise (Google's Zero Trust)
```bash
# Identity-Aware Proxy (IAP) - the PEP
gcloud iap web enable \
  --resource-type=backend-services \
  --oauth2-client-id=CLIENT_ID \
  --oauth2-client-secret=CLIENT_SECRET \
  backend-service-app

# IAP enforces:
# 1. User must authenticate with Google Identity
# 2. MFA required (organization policy)
# 3. Device must be trusted (Endpoint Verification)
# 4. Context-aware access policies applied
```

```bash
# Context-aware access policy
gcloud access-context-manager levels create moderate_risk \
  --title="Moderate Risk" \
  --basic-level-spec=@conditions.yaml
```

```yaml
# conditions.yaml
conditions:
  - devicePolicy:
      allowedEncryptionStatuses:
        - ENCRYPTED
      osConstraints:
        - osType: DESKTOP_MAC
          minimumVersion: "14.0"
        - osType: DESKTOP_WINDOWS
          minimumVersion: "10.0.19041"
    ipSubnetworks:
      - ipCidrRange: 10.0.0.0/8
```

### Workload Identity Federation
```bash
# Allow Kubernetes pods to impersonate GCP service accounts
gcloud iam service-accounts add-iam-policy-binding \
  gke-service-account@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project.svc.id.goog[production/order-service]"

# Allow GitHub Actions to access GCP without long-lived keys
gcloud iam workload-identity-pools create github-pool \
  --location="global" \
  --display-name="GitHub Actions Pool"

gcloud iam workload-identity-pools providers create-oidc github-provider \
  --workload-identity-pool="github-pool" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

## Azure Examples

### Azure AD Conditional Access (Zero Trust Policy Engine)
```json
{
  "displayName": "Require compliant device for all apps",
  "state": "enabled",
  "conditions": {
    "applications": {
      "includeApplications": ["All"]
    },
    "users": {
      "includeUsers": ["All"]
    },
    "platforms": {
      "includePlatforms": ["all"]
    }
  },
  "grantControls": {
    "operator": "AND",
    "builtInControls": [
      "mfa",
      "compliantDevice"
    ]
  },
  "sessionControls": {
    "signInFrequency": {
      "value": 4,
      "type": "hours",
      "isEnabled": true
    }
  }
}
```

### Azure AD Application Proxy (On-Prem to Zero Trust)
```bash
# Publish on-premises app via App Proxy (no inbound firewall rules)
az ad app proxy application create \
  --display-name "Internal HR System" \
  --internal-url "http://hr-server.internal:8080" \
  --external-url "https://hr.contoso.com" \
  --connector-group-id "default"

# App is now accessible only to authenticated, authorized users
# No VPN, no open firewall ports
```

### Azure Confidential Computing
```bash
# Run workloads in hardware-based trusted execution environments
az vm create \
  --name confidential-vm \
  --resource-group myResourceGroup \
  --image "Canonical:UbuntuServer:22_04-lts-gen2:latest" \
  --security-type ConfidentialVM \
  --os-disk-security-encryption-type DiskWithVMGuestState \
  --enable-secure-boot true \
  --enable-vtpm true
```

## Summary Decision Matrix

| Zero Trust Pillar | Traditional | Zero Trust | Implementation |
|------------------|-------------|------------|----------------|
| Identity | Username+password | MFA + passwordless + risk-based | Okta, Azure AD, Google Identity |
| Device | Trusted if on network | Must be managed, compliant, healthy | Intune, Jamf, Endpoint Verification |
| Network | VPN = trusted | Network is irrelevant trust factor | BeyondCorp, Zscaler, Cloudflare Access |
| Application | IP-based access | Identity-based access, mTLS | SPIFFE/SPIRE, OPA, AWS IAM |
| Data | Protection at perimeter | Encryption everywhere, DLP, classification | KMS, RMS, DLP tools |
| Visibility | Perimeter monitoring | Continuous monitoring, UEBA | SIEM, XDR, CASB |

Zero Trust is not a product—it's a strategy and architectural philosophy. It acknowledges that perimeter-based security is obsolete in a world of cloud, remote work, and API-driven architectures. Every solution architect building systems today must design with Zero Trust principles: assume the network is hostile, verify every request, minimize blast radius, and instrument everything for continuous monitoring. Start with identity and device trust for user access, progress to workload identity and mTLS for service communication, and evolve to micro-segmentation and data-level controls.
