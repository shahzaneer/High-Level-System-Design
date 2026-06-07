# Identity & Access Management (IAM)

## Introduction
Identity & Access Management (IAM) is the security discipline that ensures the right individuals and services have the right access to the right resources at the right time. In cloud-native and microservices architectures, IAM extends beyond human users to encompass service accounts, CI/CD pipelines, IoT devices, and third-party integrations—every entity that interacts with your system must be identified, authenticated, and authorized.

The shift from perimeter-based security (firewalls, VPNs) to identity-based security ("identity is the new perimeter") has made IAM the central nervous system of cloud security. A single misconfigured IAM policy can expose an entire cloud account. Understanding IAM deeply—roles, policies, federation, temporary credentials—is arguably the most important security skill for a solution architect.

## Definition

**Identity & Access Management (IAM)** is a framework of policies, processes, and technologies that manage digital identities and control access to resources. Core components:

- **Identity**: A digital representation of a user, service, or device (user account, service account, role)
- **Authentication**: Verifying who the identity is (passwords, MFA, certificates, SSO)
- **Authorization**: Determining what the identity can do (policies, roles, permissions)
- **Audit**: Recording and monitoring access events for compliance and security analysis

**Key principles**:
- **Least Privilege**: Identities should have only the permissions necessary to perform their function, nothing more
- **Separation of Duties**: Critical operations require multiple identities (approver + executor)
- **Just-in-Time (JIT) Access**: Permissions granted temporarily, only when needed, and automatically revoked
- **Zero Standing Privileges**: No permanent administrative access; all elevated access is time-bound and audited

## Concept Explanation

### IAM Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    IAM SYSTEM                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │  Users   │  │  Groups  │  │  Roles   │             │
│  │  (human) │  │ (collections)│ (federated/│          │
│  │          │  │          │  │  service) │             │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘             │
│       │             │              │                    │
│       └─────────┬───┘──────────────┘                    │
│                 │                                      │
│       ┌─────────▼─────────┐                            │
│       │   Policy Engine   │                            │
│       │  (Evaluates ALL   │                            │
│       │   applicable      │                            │
│       │   policies:       │                            │
│       │   Allow/Deny)     │                            │
│       └─────────┬─────────┘                            │
│                 │                                      │
│       ┌─────────▼─────────┐                            │
│       │   Resources       │                            │
│       │ (S3, EC2, RDS,    │                            │
│       │  Lambda, APIs...) │                            │
│       └───────────────────┘                            │
└─────────────────────────────────────────────────────────┘
```

### Policy Evaluation Logic

Cloud IAM systems use an explicit deny-overrides-allow model:

```
1. By default, ALL access is DENIED (implicit deny)
2. Any explicit ALLOW grants permission... UNLESS
3. An explicit DENY overrides any ALLOW

Evaluation:
  Is there an explicit DENY matching the request? → DENY
  Is there an explicit ALLOW matching the request? → ALLOW
  Otherwise → IMPLICIT DENY (default)
```

### RBAC (Role-Based Access Control)

Permissions are grouped into roles. Users/groups are assigned roles. Roles are assigned to resources at a scope.

```json
{
  "roles": [
    {
      "name": "OrderViewer",
      "description": "Can view orders and their status",
      "permissions": [
        "orders:read",
        "orders:list",
        "inventory:read"
      ]
    },
    {
      "name": "OrderManager",
      "description": "Can manage orders (view, create, update status)",
      "permissions": [
        "orders:read",
        "orders:list",
        "orders:create",
        "orders:update:status",
        "inventory:read"
      ]
    }
  ],
  "assignments": [
    {"role": "OrderViewer", "principal": "alice@company.com", "scope": "project:retail"},
    {"role": "OrderManager", "principal": "bob@company.com", "scope": "project:retail"}
  ]
}
```

### ABAC (Attribute-Based Access Control)

Policy decisions are based on attributes of the principal, resource, action, and environment:

```
Policy: Allow access to S3 objects IF:
  - Principal.Department == Resource.Department
  - Action in ["s3:GetObject", "s3:PutObject"]
  - Resource.Tags["Classification"] != "restricted"
  - Environment.SourceIP in 10.0.0.0/8
  - Environment.CurrentTime between 09:00-17:00
