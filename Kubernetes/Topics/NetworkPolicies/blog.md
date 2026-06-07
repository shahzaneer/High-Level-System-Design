# Network Policies

NetworkPolicies are Kubernetes-native firewall rules controlling pod-to-pod communication. By default, all pods can communicate with all other pods (no isolation). NetworkPolicies define ALLOW rules—they are additive whitelists, not deny-rule blacklists. To use them, the cluster must have a CNI plugin that supports NetworkPolicy (Calico, Cilium, Weave, or cloud-managed like GKE Dataplane V2).

## Declarative (YAML)

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}       # Selects ALL pods
  policyTypes:
  - Ingress
---
# Default deny all egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
---
# Allow specific traffic (whitelist model)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
  - Ingress
  - Egress

  ingress:
  # Allow from API gateway pods
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  
  # Allow from monitoring (Prometheus scraping)
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090

  egress:
  # Allow to database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow DNS (required!)
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  
  # Allow to specific external IPs (if needed)
  - to:
    - ipBlock:
        cidr: 172.217.0.0/16   # Google APIs
        except:
        - 172.217.1.0/24       # except this range
    ports:
    - protocol: TCP
      port: 443
```

## Common Patterns

```yaml
# Allow ingress only from same namespace
ingress:
- from:
  - podSelector: {}

# Allow from specific namespace (all pods in it)
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: monitoring

# Allow from anywhere (specific port)
ingress:
- from:
  - ipBlock:
      cidr: 0.0.0.0/0
  ports:
  - port: 443
```

## Imperative

```bash
# Create NetworkPolicy from YAML only—no imperative equivalent
kubectl apply -f network-policy.yaml
kubectl get networkpolicies -n production
kubectl describe networkpolicy order-service-policy -n production
```

## Isolation Guarantees

| Scenario | Without NetworkPolicy | With deny-all + explicit allows |
|----------|----------------------|-------------------------------|
| Pod A → Pod B (same namespace) | Allowed | Denied unless explicitly allowed |
| Pod A → Pod B (different namespace) | Allowed | Denied unless explicitly allowed |
| External → Pod B | Not applicable | Denied |
| Pod A → External IP | Allowed | Denied unless explicitly allowed |
| Pod A → DNS (kube-dns) | Allowed | MUST explicitly allow! |

## NetworkPolicy is NOT a Firewall

They are Layer-3/4 controls only (IP/port). For Layer-7 filtering (HTTP path, host header), use service mesh (Istio AuthorizationPolicy) or ingress controller with WAF.

## Imperative vs Declarative

No imperative creation for NetworkPolicies. Always declarative YAML. They must be version-controlled as part of the security posture.
