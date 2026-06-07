# Ingress

Ingress exposes HTTP/HTTPS routes from outside the cluster to Services inside. It provides load balancing, TLS termination, and name-based virtual hosting. An Ingress requires an Ingress Controller (nginx-ingress, AWS Load Balancer Controller, GCE Ingress, etc.) to be installed in the cluster.

Gateway API is the next-generation replacement for Ingress, providing more expressive routing and better role separation.

## Imperative (kubectl)

```bash
# Create a simple ingress (limited imperatively; YAML is preferred)
kubectl create ingress order-ingress \
  --rule="api.example.com/orders=order-service:8080" \
  --class=nginx

# TLS ingress
kubectl create ingress order-ingress-tls \
  --rule="api.example.com/orders=order-service:8080" \
  --tls=api-tls-secret \
  --class=nginx
```

## Declarative (YAML)

```yaml
# Basic Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-api
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/orders(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: order-service
            port:
              number: 8080
      - path: /api/payments(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 8080
```

## Path Types

| Type | Behavior |
|------|----------|
| Prefix | Matches path prefix (e.g., `/api` matches `/api/v1/orders`) |
| Exact | Matches URL path exactly (case-sensitive) |
| ImplementationSpecific | Controller-dependent matching |

## TLS Configuration

```bash
# Create TLS secret
kubectl create secret tls api-tls \
  --cert=fullchain.pem \
  --key=privkey.pem

# Or use cert-manager (automated)
```

```yaml
# cert-manager annotation for auto-TLS
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

## Multiple Paths and Backends

```yaml
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/orders
        pathType: Prefix
        backend:
          service:
            name: order-service-v2  # Specific version
            port: { number: 8080 }
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port: { number: 8080 }
  
  - host: beta.api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: order-service-beta
            port: { number: 8080 }
```

## Default Backend

```yaml
spec:
  defaultBackend:
    service:
      name: default-404
      port:
        number: 80
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port: { number: 8080 }
```

## Cloud-Specific Ingress Classes

```yaml
# AWS: Application Load Balancer
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
spec:
  ingressClassName: alb

# GCP: Global HTTPS Load Balancer
metadata:
  annotations:
    kubernetes.io/ingress.class: gce
    kubernetes.io/ingress.global-static-ip-name: api-ip
spec:
  ingressClassName: gce

# Azure: Application Gateway
metadata:
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  ingressClassName: azure-application-gateway
```

## Gateway API (Next-Gen)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: order-routes
spec:
  parentRefs:
  - name: api-gateway
  hostnames:
  - api.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/orders
    backendRefs:
    - name: order-service
      port: 8080
      weight: 90             # 90% traffic
    - name: order-service-v2
      port: 8080
      weight: 10             # 10% canary
```

## Imperative vs Declarative

Ingress almost always declarative. Imperative `kubectl create ingress` is too limited for TLS, path types, and annotations. Always define Ingress in YAML, version-controlled.
