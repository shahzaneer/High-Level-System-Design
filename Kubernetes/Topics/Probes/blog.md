# Probes (Liveness, Readiness, Startup)

Probes are health checks that Kubernetes uses to determine pod health. Three types: liveness (should I restart this container?), readiness (should I send traffic to this pod?), startup (is the application done starting up?). Probes can use exec commands, HTTP GET, TCP socket, or gRPC checks.

## Probe Types

| Probe | Question | On Failure | On Success |
|-------|----------|-----------|------------|
| Liveness | Is the app alive? | Restart container | Nothing |
| Readiness | Is the app ready for traffic? | Remove from Service endpoints | Add to Service endpoints |
| Startup | Has the app finished starting? | Restart container | Start liveness/readiness checks |

## Declarative (YAML)

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: order-service:v1
    
    # Startup probe: protects slow-starting apps
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 30    # 30 × 10s = up to 5 minutes to start
      successThreshold: 1
    
    # Liveness: is the process healthy?
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0  # Don't start until Startup probe succeeds
      periodSeconds: 15
      timeoutSeconds: 5
      failureThreshold: 3     # 3 failures → restart
      successThreshold: 1
    
    # Readiness: can it serve traffic?
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 2     # 2 failures → remove from service
      successThreshold: 1
```

## Probe Handlers

```yaml
# HTTP GET probe
httpGet:
  path: /health
  port: 8080
  httpHeaders:
  - name: X-Custom-Header
    value: probe
  
# TCP socket probe
tcpSocket:
  port: 8080

# Exec probe (run command inside container)
exec:
  command:
  - /bin/sh
  - -c
  - |
    pg_isready -h localhost -p 5432

# gRPC probe (Kubernetes 1.24+)
grpc:
  port: 50051
  service: grpc.health.v1.Health
```

## Implementation Endpoints

```python
# Flask health endpoints
@app.route('/healthz')
def healthz():
    # Liveness: just confirm process is alive
    return '', 200

@app.route('/ready')
def ready():
    # Readiness: check ALL critical dependencies
    checks = {
        'database': check_db_connection(),
        'cache': check_redis(),
        'message_queue': check_kafka(),
    }
    if all(checks.values()):
        return '', 200
    return jsonify(checks), 503

@app.route('/startup')
def startup():
    # Startup: confirm application initialization is complete
    if app_init_complete:
        return '', 200
    return 'Initializing', 503
```

## Imperative

```bash
# No imperative command for probes—always declarative in Pod spec

# Check probe status
kubectl describe pod order-service | grep -A 20 "Conditions:"
kubectl get events --field-selector involvedObject.name=order-service
```

## Best Practices

1. **Always have probes in production**: No probe = K8s doesn't know if app is healthy
2. **Liveness checks minimal work**: Just verify the process responds. Don't check DB connectivity in liveness (DB outage will restart ALL pods—cascading failure!)
3. **Readiness checks all dependencies**: DB, cache, external APIs that are REQUIRED
4. **Use startup probes for slow-starting apps**: Prevents liveness killing pods during legitimate startup
5. **Set failureThreshold high enough**: 1 failure could be a transient blip. 3 consecutive failures is a real problem.

## Anti-Patterns

```yaml
# BAD: Liveness checks database connectivity
livenessProbe:
  exec:
    command: ["check_db_connection.sh"]
# DB outage → all pods restart → restart storm → makes DB worse

# BAD: No probes at all
# K8s can't tell if pod is healthy → sends traffic to crashed pods
```
