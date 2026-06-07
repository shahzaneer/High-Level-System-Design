# Gateway API (Next-Gen Ingress)

The Gateway API is the successor to Kubernetes Ingress, providing more expressive, role-oriented, and portable HTTP routing. While Ingress conflates infrastructure concerns (TLS, load balancer type) with routing rules, Gateway API cleanly separates them: Cluster Operators manage Gateways (infrastructure); Application Developers manage HTTPRoutes (routing). It supports header-based routing, weighted traffic splitting, cross-namespace routing, and gRPC.

Gateway API is CRD-based (no annotations), supports multiple Gateway Classes (different implementations: cloud LBs, service mesh, open-source), and is the strategic direction for Kubernetes networking. Ingress is not deprecated, but Gateway API is where innovation is happening.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                SEPARATION OF CONCERNS                │
├─────────────────────────────────────────────────────┤
│ GatewayClass (Infrastructure Provider):              │
│   Defines WHAT KIND of gateway this is              │
│   Controller: aws-lb, gke-lb, istio, nginx          │
│                                                     │
│ Gateway (Cluster Operator):                          │
│   Defines WHERE traffic enters                      │
│   Listeners: ports, protocols, TLS certificates     │
│   Address: IP, hostname                             │
│                                                     │
│ HTTPRoute (Application Developer):                   │
│   Defines WHERE traffic goes                        │
│   Rules: path matching, header matching, weights    │
│   BackendRefs: Services to route to                 │
└─────────────────────────────────────────────────────┘
```

## YAML Example

```yaml
# Gateway: cluster operator managed
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: api-gateway
  namespace: ingress
spec:
  gatewayClassName: aws-lb
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: "*.example.com"
    tls:
      mode: Terminate
      certificateRefs:
      - name: wildcard-tls
---
# HTTPRoute: developer managed
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: order-routes
  namespace: production
spec:
  parentRefs:
  - name: api-gateway
    namespace: ingress
  hostnames:
  - api.example.com
  rules:
  # Route by path
  - matches:
    - path:
        type: PathPrefix
        value: /api/orders
    backendRefs:
    - name: order-service
      port: 8080
  
  # Canary deployment (weighted routing)
  - matches:
    - path:
        type: PathPrefix
        value: /api/payments
    backendRefs:
    - name: payment-service-v1
      port: 8080
      weight: 90
    - name: payment-service-v2
      port: 8080
      weight: 10
  
  # Header-based routing (A/B testing)
  - matches:
    - path:
        type: PathPrefix
        value: /api/catalog
      headers:
      - name: X-Experimental-Features
        value: "enabled"
    backendRefs:
    - name: catalog-service-beta
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /api/catalog
    backendRefs:
    - name: catalog-service
      port: 8080
```

## Gateway API vs Ingress

| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| Role separation | No (all in one resource) | Yes (GatewayClass, Gateway, Route) |
| Header-based routing | Annotation-dependent | Native |
| Weighted routing | Annotation-dependent | Native |
| Cross-namespace routing | No | Yes (ReferenceGrant) |
| TCP/UDP routing | No | Yes |
| gRPC routing | No | Yes |
| Multiple gateway implementations | Class annotation | GatewayClass resource |

## Gateway Implementations

```yaml
# AWS Load Balancer Controller
gatewayClassName: aws-lb

# GKE Gateway Controller
gatewayClassName: gke-l7-global-external-managed

# Istio
gatewayClassName: istio

# Envoy Gateway
gatewayClassName: eg

# NGINX Gateway Fabric
gatewayClassName: nginx
```

Gateway API is the future of Kubernetes HTTP routing. It solves Ingress's limitations (expression, role separation, portability) while supporting the same backends. New Kubernetes deployments should use Gateway API; existing Ingress deployments should migrate over time.