```

### Service Accounts and Workload Identity

In microservices, services need their own identities separate from human users:

```yaml
# Kubernetes service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-processor
  namespace: production
  annotations:
    # AWS IAM Roles for Service Accounts (IRSA)
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-processor-role
---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: order-processor
  containers:
  - name: app
    image: order-processor:latest
```

```python
# AWS SDK automatically uses the service account's IAM role
import boto3

# No explicit credentials needed—uses IRSA/OIDC federation
s3 = boto3.client('s3')
# This call is authorized by the pod's service account IAM role
s3.get_object(Bucket='orders', Key='order-123.json')
```

### Federation and Single Sign-On (SSO)

```
Corporate Identity (Azure AD / Okta / Google Workspace)
         │
         │  SAML / OIDC Federation
         │
         ▼
    Cloud IAM (AWS IAM / GCP IAM)
         │
         │  Assume Role (temporary credentials via STS)
         │
         ▼
    Cloud Resources (short-lived access)
```

```python
# AWS STS: assume role via SAML federation
import boto3

sts = boto3.client('sts')

response = sts.assume_role_with_saml(
    RoleArn='arn:aws:iam::123456789:role/Developer',
    PrincipalArn='arn:aws:iam::123456789:saml-provider/Okta',
    SAMLAssertion=saml_assertion,
    DurationSeconds=3600  # 1 hour max
)

# Temporary credentials
temp_credentials = response['Credentials']
# AccessKeyId, SecretAccessKey, SessionToken (all short-lived)
```

### Permission Boundaries and Guardrails

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["iam:CreateRole", "iam:AttachRolePolicy"],
      "Resource": "*"
    }
  ]
}

// But Permission Boundary limits what roles this role can create:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
// Even with Admin, this role can only operate in us-east-1
```

### Attribute-Based Access Control (ABAC) Tags

Using resource tags for dynamic authorization:

```python
# Tag-based authorization pattern
def create_resource(user, resource_data):
    # Owner tag is automatically set to the creator's department
    tags = {
        'Owner': user.id,
        'Department': user.department,
        'Environment': 'production',
        'CostCenter': user.cost_center
    }
    
    resource = cloud_api.create_resource(data=resource_data, tags=tags)
    return resource

# IAM policy grants access based on tag match
# "Allow s3:GetObject if s3:ExistingObjectTag/Department == ${aws:PrincipalTag/Department}"
# This means Alice (Dept=Engineering) can only access S3 objects tagged Department=Engineering
```

## Layman's Explanation

### IAM: The Corporate Badge System
Think of IAM as the security badge system for a large corporate campus:

