# PersistentVolumes & PersistentVolumeClaims

PVs and PVCs decouple storage provisioning from storage consumption. A PV is a piece of storage in the cluster (provisioned by an admin or dynamically via StorageClass). A PVC is a request for storage by a user (similar to a Pod requesting CPU/memory). Pods reference PVCs, not PVs directly.

This separation enables: dynamic provisioning (just ask for storage; the StorageClass handles the rest), portability (same PVC works across cloud and on-prem), and lifecycle independence (PVC outlives Pods).

## Imperative (kubectl)

```bash
# Create a PVC (imperative)
kubectl create pvc app-data \
  --storage-class=gp3 \
  --access-mode=ReadWriteOnce \
  --request-size=10Gi

# No imperative way to create PVs (admin-only, usually dynamic)
```

## Declarative (YAML)

```yaml
# StorageClass (defines HOW storage is provisioned)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com         # AWS EBS
parameters:
  type: gp3
  iopsPerGB: "3000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer  # Wait until pod scheduled
allowVolumeExpansion: true
reclaimPolicy: Delete                   # or Retain
---
# PersistentVolumeClaim (requests storage)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: order-data
spec:
  storageClassName: fast-ssd
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
---
# Static PV (admin-provisioned; rarely used with dynamic provisioning)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 500Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.1.100
    path: /exports/data
```

## Using PVC in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-db
spec:
  containers:
  - name: postgres
    image: postgres:16
    volumeMounts:
    - name: pgdata
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: pgdata
    persistentVolumeClaim:
      claimName: order-data
```

## Access Modes

| Mode | Abbreviation | Meaning |
|------|-------------|---------|
| ReadWriteOnce | RWO | Single node read/write (EBS, most common) |
| ReadOnlyMany | ROX | Many nodes read-only |
| ReadWriteMany | RWX | Many nodes read/write (EFS, NFS, Azure Files) |

## CSI Drivers (Container Storage Interface)

```bash
# Cloud-specific storage provisioners:
# AWS: ebs.csi.aws.com, efs.csi.aws.com
# GCP: pd.csi.storage.gke.io
# Azure: disk.csi.azure.com, file.csi.azure.com

# Install EBS CSI driver on EKS
aws eks create-addon --cluster-name prod --addon-name aws-ebs-csi-driver
```

## Dynamic Provisioning Flow

```
User creates PVC → StorageClass provisioner creates PV → PV bound to PVC → Pod mounts PVC
```

## Volume Expansion

```yaml
# Enable in StorageClass
allowVolumeExpansion: true

# Expand PVC (edit request size directly)
kubectl edit pvc order-data  # Change spec.resources.requests.storage
# Or patch:
kubectl patch pvc order-data -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

## Reclaim Policies

| Policy | Behavior |
|--------|----------|
| Delete | PV deleted when PVC deleted (default, and safe with cloud storage) |
| Retain | PV kept after PVC deletion (manual cleanup needed; data survives) |
| Recycle | Deprecated; basic scrub (rm -rf) and reuse |

## Snapshot and Restore

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: order-data-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: order-data
---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: order-data-restored
spec:
  dataSource:
    name: order-data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

## Imperative vs Declarative

PVCs can be created imperatively for quick tests. Production always uses declarative YAML with StorageClasses for dynamic provisioning. Never create static PVs manually—let the CSI driver handle it.
