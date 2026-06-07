# Sidecar & Ambassador Patterns

The Sidecar and Ambassador are container patterns in Kubernetes that extend application functionality without modifying the application code. They run as additional containers in the same Pod, sharing the same network namespace and volumes.

**Sidecar**: Extends the application (logging, monitoring, proxying). Runs alongside the main container for the Pod's lifetime.

**Ambassador**: Proxies network traffic for the application (service discovery, TLS termination, connection pooling). The app thinks it's talking to localhost, but the ambassador handles the complexity.

Both patterns follow the principle: separate cross-cutting concerns from business logic, deploy them together atomically.

## Sidecar Pattern

### Logging Sidecar

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: order-service:v1
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  
  # Sidecar: reads log files, ships to centralized logging
  - name: log-shipper
    image: fluent/fluent-bit:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true
    - name: fluent-config
      mountPath: /fluent-bit/etc
  
  volumes:
  - name: logs
    emptyDir: {}
  - name: fluent-config
    configMap:
      name: fluent-bit-config
```

### Monitoring Sidecar (Prometheus Exporter)

```yaml
  containers:
  - name: app
    image: myapp:v1
  - name: prometheus-exporter
    image: nginx/nginx-prometheus-exporter:latest
    args: ["-nginx.scrape-uri", "http://localhost:8080/metrics"]
    ports:
    - containerPort: 9113
```

### Istio Sidecar (Service Mesh)

```yaml
  # Injected automatically by Istio; no manual YAML needed
  containers:
  - name: app
    image: order-service:v1
  
  # istio-proxy sidecar (Envoy):
  # - mTLS between services
  # - Traffic routing (canary, retry, circuit breaking)
  # - Metrics collection (request rate, latency)
  # - Distributed tracing (propagates headers)
  
  # Application is unaware; all HTTP traffic goes through the proxy
```

## Ambassador Pattern

### Database Proxy Ambassador

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: order-service:v1
    env:
    # App connects to localhost, not the real database
    - name: DB_HOST
      value: "localhost"
    - name: DB_PORT
      value: "5432"
  
  # Ambassador: handles connection pooling, TLS, failover
  - name: db-proxy
    image: pgbouncer/pgbouncer:latest
    ports:
    - containerPort: 5432
```

The app connects to `localhost:5432`. The ambassador transparently routes to the real database with connection pooling, TLS, and failover.

### Service Discovery Ambassador

```yaml
  containers:
  - name: app
    image: legacy-app:v1
    # Old app doesn't know about service discovery
    # It makes HTTP calls to hardcoded hostnames
  
  - name: envoy-proxy
    image: envoyproxy/envoy:v1.28
    # Envoy intercepts all outbound traffic
    # Resolves Kubernetes service names to actual pod IPs
    # Handles retries, circuit breaking, mTLS
    # The legacy app gets cloud-native networking without code changes
```

## When to Use Each

| Pattern | Use Case | Example |
|---------|----------|---------|
| Sidecar | Extend app functionality | Log shipper, monitoring agent, config reloader |
| Ambassador | Proxy network traffic | DB proxy, service mesh proxy, API gateway per pod |
| Adapter | Normalize output for external systems | Convert app metrics format to Prometheus format |

## Benefits

- **Separation of concerns**: Observability logic lives in sidecar, not in app code
- **Technology heterogeneity**: Sidecar can be written in any language; app is unaffected
- **Atomic deployment**: Sidecar and app are deployed, scaled, and destroyed together
- **Reuse**: Same logging sidecar across all services; same Envoy proxy for all networking

Sidecar and Ambassador are the fundamental patterns that make service meshes (Istio, Linkerd, Consul) work. The mesh is essentially a fleet of coordinated sidecar proxies providing networking, security, and observability without application changes.
