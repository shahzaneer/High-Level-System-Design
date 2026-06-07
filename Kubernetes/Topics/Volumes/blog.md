# Volumes

Volumes provide persistent or temporary storage for pods. Kubernetes supports many volume types: emptyDir (temporary, pod lifecycle), hostPath (node filesystem), persistentVolumeClaim (persistent, cross-pod), configMap/secret (configuration), projected (combined sources), and CSI (any storage system).

## Common Volume Types

### emptyDir (Temporary Pod Storage)

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir:
      medium: Memory     # tmpfs (RAM-backed, fast but consumes memory)
      sizeLimit: "256Mi"
```
emptyDir data is deleted when pod is removed. Use for: scratch space, cache, temporary files.

### hostPath (Node Filesystem)

```yaml
volumes:
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
    type: Socket
- name: node-logs
  hostPath:
    path: /var/log
    type: Directory
```
HostPath mounts directories from the NODE. POD MUST tolerate being evicted if node dies. Use sparingly (DaemonSets for monitoring/logging agents). NEVER for application data.

### PersistentVolumeClaim

```yaml
volumes:
- name: pgdata
  persistentVolumeClaim:
    claimName: order-data    # References PVC created separately
```
The standard way to get persistent storage. PVC → StorageClass → CSI driver → cloud volume.

### CSI (Container Storage Interface)

```yaml
volumes:
- name: secrets
  csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: aws-secrets
```
CSI volumes mount anything the CSI driver supports: cloud secrets, external filesystems, etc.

### Projected Volumes (Combine Multiple Sources)

```yaml
volumes:
- name: config
  projected:
    sources:
    - secret:
        name: db-credentials
        items:
        - key: username
          path: db-username
    - configMap:
        name: app-config
    - downwardAPI:
        items:
        - path: "pod-info"
          fieldRef:
            fieldPath: metadata.name
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
        audience: api
```

## Volume Mount Options

```yaml
volumeMounts:
- name: data
  mountPath: /data
  readOnly: false
  subPath: config.json         # Mount single file, not whole volume
  subPathExpr: $(POD_NAME)      # Dynamic subpath based on env var
  mountPropagation: HostToContainer
```

## PVC as Volume in Pod

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: postgres
    image: postgres:16
    volumeMounts:
    - name: pgdata
      mountPath: /var/lib/postgresql/data
      subPath: pg16           # Isolate data within PVC
  volumes:
  - name: pgdata
    persistentVolumeClaim:
      claimName: order-db-pvc
```

## Volume Lifecycle

| Volume Type | Survives Pod Restart? | Survives Pod Deletion? | Survives Node Failure? |
|-----------|---------------------|----------------------|----------------------|
| emptyDir | Yes | No | No |
| hostPath | Yes | Yes | No (data on that node lost) |
| PVC (EBS) | Yes | Yes | Yes (volume reattached) |
| PVC (EFS) | Yes | Yes | Yes (network filesystem) |
| ConfigMap/Secret | Yes | No (pod-specific) | N/A |

## Imperative

```bash
# No imperative for volumes—always declarative in pod spec

# Check volume mounts
kubectl describe pod <pod> | grep -A10 "Mounts:"
kubectl exec <pod> -- df -h
```