- **Identity**: Your employee badge with your photo, name, and employee ID
- **Authentication**: The security guard at the entrance checks your badge is real (and that it's actually you). The turnstile (MFA) requires both a badge swipe AND a PIN.
- **Authorization**: Your badge's access level determines which doors open. An engineer's badge opens the engineering wing but not the finance department. The CFO's badge opens finance doors. Some doors (server room) require both manager AND IT badges together (separation of duties).
- **Least Privilege**: Interns get access to common areas and their team's floor only. They don't get building-wide master keys.
- **JIT Access**: Need to access the data center for emergency maintenance? You get a temporary access code that expires in 4 hours, and every access is logged.
- **Audit Trail**: Every badge swipe is recorded. Security can pull a report showing exactly who entered which room and when.

### The "Keys to the Kingdom" Problem
In the cloud, the IAM administrator role is like having a master key that opens every door, every safe, and every file cabinet in the entire company. A single compromised admin account can delete all infrastructure, exfiltrate all data, and shut down the business. This is why:
1. Admin accounts need MFA (no exceptions)
2. Long-term access keys are banned for humans
3. All admin activity is logged to an immutable audit trail
4. Break-glass emergency access procedures are documented and tested

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Account/Project Structure**: Single cloud account vs multi-account (AWS Organizations, GCP Folders, Azure Management Groups). Multi-account provides blast radius containment—a compromise in dev doesn't affect prod. This is the most fundamental cloud security decision.
- **Identity Provider Strategy**: Centralized (single IdP for all clouds and SaaS) vs decentralized. AWS SSO, GCP Cloud Identity, and Azure AD can federate, enabling single-pane-of-glass identity management across all three clouds.
- **Service Account Design**: Each microservice gets its own identity with minimal permissions. Shared credentials across services is a security anti-pattern equivalent to sharing passwords. Infrastructure-as-code deploys both the service and its IAM role together.
- **Cross-Account Access**: How do services in Account A access resources in Account B? IAM roles with trust policies, resource-based policies, or both? This design determines the security posture of multi-account architectures.

### Business Impact
- **Breach Containment**: Proper IAM means a compromised service account in the monitoring system can't delete production databases. Each service's blast radius is limited to its own IAM permissions.
- **Compliance**: SOC 2, PCI DSS (Req 7), HIPAA, and ISO 27001 all require formal access control policies, least privilege enforcement, and access reviews. Automated IAM with infrastructure-as-code makes compliance demonstrable and auditable.
- **Operational Efficiency**: Automated role provisioning via CI/CD and SSO eliminates manual access management that consumes 5-10% of IT operations time. Developers self-service through approved paths (JIT elevation, approved role templates).
- **Audit Readiness**: When an auditor asks "Who has access to production databases?" you can answer with a query, not a 2-week manual audit. Cloud IAM + CloudTrail/Logging provides complete, immutable access logs.

### Common IAM Pitfalls
- Overly broad wildcard policies (`s3:*` on `*`) — the #1 cloud security finding
- Long-lived access keys for human users (use SSO with temporary credentials instead)
- Not deleting unused IAM roles and users (attack surface creep)
- Hardcoding credentials in source code or environment variables (use workload identity)

## On-Premises Examples

### OpenLDAP + Keycloak Identity Federation
```bash
# OpenLDAP as corporate directory
slapd -h ldap:/// ldaps:///

ldapadd -x -D "cn=admin,dc=company,dc=com" -W <<EOF
dn: ou=users,dc=company,dc=com
objectClass: organizationalUnit
ou: users

dn: uid=alice,ou=users,dc=company,dc=com
objectClass: inetOrgPerson
uid: alice
cn: Alice Johnson
mail: alice@company.com
userPassword: {SSHA}...
EOF
```

```bash
# Keycloak federation with LDAP
# Configure User Federation → LDAP → corporate LDAP directory
# Users authenticate via Keycloak, Keycloak verifies against LDAP
```

### HashiCorp Vault Dynamic Credentials
```bash
# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL role with dynamic credentials
vault write database/roles/order-reader \
  db_name=postgres-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' \
    VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Application requests temporary credentials
vault read database/creds/order-reader
# {"username": "v-token-order-abc123", "password": "temp-pwd-xyz789",
#  "lease_duration": 3600, "lease_id": "..."}
```

### FreeIPA
Open-source identity management (users, groups, hosts, policies):

```bash
ipa-server-install --realm=COMPANY.COM --domain=company.com

# Create user
ipa user-add alice --first=Alice --last=Johnson --email=alice@company.com

# Create RBAC role
ipa role-add --desc="Order Viewers" orderviewers
ipa role-add-member --users=alice orderviewers

# Host-based access control (which users can SSH to which servers)
ipa hbacrule-add --hostcat=all allow-developers
ipa hbacrule-add-user --users=alice allow-developers
```

## AWS Examples

### AWS IAM Core Architecture
```hcl
# IAM Role for EC2 instance (service account)
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "app_instance" {
  name               = "app-instance-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
  
  # Permission boundary limits what this role can do
  permissions_boundary = aws_iam_policy.permission_boundary.arn
  
  max_session_duration = 3600  # 1 hour
}

# ABAC: Tag-based conditions
data "aws_iam_policy_document" "dynamodb_access" {
  statement {
    effect = "Allow"
    actions = [
      "dynamodb:GetItem",
      "dynamodb:Query",
      "dynamodb:PutItem",
      "dynamodb:UpdateItem"
    ]
    resources = ["arn:aws:dynamodb:*:*:table/Orders"]
    
    condition {
      test     = "StringEquals"
      variable = "dynamodb:LeadingKeys"
      values   = ["${aws:PrincipalTag/Department}"]
    }
    
    condition {
      test     = "StringEquals"
      variable = "aws:ResourceTag/Environment"
      values   = ["${aws:PrincipalTag/Environment}"]
    }
  }
}
```

### AWS Organizations + SCPs (Service Control Policies)
Guardrails applied across entire organization:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllOutsideUS",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-east-2", "us-west-2"]
        }
      }
    },
    {
      "Sid": "RequireIMDSv2",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:MetadataHttpTokens": "required"
        }
      }
    }
  ]
}
```

### AWS IAM Identity Center (SSO) + Permission Sets
```bash
# Create permission set for developers
aws sso-admin create-permission-set \
  --instance-arn "arn:aws:sso:::instance/ssoins-xxxx" \
  --name "DeveloperAccess" \
  --description "Developer access with guardrails" \
  --session-duration "PT8H"

