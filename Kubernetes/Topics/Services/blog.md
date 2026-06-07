# Services

Services provide stable network endpoints for Pods. Since Pods are ephemeral (IPs change on restart), a Service gives you a consistent IP address and DNS name that load-balances to healthy backend Pods. Services use label selectors to find their target Pods.

Service types: ClusterIP (internal only), NodePort (exposes on node IP at static port), LoadBalancer (provisions cloud load balancer), ExternalName (DNS CNAME alias).

## Imperative (kubectl)

```bash
# Expose a deployment as a ClusterIP service (internal)
kubectl expose deployment order-service \
  --port=8080 --target-port=8080 \
  --name=order-svc

# Expose as NodePort (external via node IP:30000-32767)
kubectl expose deployment order-service \
  --type=NodePort --port=8080 --target-port=8080

# Expose as LoadBalancer (cloud LB)
kubectl expose deployment order-service \
  --type=LoadBalancer --port=443 --target-port=8080

# Get service details
kubectl get svc
kubectl describe svc order-svc
kubectl get endpoints order-svc  # Shows pod IPs backing the service
```

## Declarative (YAML)

```yaml
# ClusterIP: internal-only (default)
apiVersion: v1
kind: Service
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    protocol: TCP
  sessionAffinity: None  # or ClientIP for sticky sessions
---
# Headless Service: no ClusterIP, returns pod IPs directly (for StatefulSets)
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
---
# Multi-port Service
apiVersion: v1
kind: Service
metadata:
  name: app-multi
spec:
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  - name: grpc
    port: 50051
    targetPort: 50051
```

## DNS Resolution

```bash
# Within the cluster, services resolve via DNS:
# <service-name>.<namespace>.svc.cluster.local

# From any pod:
curl http://order-service.production.svc.cluster.local:8080/health
curl http://order-service:8080/health  # Same namespace shortcut

# Headless service returns ALL pod IPs:
nslookup db-headless.production.svc.cluster.local
# Returns: 10.0.2.15, 10.0.3.22, 10.0.1.8 (all pod IPs)
```

## Service Types Comparison

| Type | Access | Use Case | Cloud Cost |
|------|--------|----------|------------|
| ClusterIP | Internal cluster only | Service-to-service communication | Free |
| NodePort | nodeIP:30000-32767 | Development, demo | Free |
| LoadBalancer | External IP provisioned | Production ingress | Cloud LB cost |
| ExternalName | CNAME to external DNS | Proxy to external service | Free |

## LoadBalancer Annotations

```yaml
# AWS NLB
apiVersion: v1
kind: Service
metadata:
  name: order-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone: "true"
spec:
  type: LoadBalancer
  selector:
    app: order-service
  ports:
  - port: 443
    targetPort: 8080
```

## EndpointSlices (Scalable Endpoints)

Kubernetes automatically creates EndpointSlices (not Endpoints) for services with many pods, sharding them into groups of 100 for better scalability:

```bash
kubectl get endpointslice -l kubernetes.io/service-name=order-service
```

## Imperative vs Declarative

Imperative (`kubectl expose`) is fast for prototyping. Declarative (YAML) is required for production: annotations, headless services, multi-port configs, and session affinity can't all be expressed imperatively.
