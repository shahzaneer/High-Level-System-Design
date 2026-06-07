# ConfigMaps & Secrets

ConfigMaps store non-sensitive configuration data. Secrets store sensitive data (passwords, tokens, keys). Both decouple configuration from container images, enabling the same image to run in different environments with different configurations.

Key difference: Secrets are base64-encoded (NOT encrypted by default—use encryption at rest), support size limit of 1MB, and integrate with specialized secret stores (AWS Secrets Manager, Azure Key Vault via CSI driver).

## Imperative (kubectl)

```bash
# CONFIGMAPS
# From literal values
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=API_TIMEOUT=30

# From file
kubectl create configmap nginx-config --from-file=nginx.conf

# From directory (every file becomes a key)
kubectl create configmap app-files --from-file=./config/

# From env file
kubectl create configmap app-env --from-env-file=.env

# SECRETS
# From literal values
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123!

# From file
kubectl create secret generic api-tls \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key

# Docker registry credentials
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.com \
  --docker-username=user \
  --docker-password=pass

# TLS secret
kubectl create secret tls api-tls \
  --cert=fullchain.pem --key=privkey.pem

# View (secrets are shown base64-decoded in describe)
kubectl describe configmap app-config
kubectl describe secret db-credentials
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

## Declarative (YAML)

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "debug"
  API_TIMEOUT: "30"
  app.properties: |
    server.port=8080
    database.pool.size=20
    cache.ttl=300

---
# Secret (values MUST be base64 encoded in YAML)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=              # echo -n 'admin' | base64
  password: U3VwZXJTZWNyZXQxMjMh  # echo -n 'SuperSecret123!' | base64

# Alternative: use stringData for plaintext (K8s encodes automatically)
stringData:
  username: admin
  password: SuperSecret123!
```

## Consuming in Pods

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    # As environment variables (whole configmap)
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-credentials
    
    # As individual environment variables
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    
    # As mounted files (best for certificates, config files)
    volumeMounts:
    - name: config
      mountPath: /etc/app/config
      readOnly: true
    - name: tls
      mountPath: /etc/ssl/certs
  
  volumes:
  - name: config
    configMap:
      name: app-config
  - name: tls
    secret:
      secretName: api-tls
```

## External Secrets (Production Pattern)

```yaml
# External Secrets Operator: sync from AWS/GCP/Azure secret stores
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: prod/database/password
      property: password
  - secretKey: username
    remoteRef:
      key: prod/database/credentials
      property: username
```

## Security Best Practices

1. **Encrypt Secrets at Rest**: Enable etcd encryption in K8s API server
2. **Never commit plaintext Secrets to Git**: Use SealedSecrets, SOPS, or External Secrets
3. **Use RBAC to restrict Secret access**: Not every pod should read every secret
4. **Rotate Secrets regularly**: Integrate with Vault or cloud KMS
5. **Mount as files instead of env vars**: Env vars leak in crash dumps, debug endpoints

## Imperative vs Declarative

Imperative is fine for quick local development. Production always uses declarative (Git versioned, reviewable, auditable). External Secrets Operator pattern is recommended for any cloud deployment over baking secrets into K8s manifests.