# Attach managed policy
aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn "arn:aws:sso:::instance/ssoins-xxxx" \
  --permission-set-arn "arn:aws:sso:::permissionSet/..." \
  --managed-policy-arn "arn:aws:iam::aws:policy/PowerUserAccess"

# Assign to group in dev account only
aws sso-admin create-account-assignment \
  --instance-arn "arn:aws:sso:::instance/ssoins-xxxx" \
  --target-id "111111111111" \
  --target-type "AWS_ACCOUNT" \
  --permission-set-arn "arn:aws:sso:::permissionSet/..." \
  --principal-type "GROUP" \
  --principal-id "group-id"
```

### EKS Pod Identity (IRSA)
```yaml
# Service account with IAM role for S3 access
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-processor
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-processor-s3-role
---
# IAM trust policy using OIDC federation
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/CLUSTER_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/CLUSTER_ID:sub": "system:serviceaccount:production:order-processor"
        }
      }
    }
  ]
}
```

## GCP Examples

### Cloud IAM Hierarchy
```
Organization
  └── Folder (Department: Engineering)
       ├── Project (Production)
       │    ├── Resources (GCE, GCS, Cloud SQL, etc.)
       │    └── Service Accounts
       └── Project (Development)
            └── Resources
```

```bash
# Grant role at folder level (inherited by all child projects)
gcloud resource-manager folders add-iam-policy-binding FOLDER_ID \
  --member="user:alice@company.com" \
  --role="roles/viewer" \
  --condition="expression=request.time.getHours('WET') >= 9 && request.time.getHours('WET') <= 17,title=BusinessHours"

# Custom role with fine-grained permissions
gcloud iam roles create OrderViewer \
  --project=my-project \
  --title="Order Viewer" \
  --description="Can view orders and inventory" \
  --permissions="storage.objects.get,storage.objects.list,cloudsql.instances.get" \
  --stage=GA
```

### Workload Identity (GKE Pods → GCP Services)
```bash
# Create GCP service account
gcloud iam service-accounts create gke-order-processor

# Bind Kubernetes service account to GCP service account
gcloud iam service-accounts add-iam-policy-binding \
  gke-order-processor@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project.svc.id.goog[production/order-processor]"

# Kubernetes service account annotation
kubectl annotate serviceaccount order-processor \
  --namespace production \
  iam.gke.io/gcp-service-account=gke-order-processor@my-project.iam.gserviceaccount.com
