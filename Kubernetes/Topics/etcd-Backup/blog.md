# etcd Backup & Restore

etcd is the brain of Kubernetes—the distributed key-value store that holds ALL cluster state (pods, services, secrets, RBAC, everything). If etcd data is lost, the cluster is effectively destroyed. etcd backup and restore is the single most critical operational procedure for any self-managed Kubernetes cluster. Managed K8s (EKS, GKE, AKS) handle this automatically.

## Backup

```bash
# Take a snapshot (run on a control plane node)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table

# Encrypt the backup
gpg --encrypt --recipient security@company.com /backup/etcd-snapshot.db

# Copy off-node immediately
aws s3 cp /backup/etcd-snapshot.db.gpg s3://backups/etcd/
```

### Automated Backup CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
          - key: node-role.kubernetes.io/control-plane
            effect: NoSchedule
          containers:
          - name: backup
            image: bitnami/etcd:3.5
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M).db \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
              aws s3 cp /backup/etcd-*.db s3://backups/etcd/
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup
            hostPath:
              path: /var/backups/etcd
```

## Restore

```bash
# WARNING: This is destructive! Only for catastrophic recovery.

# 1. Stop kube-apiserver on all control plane nodes
# 2. Restore snapshot on the first control plane node
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=cp-node-1 \
  --initial-cluster=cp-node-1=https://cp1:2380,cp-node-2=https://cp2:2380,cp-node-3=https://cp3:2380 \
  --initial-advertise-peer-urls=https://cp1:2380

# 3. Update etcd manifest to use restored data directory
# 4. Start etcd, verify, then restart kube-apiserver
# 5. On other control plane nodes: remove old data, let etcd re-sync from leader
```

## Validation

```bash
# After backup, verify etcd cluster health
ETCDCTL_API=3 etcdctl endpoint health --cluster

# Check for defragmentation need (fragmentation > 50% = should defrag)
ETCDCTL_API=3 etcdctl endpoint status --write-out=table
ETCDCTL_API=3 etcdctl defrag --cluster
```

## Managed K8s Equivalents

- **EKS**: etcd is fully managed by AWS. Backup via Velero for application state; etcd itself is AWS's responsibility.
- **GKE**: Same—managed control plane. Use Backup for GKE for application + config state.
- **AKS**: Managed control plane. Use Velero or Azure Backup for AKS.

Self-managed clusters (kubeadm, Rancher RKE2) MUST have etcd backup configured and tested. An untested etcd backup is not a backup.
