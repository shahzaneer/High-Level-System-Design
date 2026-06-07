# Troubleshooting: Storage Issues

Storage issues in Kubernetes involve PVs (PersistentVolumes), PVCs (PersistentVolumeClaims), StorageClasses, and CSI drivers. The most common problem: PVC stuck in Pending state.

## PVC Pending

```bash
kubectl get pvc -n production
# NAME        STATUS    VOLUME   CAPACITY   STORAGECLASS
# order-data  Pending                      fast-ssd

kubectl describe pvc order-data
# Look at Events section:
# "waiting for a volume to be created"
```

Common causes and fixes:

**1. No default StorageClass**
```bash
kubectl get storageclass
# If (default) not shown on any SC → no default provisioner

# Fix: set default StorageClass
kubectl patch storageclass gp3 -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**2. StorageClass doesn't exist**
```yaml
# PVC requests non-existent StorageClass
spec:
  storageClassName: fast-ssd   # Does this exist?
```
```bash
kubectl get storageclass fast-ssd  # Check
```

**3. CSI driver not installed**
```bash
kubectl get pods -n kube-system | grep csi
# EBS: ebs-csi-controller, ebs-csi-node
# EFS: efs-csi-controller, efs-csi-node
# No CSI pods → driver not installed
```

## PVC Bound but Pod Can't Mount

```bash
kubectl describe pod <pod> | grep -A10 "Events:"
# "Unable to attach or mount volumes"
# "Multi-Attach error for volume" (RWO volume on different node)
```

Causes:
- **Multi-Attach error**: RWO volume attached to different node. Old pod must fully terminate.
- **Node doesn't support volume type**: EBS only available on EC2 instances.
- **Volume in wrong AZ**: Pod scheduled in us-east-1a but EBS volume in us-east-1b.

## Volume Not Expanding

```bash
# Enabled in StorageClass?
kubectl get sc fast-ssd -o yaml | grep allowVolumeExpansion

# Expand PVC
kubectl patch pvc order-data -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Check if resize completed
kubectl get pvc order-data
kubectl describe pvc order-data | grep -A5 "Conditions"
# "Waiting for user to (re-)start a pod to finish file system resize"
# → Restart pod for filesystem expansion
kubectl rollout restart deployment order-service
```

## Stuck Terminating PVC/PV

```bash
# PVC stuck in Terminating (has finalizers or pod still using it)
kubectl patch pvc order-data -p '{"metadata":{"finalizers":[]}}' --type=merge

# PV stuck (reclaim policy = Retain, not cleaned up)
kubectl patch pv pvc-abc123 -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
kubectl delete pv pvc-abc123
```

## Snapshot Issues

```bash
# Check VolumeSnapshotClass exists
kubectl get volumesnapshotclass

# Check VolumeSnapshot status
kubectl describe volumesnapshot order-data-snap
# "Waiting for a snapshot to be created" → CSI snapshotter issue
```

## CSI Driver Health

```bash
kubectl get pods -n kube-system | grep csi
# csi-controller should be Running
# csi-node daemonset should be Running on all nodes

kubectl logs -n kube-system <csi-controller-pod>
kubectl logs -n kube-system <csi-node-pod>
```

## Disk Space (Inside Container)

```bash
kubectl exec <pod> -- df -h
kubectl exec <pod> -- du -sh /var/lib/data/*

# If readOnlyRootFilesystem: true, /tmp needs an emptyDir volume
# Or pod can't write temporary files
```

## Diagnostic Commands

```bash
kubectl get pv,pvc -n production
kubectl get storageclass
kubectl get volumesnapshot,volumesnapshotcontent
kubectl describe pv <pv-name>  # Check reclaim policy, status
kubectl get events --field-selector reason=FailedAttachVolume
kubectl get events --field-selector reason=FailedMount
```