```

### Organization Policy Service (Guardrails)
```yaml
# constraints/policy.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: GCPStorageBucketPolicy
metadata:
  name: require-uniform-bucket-level-access
spec:
  match:
    namespaces: ["production"]
  parameters:
    mode: "deny"
```

```bash
# Disable service account key creation (enforce workload identity)
gcloud org-policies set-policy policy.yaml
```

## Azure Examples

### Azure RBAC
```bash
# Built-in role assignment at resource group scope
az role assignment create \
  --assignee alice@company.com \
  --role "Contributor" \
  --resource-group myResourceGroup

# Custom role
az role definition create --role-definition '{
  "Name": "Order Processor",
  "Description": "Process orders but cannot delete",
  "Actions": [
    "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read",
    "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/write"
  ],
  "NotActions": [
    "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/delete"
  ],
  "AssignableScopes": ["/subscriptions/SUBSCRIPTION_ID"]
}'

# PIM (Privileged Identity Management) - JIT elevation
az role assignment create \
  --assignee alice@company.com \
  --role "Contributor" \
  --scope "/subscriptions/SUBSCRIPTION_ID" \
  --condition "@Resource[Microsoft.Authorization/roleEligibilityScheduleRequests] AssignedToRequestor()" \
  --condition-version "2.0"
```

### Azure AD PIM (Just-in-Time Access)
```json
{
  "properties": {
    "roleDefinitionId": "/subscriptions/.../roleDefinitions/...",
    "principalId": "user-id",
    "scheduleInfo": {
      "startDateTime": null
    },
    "assignmentType": "Eligible",
    "memberType": "Direct"
  }
}
```

### Managed Identities (Azure Workload Identity)
```bash
# System-assigned managed identity
az vm identity assign \
  --name myVM \
  --resource-group myResourceGroup

# The VM now has an identity in Azure AD
# Application code uses DefaultAzureCredential (no secrets)
```

```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

# Automatically uses managed identity when running on Azure VM/App Service
credential = DefaultAzureCredential()
blob_service = BlobServiceClient(
    account_url="https://mystorage.blob.core.windows.net",
    credential=credential
)
```

### Azure Blueprints (Governance at Scale)
```json
{
  "properties": {
    "resourceGroups": {
      "networking": {
        "location": "eastus"
      }
    },
    "roleAssignments": [
      {
        "principalIds": ["contributor-group-id"],
        "roleDefinitionId": "/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c",
        "scope": "/subscriptions/SUBSCRIPTION_ID"
      }
    ],
    "policyAssignments": [
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/require-sql-tde",
        "parameters": {}
      }
    ]
  }
}
```

## Summary Decision Matrix

| IAM Capability | AWS | GCP | Azure |
|---------------|-----|-----|-------|
| Human identities | IAM Identity Center (SSO) | Cloud Identity / Workforce Identity | Azure AD / Entra ID |
| Service identities | IAM Roles + Instance Profile | Service Accounts | Managed Identities |
| Kubernetes workload identity | IRSA (EKS Pod Identity) | Workload Identity | Azure AD Pod Identity / Workload Identity |
| Organization guardrails | SCPs + RCPs | Organization Policy Service | Azure Policy + Blueprints |
| JIT privileged access | IAM Roles Anywhere + STS | IAM Conditions (time-bound) | PIM (Privileged Identity Management) |
| ABAC | Tags + Condition keys | IAM Conditions | Azure ABAC (conditions on roles) |
| Cross-account access | AssumeRole (trust policies) | Service account impersonation | Cross-tenant access |
| Audit trail | CloudTrail | Cloud Audit Logs | Activity Log + Azure Monitor |

IAM is the foundation on which all other cloud security controls are built. A well-designed IAM architecture—multi-account strategy, least-privilege roles, workload identity for services, SSO for humans, SCPs/guardrails for invariants—is the difference between a secure cloud environment and the next cloud breach headline. Solution architects must design IAM before designing infrastructure, because IAM determines who can do what to which resources, and retrofitting access controls is infinitely harder than building them in from the start.
