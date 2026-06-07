# Init Containers

Init Containers run before application containers start, in sequence, each completing successfully before the next begins. They perform setup tasks that must complete before the main application runs: database migrations, waiting for dependencies, file permission fixes, secret fetching. Init containers use the same image as the app container or specialized utility images.

## Declarative (YAML)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-service
spec:
  initContainers:
  # 1. Wait for database to be available
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      until nslookup postgres.production.svc.cluster.local; do
        echo "Waiting for database DNS..."
        sleep 2
      done
  
  # 2. Run database migrations
  - name: db-migrate
    image: order-service:v1.2.0
    command: ['python', 'manage.py', 'migrate']
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url
  
  # 3. Pre-populate cache
  - name: warm-cache
    image: order-service:v1.2.0
    command: ['python', 'scripts/warm_cache.py']
  
  # Main application containers  
  containers:
  - name: app
    image: order-service:v1.2.0
    ports:
    - containerPort: 8080
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url
```

## Imperative

```bash
# No imperative command for init containers—always declarative

# View init container logs
kubectl logs order-service -c wait-for-db     # Init container log
kubectl logs order-service -c app              # Main container log
```

## Common Patterns

```yaml
# Pattern 1: Wait for service
initContainers:
- name: wait-for-redis
  image: busybox
  command: ['sh', '-c', 'until nc -z redis 6379; do sleep 1; done']

# Pattern 2: Set permissions
initContainers:
- name: fix-permissions
  image: busybox
  command: ['sh', '-c', 'chown -R 1000:1000 /data']
  volumeMounts:
  - name: data
    mountPath: /data

# Pattern 3: Fetch secrets from Vault
initContainers:
- name: vault-agent
  image: hashicorp/vault:latest
  command: ['vault', 'agent', '-config=/vault/config.hcl']
  volumeMounts:
  - name: secrets
    mountPath: /vault/secrets

# Pattern 4: Schema migration
initContainers:
- name: migrate
  image: flyway/flyway:10
  command: ['flyway', 'migrate']
  env:
  - name: FLYWAY_URL
    value: jdbc:postgresql://db:5432/orders
```

## Init vs Sidecar

| Feature | Init Container | Sidecar Container |
|---------|---------------|-------------------|
| Runs | Before main app | Alongside main app |
| Lifecycle | Completes and exits | Runs entire pod lifetime |
| Use case | Setup, migration, waiting | Logging, proxying, monitoring |
| Restart | Rerun if pod restarts | Continuous operation |

## Best Practices

1. **Init containers run to completion**: They must exit successfully. If they fail, K8s retries
2. **Use same image as app**: Ensures same dependencies for migration scripts
3. **Keep init containers fast**: Pod stays in Init state until all init containers complete
4. **Don't use for runtime tasks**: Init containers stop after completion; use sidecars for ongoing tasks
5. **Security**: Init containers run with same security context. Grant only needed permissions.
