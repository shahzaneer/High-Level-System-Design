# RBAC (Role-Based Access Control)

RBAC controls who can do what in Kubernetes. It binds subjects (users, groups, service accounts) to roles that define permissions on resources. RBAC is enabled by default and is the primary authorization mechanism in Kubernetes—without it, anyone with API access can do anything.

Key resources: Role (namespaced permissions), ClusterRole (cluster-wide permissions), RoleBinding (binds Role to subject), ClusterRoleBinding (binds ClusterRole to subject).

## Imperative

```bash
# Create Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n production

# Create RoleBinding
kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=alice \
  -n production

# Create ClusterRole
kubectl create clusterrole secret-reader \
  --verb=get,list \
  --resource=secrets

# Create ClusterRoleBinding
kubectl create clusterrolebinding read-secrets \
  --clusterrole=secret-reader \
  --group=developers

# Check permissions (can I create deployments?)
kubectl auth can-i create deployments -n production
kubectl auth can-i '*' '*' --as=alice  # Check what alice can do
```

## Declarative (YAML)

```yaml
# Role: namespaced permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-developer
  namespace: production
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "pods/log", "deployments", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods/exec", "pods/portforward"]
  verbs: ["create"]
---
# RoleBinding: bind role to subject
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-developers-binding
  namespace: production
subjects:
- kind: User
  name: alice@company.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: order-engineering
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: order-service
  namespace: production
roleRef:
  kind: Role
  name: order-developer
  apiGroup: rbac.authorization.k8s.io
---
# ClusterRole: cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/metrics"]
  verbs: ["get", "list", "watch"]
---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-readers-binding
subjects:
- kind: Group
  name: sre-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

## Aggregated ClusterRoles

```yaml
# Admin role: combines view + edit roles
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-admin
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies"]
  verbs: ["*"]
```

## Best Practices

1. **Principle of Least Privilege**: Grant minimum permissions needed
2. **Use Groups, not Users**: Easier to manage; add/remove users from groups
3. **Avoid wildcards**: `verbs: ["*"]` and `resources: ["*"]` are dangerous
4. **Namespace scope when possible**: Use Role + RoleBinding over ClusterRole
5. **Audit**: `kubectl get rolebindings,clusterrolebindings --all-namespaces`
6. **Use cloud IAM integration**: EKS (aws-auth ConfigMap, IRSA), GKE (Workload Identity), AKS (Azure RBAC)

## Imperative vs Declarative

Imperative (`kubectl create role`) is great for quick permission checks and ad-hoc bindings. Declarative YAML is required for any persistent RBAC policy—version controlled, auditable, and reproducible.
