# Service Accounts

Service Accounts provide identities for Pods (not humans). When a Pod makes API calls to the Kubernetes API or authenticates to external services, it uses its Service Account. Every namespace has a `default` Service Account auto-mounted to every pod—but you should create dedicated Service Accounts with specific RBAC permissions.

Modern workload identity patterns (EKS IRSA, GKE Workload Identity, AKS Workload Identity) extend Service Accounts to authenticate to cloud APIs without static credentials.

## Imperative

```bash
# Create Service Account
kubectl create serviceaccount order-processor -n production

# View
kubectl get serviceaccounts -n production
kubectl describe sa order-processor -n production

# The SA automatically creates a token (used for API authentication)
kubectl get secret $(kubectl get sa order-processor -o jsonpath='{.secrets[0].name}') -o yaml
```

## Declarative (YAML)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-processor
  namespace: production
  annotations:
    # AWS IRSA: map K8s SA to AWS IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-processor
    # GCP Workload Identity
    iam.gke.io/gcp-service-account: order-processor@my-project.iam.gserviceaccount.com
    # Azure Workload Identity
    azure.workload.identity/client-id: "CLIENT_ID"
---
# Pod using the Service Account
apiVersion: v1
kind: Pod
metadata:
  name: order-service
spec:
  serviceAccountName: order-processor    # Explicit SA
  automountServiceAccountToken: true     # Mount token at /var/run/secrets/...
  containers:
  - name: app
    image: order-service:latest
    # AWS SDK/Google SDK/Azure SDK automatically use 
    # the service account token to get cloud credentials
```

## Disable Default SA Token Mounting

```yaml
apiVersion: v1
kind: Pod
spec:
  automountServiceAccountToken: false   # No API access from this pod
  containers:
  - name: app
    image: myapp:v1
```

## Token Projection (Short-Lived Tokens)

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
      readOnly: true
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600       # 1-hour token
          audience: api                 # Bound to specific audience
```

## IRSA (IAM Roles for Service Accounts) - AWS

```bash
# Create OIDC provider for EKS cluster
eksctl utils associate-iam-oidc-provider --cluster prod --approve

# Create IAM role with trust policy for the service account
eksctl create iamserviceaccount \
  --name order-processor \
  --namespace production \
  --cluster prod \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

```yaml
# This creates the ServiceAccount with the annotation automatically
# Pod using it gets temporary AWS credentials via STS
# No long-lived IAM access keys anywhere
```

## Best Practices

1. **Dedicated SA per application**: Don't reuse the default SA
2. **Least privilege via RBAC**: SA → Role → RoleBinding with minimal permissions
3. **Use cloud workload identity**: No static credentials for cloud APIs
4. **Disable automount when not needed**: Defense in depth
5. **Use projected tokens**: Short-lived, audience-bound, more secure

## Imperative vs Declarative

Imperative for quick SA creation. Declarative for production, especially with cloud IAM annotations and RBAC bindings. Always declarative in GitOps workflows.
