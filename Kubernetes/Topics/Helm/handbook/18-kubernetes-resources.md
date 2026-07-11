# Chapter 18: Helm with Kubernetes Resources

Helm's primary job is rendering YAML manifests that Kubernetes understands. Every resource type has its own lifecycle, immutability rules, and upgrade behavior — and Helm interacts with each differently. This chapter documents Helm's relationship with every major Kubernetes resource, providing complete working templates, values examples, and production guidance.

---

## 18.1 Deployments

The most common workload resource. Deployments manage ReplicaSets which manage Pods.

### Overview

Helm treats Deployments as standard templates. However, `spec.selector` is immutable after creation, so Helm cannot patch it during upgrades. `spec.replicas`, `spec.template`, and `spec.strategy` are all mutable.

### Template Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  revisionHistoryLimit: {{ .Values.revisionHistoryLimit }}
  strategy:
    {{- toYaml .Values.strategy | nindent 4 }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "mychart.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### Values Example

```yaml
replicaCount: 3
revisionHistoryLimit: 10
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25"
service:
  port: 80
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

### Management Considerations

**Updating Replicas:** Changing `replicaCount` triggers a Helm upgrade that patches the Deployment. No new ReplicaSet is created.

**Updating Pod Template:** Changing container image or env vars triggers a rolling update via the ReplicaSet controller. Helm issues a PATCH; the controller handles the rollout.

**Selector Immutability:** If `.Values.selectorLabels` changes, `helm upgrade` will fail with:
```
spec.selector: Invalid value: ... field is immutable
```
Mitigation: delete and recreate (`helm upgrade --force`) or perform a blue/green deployment.

**`--force` During Upgrade:** `helm upgrade --force` deletes and recreates the Deployment entirely. All existing Pods are terminated immediately. Proceed with caution in production.

**Rollback Behavior:** `helm rollback` reverts to the previous release's manifest. The Deployment rolls back to the prior ReplicaSet template, so Pods also revert.

### Common Mistakes

1. Changing `matchLabels` in `selector` — immutable field, causes upgrade failure.
2. Forgetting `revisionHistoryLimit` — default is 10; large clusters may accumulate stale ReplicaSets.
3. Using `Recreate` strategy without considering downtime.

### Production Notes

- Always set `spec.revisionHistoryLimit` explicitly (5–10 is typical).
- Use `RollingUpdate` with `maxUnavailable: 1` for stateful apps that cannot tolerate multiple-down scenarios.
- The `checksum/config` annotation pattern (shown above) forces a rolling Pod restart when ConfigMap changes — see Section 18.7.

---

## 18.2 DaemonSets

DaemonSets ensure one Pod runs on every (or selected) node.

### Overview

DaemonSets support rolling updates natively since Kubernetes 1.6. The `updateStrategy` field controls how Pods are replaced node-by-node. Helm creates and patches DaemonSets identically to Deployments.

### Template Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  updateStrategy:
    type: {{ .Values.daemonset.updateStrategy.type }}
    {{- if eq .Values.daemonset.updateStrategy.type "RollingUpdate" }}
    rollingUpdate:
      maxUnavailable: {{ .Values.daemonset.updateStrategy.maxUnavailable }}
      maxSurge: {{ .Values.daemonset.updateStrategy.maxSurge }}
    {{- end }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.daemonset.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.daemonset.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      hostNetwork: {{ .Values.daemonset.hostNetwork }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          securityContext:
            privileged: {{ .Values.daemonset.privileged }}
```

### Values Example

```yaml
daemonset:
  updateStrategy:
    type: RollingUpdate
    maxUnavailable: 1
    maxSurge: 0
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule
  hostNetwork: false
  privileged: false
```

### Management Considerations

**Node-Specific Configuration:** Use `nodeSelector`, `tolerations`, and `affinity` in values. Different releases of the same chart can target different node pools.

**Update Strategy:** `OnDelete` requires manual Pod deletion to apply changes. `RollingUpdate` is preferred for automated rollouts.

**Tolerations:** Must be set to schedule on tainted nodes (e.g., control-plane nodes for monitoring agents).

**`nodeSelector` Changes:** Changing nodeSelector during an upgrade may leave Pods running on nodes that no longer match — the DaemonSet controller must reconcile.

### Common Mistakes

1. Using `maxUnavailable: 100%` on a critical node-level agent (e.g., CNI plugin) — can cause cluster-wide outage.
2. Forgetting tolerations when targeting control-plane nodes.
3. Not setting `hostNetwork: true` for network monitoring agents that need it.

### Production Notes

- For node-level monitoring (Prometheus node-exporter, Fluentd, etc.), set `updateStrategy.rollingUpdate.maxUnavailable: 1` and add a `PodDisruptionBudget`.
- Prefer `maxSurge: 0` on clusters with limited capacity.
- Use `hostPID: true` and `hostNetwork: true` only when absolutely necessary — they bypass namespace isolation.

---

## 18.3 StatefulSets

StatefulSets manage stateful applications with stable network identities and persistent storage.

### Overview

StatefulSets differ from Deployments in three critical ways for Helm: (1) Pods have ordinal, stable identities (`myapp-0`, `myapp-1`, ...), (2) PVCs are created via `volumeClaimTemplates` and survive Pod restarts, and (3) updates proceed in reverse ordinal order by default.

### Template Example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  serviceName: {{ include "mychart.fullname" . }}-headless
  replicas: {{ .Values.replicaCount }}
  podManagementPolicy: {{ .Values.statefulset.podManagementPolicy }}
  updateStrategy:
    type: RollingUpdate
    {{- if .Values.statefulset.updatePartition }}
    rollingUpdate:
      partition: {{ .Values.statefulset.updatePartition }}
    {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: {{ .Values.persistence.storageClass }}
        resources:
          requests:
            storage: {{ .Values.persistence.size }}
```

### Values Example

```yaml
replicaCount: 3
statefulset:
  podManagementPolicy: OrderedReady
  updatePartition: 0
persistence:
  storageClass: "gp3"
  size: 10Gi
```

### Management Considerations

**Headless Service:** A headless service (`clusterIP: None`) must exist for the StatefulSet to provide stable DNS names (`myapp-0.myapp-headless.namespace.svc.cluster.local`). The template must also provide this Service.

**`volumeClaimTemplates`:** PVCs are named `data-myapp-0`, `data-myapp-1`, etc. These PVCs are NOT deleted when the StatefulSet is deleted (or when Helm uninstalls the release). They must be manually deleted.

**Ordered Updates:** By default (`OrderedReady`), Pods update from highest ordinal to lowest (`myapp-2` first, then `myapp-1`, then `myapp-0`). With `Parallel` management, all Pods update simultaneously.

**Partition Updates:** Setting `partition: N` means only Pods with ordinal >= N are updated. This enables canary-style StatefulSet updates. Reset `partition: 0` to complete the rollout.

**PVC Management During Rollback:** `helm rollback` reverts the StatefulSet manifest but does NOT touch PVCs. If a rollback involves a downgrade that is incompatible with existing data, manual intervention is required.

### Common Mistakes

1. Forgetting to create the headless Service.
2. Expecting `helm uninstall` to delete PVCs — it doesn't. Use `kubectl delete pvc -l app=myapp` post-uninstall.
3. Changing `volumeClaimTemplates` after creation — PVC templates are immutable once applied.
4. Not setting `podManagementPolicy: Parallel` for stateless-like workloads on StatefulSet.

### Production Notes

- Use `partition` for safe rolling upgrades of databases (e.g., upgrade one replica first, validate, then set `partition: 0`).
- Always define `podDisruptionBudget` for StatefulSets to ensure quorum is maintained.
- Back up PVCs before Helm upgrades that change application versions with potential data migrations.

---

## 18.4 CronJobs

CronJobs create Jobs on a time-based schedule.

### Overview

Helm creates and patches CronJobs normally. The `schedule` field uses standard crontab format (5 fields). CronJob ownership of Jobs is managed by the CronJob controller — Helm does not track those Jobs.

### Template Example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  schedule: {{ .Values.cronjob.schedule | quote }}
  concurrencyPolicy: {{ .Values.cronjob.concurrencyPolicy }}
  successfulJobsHistoryLimit: {{ .Values.cronjob.successfulJobsHistoryLimit }}
  failedJobsHistoryLimit: {{ .Values.cronjob.failedJobsHistoryLimit }}
  startingDeadlineSeconds: {{ .Values.cronjob.startingDeadlineSeconds }}
  suspend: {{ .Values.cronjob.suspend }}
  jobTemplate:
    spec:
      backoffLimit: {{ .Values.cronjob.backoffLimit }}
      ttlSecondsAfterFinished: {{ .Values.cronjob.ttlSecondsAfterFinished }}
      template:
        spec:
          restartPolicy: {{ .Values.cronjob.restartPolicy }}
          containers:
            - name: {{ .Chart.Name }}
              image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
              command: {{ .Values.cronjob.command }}
```

### Values Example

```yaml
cronjob:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 300
  suspend: false
  backoffLimit: 3
  ttlSecondsAfterFinished: 86400
  restartPolicy: OnFailure
  command:
    - /bin/sh
    - -c
    - "echo 'Backup complete'"
```

### Management Considerations

**Schedule Format:** Kubernetes uses 5-field crontab, not 6-field (no seconds field). `"*/5 * * * *"` = every 5 minutes. Invalid schedules prevent the CronJob from being applied.

**`concurrencyPolicy`:**
- `Allow` (default): Multiple Jobs may run simultaneously.
- `Forbid`: Skip the next run if the previous Job is still running.
- `Replace`: Terminate the running Job and start a new one.

**`startingDeadlineSeconds`:** If a Job misses its scheduled time by more than this value, it is skipped. Useful for clusters that may be down during maintenance windows.

**How Helm Handles CronJob Updates:** Helm patches the CronJob resource. Existing Jobs spawned by the CronJob remain unaffected. Future invocations use the new spec.

**`suspend`:** Set to `true` to pause scheduling without deleting the CronJob. Helm upgrade can toggle this.

### Common Mistakes

1. Using 6-field cron syntax (with seconds) — not supported by Kubernetes.
2. Setting `concurrencyPolicy: Forbid` but having long-running Jobs that block the schedule.
3. Too-low `successfulJobsHistoryLimit` — losing job completion history needed for audits.
4. Not setting `ttlSecondsAfterFinished` — accumulating Job and Pod objects indefinitely.

### Production Notes

- Always set `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` to prevent stale Job accumulation (default is 3 and 1).
- Use `ttlSecondsAfterFinished` to auto-clean completed Jobs; recommended value: `86400` (24 hours) for audit trails.
- For idempotent jobs, use `concurrencyPolicy: Replace`; for mutually exclusive jobs, use `Forbid`.
- Set `startingDeadlineSeconds` high enough to tolerate control-plane outages (>=300 seconds).

---

## 18.5 Jobs

Jobs create one or more Pods and ensure a specified number complete successfully.

### Overview

Two categories: (1) **Hook Jobs** — defined with `"helm.sh/hook": job` annotation, used for pre-install/post-upgrade hooks; (2) **Application Jobs** — standalone Jobs managed as part of a release (e.g., database migrations). Helm handles them differently.

### Template Example (Hook Job for Migrations)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-migration
  annotations:
    "helm.sh/hook": post-upgrade,post-install
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  backoffLimit: {{ .Values.job.backoffLimit }}
  ttlSecondsAfterFinished: {{ .Values.job.ttlSecondsAfterFinished }}
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: {{ include "mychart.serviceAccountName" . }}
      containers:
        - name: migration
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["npm", "run", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "mychart.fullname" . }}-db
                  key: url
```

### Values Example

```yaml
job:
  backoffLimit: 3
  ttlSecondsAfterFinished: 600
```

### Management Considerations

**Hook Jobs vs Application Jobs:**
- **Hook Jobs:** Use `helm.sh/hook` annotation. Helm tracks hook execution separately; hooks do not block the release. `--wait` waits for hooks to complete. Hook deletion is controlled by `helm.sh/hook-delete-policy`.
- **Application Jobs:** No hook annotation. They are upgraded/rolled back as part of the release, like any other resource.

**`ttlSecondsAfterFinished`:** Sets TTL for automatic cleanup after Job completion. `null` means no automatic cleanup.

**`backoffLimit`:** Number of retries before marking the Job as failed. Default is 6.

**Helm Upgrade Behavior:** During `helm upgrade`, existing Job objects from the previous release are patched. Since `spec.selector` and `spec.template` of Jobs may be immutable depending on the Job's state, upgrades can fail. For hook Jobs, use `before-hook-creation` delete policy to avoid this.

### Common Mistakes

1. Not setting `ttlSecondsAfterFinished` — completed Jobs and their Pods consume etcd storage forever.
2. Forgetting `"helm.sh/hook-delete-policy": "hook-succeeded"` on post-upgrade hooks — subsequent upgrades fail because the hook Job already exists and is immutable.
3. Using `restartPolicy: Always` — not allowed on Jobs (must be `Never` or `OnFailure`).

### Production Notes

- Hook Jobs should always include `helm.sh/hook-delete-policy` with at least `before-hook-creation`.
- For database migrations, use `post-upgrade` hooks with `--wait` to ensure migrations complete before new Pods serve traffic.
- For one-time setup Jobs, consider using `post-install` hooks with `hook-succeeded` delete policy.
- Set `backoffLimit` low (3–5) for Jobs that should fail fast.

---

## 18.6 Secrets

Secrets store sensitive data such as passwords, tokens, and keys.

### Overview

Helm templates must NEVER contain hardcoded secret values in `values.yaml` or templates. Values are stored in release Secrets (base64-encoded) and can be retrieved by anyone with release access. All secrets should be pulled from external sources or managed outside Helm.

### Template Example (External Reference Pattern)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
type: Opaque
data:
  {{- if .Values.secrets.databaseUrl }}
  DATABASE_URL: {{ .Values.secrets.databaseUrl | b64enc | quote }}
  {{- end }}
  {{- if .Values.secrets.apiKey }}
  API_KEY: {{ .Values.secrets.apiKey | b64enc | quote }}
  {{- end }}
```

### Values Example (NEVER commit actual secrets)

```yaml
secrets:
  databaseUrl: ""
  apiKey: ""
```

### Secure Pattern: External Secrets via values override

```bash
helm install myapp ./chart \
  --set secrets.databaseUrl="$DB_URL" \
  --set secrets.apiKey="$API_KEY"
```

### External Secrets Operator Pattern (recommended)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: {{ .Values.externalSecrets.storeName }}
    kind: {{ .Values.externalSecrets.storeKind }}
  target:
    name: {{ include "mychart.fullname" . }}
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: {{ .Values.externalSecrets.dbSecretPath }}
        property: url
```

### Management Considerations

**NEVER Hardcode in Templates:** Helm stores all values in a Secret (Release Secret) in base64. Anyone with `kubectl get secret` access to the release namespace can decode them.

**Pulling from External Sources:** Use one of:
- External Secrets Operator (ESO) — syncs secrets from AWS Secrets Manager, GCP Secret Manager, Azure Key Vault.
- Sealed Secrets — encrypted secrets safe for Git.
- Vault (with `vault` CLI or CSI driver).
- SOPS + Helm Secrets plugin.
- `--set` or `-f` with values from CI/CD secret stores.

**Immutable Secrets:** Since Kubernetes 1.21, Secrets can be set as `immutable: true`. Helm cannot update an immutable Secret; upgrades fail. Use unique names (e.g., append a hash) or avoid immutability on Helm-managed Secrets.

**`stringData` vs `data`:**
- `data`: values must be base64-encoded in the manifest.
- `stringData`: values are plaintext; Kubernetes encodes them automatically. Helm templates can use `stringData` for readability, but the values are still stored in the Release Secret.

### Common Mistakes

1. Hardcoding secrets in `values.yaml` and committing to Git.
2. Using `immutable: true` on Helm-managed Secrets — breaks upgrades.
3. Forgetting that Helm's `--set` passes values via command line, which may be logged to shell history.
4. Assuming Helm encryption protects values — the Release Secret stores plain base64.

### Production Notes

- Use External Secrets Operator or Sealed Secrets as the primary secret management strategy.
- For Airflow/Argo-style configs, use `stringData` for readability but templatize external secret references.
- Rotate credentials outside Helm; use a secret operator to sync updates into the cluster.
- Audit release Secrets with `helm get values <release> --all` to ensure no secrets leaked.

---

## 18.7 ConfigMaps

ConfigMaps store non-sensitive configuration data.

### Overview

ConfigMaps are the primary way to inject configuration into Pods. Helm templates can generate ConfigMaps from `values.yaml`. The key production pattern is the **checksum annotation** — forcing a Pod restart when ConfigMaps change.

### Template Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}-config
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
data:
  app.conf: |
    [server]
    port = {{ .Values.service.port }}
    log_level = {{ .Values.config.logLevel | default "info" }}
    max_connections = {{ .Values.config.maxConnections }}
  {{- with .Values.config.extra }}
  extra.conf: |
    {{- . | nindent 4 }}
  {{- end }}
binaryData:
  {{- if .Values.config.caBundle }}
  ca-bundle.crt: {{ .Values.config.caBundle }}
  {{- end }}
```

### Values Example

```yaml
config:
  logLevel: "debug"
  maxConnections: 100
  extra: |
    feature_flags:
      new_auth: true
      dark_mode: false
  caBundle: ""
```

### Rolling Updates on ConfigMap Change (Checksum Annotation)

In the Deployment template, add an annotation whose value is the SHA256 hash of the ConfigMap:

```yaml
template:
  metadata:
    annotations:
      checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

This ensures that any change to the rendered ConfigMap changes the Pod template, triggering a rolling update.

### Management Considerations

**Immutable ConfigMaps:** Like Secrets, ConfigMaps can be `immutable: true`. Helm cannot update them. Avoid this pattern for Helm-managed ConfigMaps, or use versioned names (e.g., `myapp-config-v{{ .Release.Revision }}`).

**`binaryData`:** Stores binary data (base64-encoded) in ConfigMap. Useful for CA bundles, license files, etc. Values are NOT further encoded — provide raw base64 in values.

**Helm Upgrade Behavior:** Helm patches the ConfigMap in place. If mounted as a volume, kubelet updates the symlink within ~60 seconds (by default). If used as `envFrom`, environment variables do NOT update without a Pod restart — use the checksum annotation.

### Common Mistakes

1. Expecting `envFrom` ConfigMap changes to propagate to running Pods without restarts.
2. Hitting the ConfigMap size limit (1 MiB). For large configs, consider splitting or using a separate volume.
3. Using `immutable: true` and then expecting Helm upgrades to work.
4. Not including the checksum annotation — Pods run with stale configuration indefinitely.

### Production Notes

- Always use the checksum annotation pattern for ConfigMaps consumed by Deployments/StatefulSets.
- For large configurations (>500KB), consider splitting into multiple ConfigMaps.
- ConfigMap mounted as a volume auto-updates on the node within a minute; set `kubelet` config `configMapAndSecretChangeDetectionStrategy: Watch` for faster propagation.
- For applications that support hot-reload, combine volume mounts with checksum annotations for a two-tier strategy.

---

## 18.8 PVC (PersistentVolumeClaim)

PVCs request storage from StorageClasses (dynamic) or bind to existing PVs (static).

### Overview

Helm creates PVCs as standard resources. **Critical point:** `helm uninstall` does NOT delete PVCs created by templates unless they were created via StatefulSet's `volumeClaimTemplates` (and even then, the PVCs survive). The `--keep-history` flag has no effect on PVCs — it only keeps release history Secrets.

### Template Example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "mychart.fullname" . }}-data
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- if .Values.persistence.keepAfterUninstall }}
  annotations:
    "helm.sh/resource-policy": keep
  {{- end }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode }}
  storageClassName: {{ .Values.persistence.storageClass }}
  resources:
    requests:
      storage: {{ .Values.persistence.size }}
```

### Values Example

```yaml
persistence:
  accessMode: ReadWriteOnce
  storageClass: "gp3"
  size: 50Gi
  keepAfterUninstall: true
```

### Management Considerations

**Retention During Uninstall:** By default, PVCs are deleted on `helm uninstall`. Use annotation `"helm.sh/resource-policy": keep` to preserve them. This is recommended for production data.

**StatefulSet PVCs:** PVCs created via `volumeClaimTemplates` are NOT managed by Helm directly — they are created by the StatefulSet controller. On `helm uninstall`, the StatefulSet is deleted but the PVCs remain. On `helm upgrade`, changes to `volumeClaimTemplates` are rejected by the Kubernetes API (immutable field).

**`storageClassName`:** Changing storageClass on an existing PVC is not allowed. You must delete and recreate the PVC.

**Resizing:** Most CSI drivers support online expansion. Update `.spec.resources.requests.storage` to a larger value. Helm applies this change, and the CSI driver handles the rest. Shrinking is generally not supported.

### Common Mistakes

1. Assuming `helm uninstall` preserves PVCs — it removes them unless `resource-policy: keep` is set.
2. Attempting to change `volumeClaimTemplates` in StatefulSet — immutable.
3. Not setting `storageClassName` — uses the default StorageClass, which may not be appropriate for production.
4. Forgetting that `ReadWriteOnce` binds to a single node — Pods on different nodes cannot share the volume.

### Production Notes

- Always use `"helm.sh/resource-policy": keep` on stateful PVCs.
- Pre-provision PVs with `persistentVolumeReclaimPolicy: Retain` for critical data.
- Set up alerts on PVC usage (`kubelet_volume_stats_used_bytes`) to detect near-full conditions before they cause outages.
- Use the `--wait` flag with Helm upgrades to ensure PVC binding completes before Pods start.

---

## 18.9 PV (PersistentVolume)

PVs are cluster-level storage resources provisioned statically or dynamically.

### Overview

PVs are cluster-scoped resources. Helm can create them, but this is uncommon — most production setups use dynamic provisioning via StorageClasses. Helm-managed PVs are useful for static provisioning with pre-created storage assets.

### Template Example (Static PV)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: {{ include "mychart.fullname" . }}-pv
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  capacity:
    storage: {{ .Values.persistence.size }}
  accessModes:
    - {{ .Values.persistence.accessMode }}
  persistentVolumeReclaimPolicy: {{ .Values.persistence.reclaimPolicy }}
  storageClassName: {{ .Values.persistence.storageClass }}
  nfs:
    server: {{ .Values.persistence.nfs.server }}
    path: {{ .Values.persistence.nfs.path }}
```

### Values Example

```yaml
persistence:
  size: 100Gi
  accessMode: ReadWriteMany
  reclaimPolicy: Retain
  storageClass: "nfs-static"
  nfs:
    server: "192.168.1.100"
    path: "/exports/data"
```

### Management Considerations

**Static vs Dynamic Provisioning:**
- **Static:** Admin creates PVs. Helm creates PVCs that bind to matching PVs. Helm template can create both PV and PVC, but the PV is cluster-scoped and may conflict with other releases.
- **Dynamic:** StorageClass + provisioner create PVs on demand. Helm creates only PVCs. This is the recommended pattern.

**Reclaim Policies:**
- `Retain`: PV is NOT deleted when PVC is deleted. Manual cleanup required.
- `Delete`: PV and underlying storage are deleted (default).
- `Recycle`: Deprecated. Wipes data and makes PV available again.

**Pre-Provisioned PVs:** If you create PVs via Helm, they must be created BEFORE the PVC or in the same release with correct ordering. Helm does not guarantee creation order within a release unless you use hooks with weights.

### Common Mistakes

1. Creating cluster-scoped PVs without RBAC to match — namespace-scoped users cannot see them.
2. Using `Delete` reclaim policy for critical data — use `Retain`.
3. PV `storageClassName` mismatch with PVC — binding fails silently; PVC stays Pending.

### Production Notes

- Prefer dynamic provisioning over Helm-managed PVs.
- If static PVs are necessary, use `persistentVolumeReclaimPolicy: Retain` and a naming convention that identifies the release.
- ClusterRoles are needed for Helm to manage PVs (cluster-scoped). Ensure the user has appropriate permissions.
- Monitor PV usage with `kube_persistentvolume_status_phase` metric.

---

## 18.10 StorageClass

StorageClass defines classes of storage with a provisioner and parameters.

### Overview

StorageClasses are cluster-scoped resources that Helm can create (typically during initial cluster bootstrapping). They define how PVCs are dynamically provisioned.

### Template Example

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: {{ .Values.storageClass.name }}
  annotations:
    storageclass.kubernetes.io/is-default-class: "{{ .Values.storageClass.isDefault }}"
provisioner: {{ .Values.storageClass.provisioner }}
parameters:
  {{- toYaml .Values.storageClass.parameters | nindent 2 }}
{{- if .Values.storageClass.reclaimPolicy }}
reclaimPolicy: {{ .Values.storageClass.reclaimPolicy }}
{{- end }}
{{- if .Values.storageClass.volumeBindingMode }}
volumeBindingMode: {{ .Values.storageClass.volumeBindingMode }}
{{- end }}
allowVolumeExpansion: {{ .Values.storageClass.allowVolumeExpansion }}
```

### Values Example

```yaml
storageClass:
  name: "gp3-encrypted"
  isDefault: false
  provisioner: ebs.csi.aws.com
  reclaimPolicy: Retain
  volumeBindingMode: WaitForFirstConsumer
  allowVolumeExpansion: true
  parameters:
    type: gp3
    encrypted: "true"
    kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/abc-123"
```

### Management Considerations

**Default StorageClass:** The annotation `storageclass.kubernetes.io/is-default-class: "true"` makes it the default. Only one default should exist per cluster. Helm managing multiple defaults causes unpredictable behavior.

**Provisioner:** Must match the CSI driver installed in the cluster (e.g., `ebs.csi.aws.com`, `pd.csi.storage.gke.io`, `file.csi.azure.com`).

**Parameters:** These are provisioner-specific. EBS supports `type`, `encrypted`, `iops`, `throughput`. GCP PD supports `type`, `replication-type`. Always consult the CSI driver documentation.

**Helm Upgrade Behavior:** StorageClasses are mutable (most fields). Changing `parameters` does NOT affect existing PVs — only new PVCs.

### Common Mistakes

1. Creating multiple default StorageClasses — causes unpredictable PVC defaulting.
2. Forgetting `allowVolumeExpansion: true` — cannot resize PVCs later without recreating.
3. Using `volumeBindingMode: Immediate` for topology-aware storage — may bind to a zone with no compute capacity.
4. Declaring a StorageClass with a provisioner not installed in the cluster — PVCs stay Pending indefinitely.

### Production Notes

- Use `volumeBindingMode: WaitForFirstConsumer` for cloud environments — ensures PV is created in the same zone as the Pod.
- Set `reclaimPolicy: Retain` for StorageClasses used by production PVCs.
- For multi-tenant clusters, create StorageClasses with different QoS parameters (IOPS, throughput) per tenant tier.
- Manage StorageClasses via a dedicated "infrastructure" Helm release that runs before workload releases.

---

## 18.11 Ingress

Ingress exposes HTTP/HTTPS routes from outside the cluster to services.

### Overview

Ingress resources require an Ingress Controller (NGINX, Traefik, HAProxy, etc.). Helm templates produce Ingress manifests that route traffic based on hostname and path.

### Template Example

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    {{- with .Values.ingress.annotations }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className | quote }}
  {{- if .Values.ingress.tls }}
  tls:
    - hosts:
        - {{ .Values.ingress.host | quote }}
      secretName: {{ .Values.ingress.tlsSecretName }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host | quote }}
      http:
        paths:
          {{- range .Values.ingress.paths }}
          - path: {{ .path }}
            pathType: {{ .pathType | default "Prefix" }}
            backend:
              service:
                name: {{ $.Release.Name }}
                port:
                  number: {{ .servicePort }}
          {{- end }}
{{- end }}
```

### Values Example

```yaml
ingress:
  enabled: true
  className: "nginx"
  host: "app.example.com"
  tls: true
  tlsSecretName: "app-tls-cert"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  paths:
    - path: /
      pathType: Prefix
      servicePort: 80
    - path: /api
      pathType: Prefix
      servicePort: 8080
```

### Management Considerations

**Host Rules:** Each `host` entry defines a virtual host. Traffic not matching any host goes to the default backend (if configured in the controller).

**TLS:** The `tls` section references a Secret containing the certificate. The Secret must exist in the same namespace. Helm does not create the TLS certificate — use cert-manager (see Section 18.27).

**`ingressClassName`:** References the IngressClass resource. Required in Kubernetes 1.18+. If omitted, the default IngressClass (if configured) is used.

**Path Types:**
- `Prefix`: Matches the beginning of the URL path (e.g., `/api` matches `/api/v1/users`).
- `Exact`: Matches exactly.
- `ImplementationSpecific`: Controller-specific matching.

**Multiple Ingress Controllers:** Different Ingress resources can point to different IngressClasses to route traffic through different controllers (e.g., internal NGINX for one app, external Traefik for another).

### Common Mistakes

1. Using `pathType: Prefix` on `/` — matches everything; order paths from most specific to least specific.
2. Not creating the TLS Secret before enabling TLS — Ingress controller fails to serve HTTPS.
3. Forgetting `ingressClassName` — uses the default controller, which may not exist.
4. Not setting up wildcard DNS before creating Ingress — host rules won't match.

### Production Notes

- Use cert-manager with `cert-manager.io/cluster-issuer` annotation for automatic TLS certificate provisioning and renewal.
- Always set `ssl-redirect: "true"` for public-facing Ingresses.
- Use `external-dns.alpha.kubernetes.io/hostname` annotation with ExternalDNS for automatic DNS record management.
- Implement NetworkPolicies to restrict Ingress controller egress (Section 18.13).

---

## 18.12 IngressClass

IngressClass defines a class of Ingress controller and optional parameters.

### Template Example

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: {{ .Values.ingressClass.name }}
  annotations:
    ingressclass.kubernetes.io/is-default-class: "{{ .Values.ingressClass.isDefault }}"
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  controller: {{ .Values.ingressClass.controller }}
  {{- if .Values.ingressClass.parameters }}
  parameters:
    apiGroup: {{ .Values.ingressClass.parameters.apiGroup }}
    kind: {{ .Values.ingressClass.parameters.kind }}
    name: {{ .Values.ingressClass.parameters.name }}
    scope: {{ .Values.ingressClass.parameters.scope | default "Namespace" }}
    namespace: {{ .Values.ingressClass.parameters.namespace }}
  {{- end }}
```

### Values Example

```yaml
ingressClass:
  name: "nginx-internal"
  isDefault: false
  controller: "k8s.io/ingress-nginx"
  parameters:
    apiGroup: "k8s.nginx.org"
    kind: "Policy"
    name: "internal-policy"
    scope: "Namespace"
    namespace: "ingress-nginx"
```

### Management Considerations

**Default IngressClass:** Annotation `ingressclass.kubernetes.io/is-default-class: "true"` marks it as default. All Ingresses without `ingressClassName` use this class.

**Controller-Specific Parameters:** The `parameters` section references a controller-specific resource (e.g., NGINX Policy, AWS Load Balancer Controller configuration). The API group must match the controller.

**Helm Management:** IngressClass is cluster-scoped. It should be managed by a dedicated infrastructure Helm release, not per-application charts.

### Common Mistakes

1. Multiple default IngressClasses — Ingresses without `ingressClassName` may route unpredictably.
2. Referencing a parameters object that doesn't exist — the Ingress controller may ignore the IngressClass.
3. Creating IngressClass in an application chart — cluster-scoped resources cause conflicts across releases.

### Production Notes

- Define IngressClasses once per cluster, managed by the platform team.
- Use separate IngressClasses for internal (private LB) and external (public LB) traffic.
- Attach controller-specific parameters for WAF rules, SSL policies, and logging configurations.

---

## 18.13 NetworkPolicy

NetworkPolicy is a firewall rule at the Pod/IP level. Requires a CNI that supports it (Calico, Cilium, Weave, etc.).

### Template Example

```yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  policyTypes:
    {{- toYaml .Values.networkPolicy.policyTypes | nindent 4 }}
  ingress:
    {{- toYaml .Values.networkPolicy.ingress | nindent 4 }}
  {{- if .Values.networkPolicy.egress }}
  egress:
    {{- toYaml .Values.networkPolicy.egress | nindent 4 }}
  {{- end }}
{{- end }}
```

### Values Example

```yaml
networkPolicy:
  enabled: true
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### Management Considerations

**`podSelector`:** Defines which Pods the policy applies to. Use the same `matchLabels` as the Deployment selector.

**Ingress vs Egress Rules:** `ingress` controls inbound traffic TO the selected pods. `egress` controls outbound traffic FROM the selected pods.

**`policyTypes`:** Must include `Ingress` and/or `Egress`. Declaring a type but providing no rules denies all traffic of that type. Omitting a type allows all traffic of that type.

**Namespace Isolation:** Use `namespaceSelector` to allow/deny traffic from entire namespaces. The label `kubernetes.io/metadata.name` is automatically set to the namespace name.

**Helm Upgrade Behavior:** NetworkPolicies are patched in place. Changes take effect immediately (controlled by CNI).

### Common Mistakes

1. Declaring `policyTypes: [Ingress]` without any ingress rules — blocks all inbound traffic (including DNS responses).
2. Forgetting DNS egress — Pods cannot resolve names without `kube-dns` egress rule.
3. Using `podSelector: {}` (selects all pods in namespace) when a narrower selector was intended.
4. Not installing a NetworkPolicy-enabled CNI — policies exist but are not enforced.

### Production Notes

- Apply a default-deny-all NetworkPolicy in every namespace as a safety baseline.
- Whitelist only the namespaces and Pods that need access.
- Always include egress to `kube-dns` (port 53 UDP/TCP) unless you have a DNS proxy on the node.
- Use `ipBlock` for IP-based rules (CIDR notation), but prefer `podSelector` and `namespaceSelector` for Kubernetes-native policies.

---

## 18.14 RBAC (ServiceAccount, Role, RoleBinding, ClusterRole, ClusterRoleBinding)

RBAC controls who can do what in the cluster.

### Overview

Helm creates RBAC resources like any other template. However, cluster-scoped resources (ClusterRole, ClusterRoleBinding) must be treated carefully — multiple releases may conflict.

### Template Example (Namespace-Scoped RBAC)

```yaml
{{- if .Values.rbac.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "mychart.serviceAccountName" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
rules:
  {{- toYaml .Values.rbac.rules | nindent 2 }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "mychart.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ include "mychart.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
{{- end }}
```

### Template Example (Cluster-Scoped RBAC)

```yaml
{{- if and .Values.rbac.create .Values.rbac.clusterWide }}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
rules:
  {{- toYaml .Values.rbac.clusterRules | nindent 2 }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ include "mychart.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ include "mychart.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
{{- end }}
```

### Values Example

```yaml
rbac:
  create: true
  clusterWide: false
  rules:
    - apiGroups: [""]
      resources: ["pods", "configmaps", "secrets"]
      verbs: ["get", "list", "watch"]
    - apiGroups: ["apps"]
      resources: ["deployments"]
      verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  clusterRules:
    - apiGroups: [""]
      resources: ["nodes"]
      verbs: ["get", "list", "watch"]
```

### Management Considerations

**Namespace-Scoped vs Cluster-Scoped:**
- Role + RoleBinding: Permissions within one namespace.
- ClusterRole + ClusterRoleBinding: Permissions across all namespaces or cluster-level resources (nodes, PVs, etc.).
- ClusterRole + RoleBinding: Grants cluster-wide permissions but scoped to a single namespace (useful pattern).

**Helm's Interaction with RBAC:** Helm creates and patches RBAC resources. Deleting a release deletes its Roles/RoleBindings. **ClusterRoles/ClusterRoleBindings are cluster-scoped** — ensure unique names across releases by prefixing with the release name.

**Deduplication of ClusterRole Names:** Use the fullname template function, which includes the release name: `{{ include "mychart.fullname" . }}`.

### Common Mistakes

1. Not gating ClusterRole creation behind `rbac.clusterWide` — multiple instances of the chart create conflicting ClusterRoles.
2. Using ClusterRoleBinding without including the namespace in the subject — grants cross-namespace access inadvertently.
3. Forgetting `verbs: ["get", "list", "watch"]` for crud-type access — "watch" is needed for controllers and informers.

### Production Notes

- Follow the Principle of Least Privilege: define only the exact verbs and resources needed.
- Use `resourceNames` in rules to restrict access to specific named resources.
- Audit RBAC with `kubectl auth can-i --as system:serviceaccount:<ns>:<sa> <verb> <resource>`.
- For controllers/operators, use ClusterRole + RoleBinding to keep namespace isolation.

---

## 18.15 CRDs (CustomResourceDefinitions)

CRDs extend the Kubernetes API with custom resource types.

### Overview

Helm has **special handling** for CRDs. CRDs in the `crds/` directory are installed before any templates and are NEVER updated or deleted by Helm upgrades or rollbacks. This prevents accidental CRD deletion (which would cascade-delete all custom resources).

### CRD in the crds/ Directory (Recommended)

Place CRD YAML files in `crds/` at the chart root. Template rendering is NOT applied to `crds/`. They are installed raw.

```
mychart/
  crds/
    myresource.example.com.yaml
  templates/
    myresource.yaml
```

### CRD as a Template (Not Recommended for Most Cases)

If you must template CRDs (e.g., conditional installation), place them in `templates/` — but be aware Helm WILL update, rollback, and potentially delete them.

```yaml
{{- if .Values.crd.install }}
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.{{ .Values.crd.group }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/resource-policy": keep
spec:
  group: {{ .Values.crd.group }}
  names:
    kind: MyResource
    listKind: MyResourceList
    plural: myresources
    singular: myresource
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                image:
                  type: string
{{- end }}
```

### Values Example

```yaml
crd:
  install: true
  group: "example.com"
```

### Management Considerations

**crds/ Directory Special Handling:**
- Installed BEFORE any templates in the release.
- NOT updated on `helm upgrade` — if you change a CRD in `crds/`, you must manually apply it: `kubectl apply -f crds/`.
- NOT deleted on `helm uninstall` — prevents data loss if custom resources exist.
- NOT included in `helm rollback`.

**CRD Installation Order:** CRDs must be installed before custom resources that depend on them. Helm's `crds/` directory guarantees this. For template-based CRDs, use hook weights.

**CRD Upgrades:** Since Helm 3, CRDs in `crds/` are never upgraded. For CRD version changes:
1. Update the CRD in `crds/`
2. Run `kubectl apply -f crds/new-version.yaml`
3. Helm upgrade the rest of the chart

**Conversion Webhooks:** If your CRD uses conversion webhooks, ensure the webhook service is running BEFORE applying the CRD. Helm's `crds/` directory handles this since CRDs are installed first, but the webhook itself must already exist.

### Common Mistakes

1. Expecting `helm upgrade` to update CRDs in the `crds/` directory — it doesn't.
2. Placing CRDs in `templates/` without `"helm.sh/resource-policy": keep` — uninstalling the release deletes the CRD and all custom resources.
3. Not versioning CRDs properly — adding new schema fields without bumping the API version breaks existing resources.

### Production Notes

- Always place CRDs in `crds/` unless you need conditional installation or runtime templating.
- Use `"helm.sh/resource-policy": keep` on CRD templates in `templates/`.
- Manage CRD lifecycle separately from application lifecycle.
- Use CRD schema validation to catch invalid custom resources at creation time.

---

## 18.16 ServiceAccounts

ServiceAccounts provide an identity for Pods. They are critical for cloud IAM integration.

### Overview

Helm templates can create ServiceAccounts. The modern pattern ties cloud IAM roles to ServiceAccounts via annotations rather than node-level instance profiles.

### Template Example

```yaml
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "mychart.serviceAccountName" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    {{- toYaml .Values.serviceAccount.annotations | nindent 4 }}
automountServiceAccountToken: {{ .Values.serviceAccount.automount }}
{{- with .Values.serviceAccount.imagePullSecrets }}
imagePullSecrets:
  {{- toYaml . | nindent 2 }}
{{- end }}
{{- end }}
```

### Values Example (AWS IRSA)

```yaml
serviceAccount:
  create: true
  automount: true
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789:role/myapp-role"
  imagePullSecrets:
    - name: ecr-registry
```

### Values Example (GCP Workload Identity)

```yaml
serviceAccount:
  create: true
  automount: true
  annotations:
    iam.gke.io/gcp-service-account: "myapp@myproject.iam.gserviceaccount.com"
```

### Values Example (Azure Pod Identity)

```yaml
serviceAccount:
  create: true
  automount: true
  annotations:
    aadpodidbinding: "myapp-identity"
```

### Management Considerations

**`automountServiceAccountToken`:** Set to `false` if the Pod does not need Kubernetes API access. Reduces attack surface. Default is `true`.

**`imagePullSecrets`:** References Secrets for pulling images from private registries. These must exist before Pod creation. Helm can create the Secret in the same release.

**IRSA (AWS IAM Roles for Service Accounts):** The `eks.amazonaws.com/role-arn` annotation maps an AWS IAM role to the ServiceAccount. The AWS SDKs in containers automatically use this. Requires the EKS OIDC provider to be set up.

**Workload Identity (GCP):** Annotate with `iam.gke.io/gcp-service-account` to bind a GCP service account. Requires Workload Identity to be enabled on the GKE cluster.

**Pod Identity (Azure):** Uses `aadpodidbinding` annotation. The `aad-pod-identity` or Azure Workload Identity controller manages the mapping.

**Helm Upgrade Behavior:** ServiceAccounts are mutable. Changing annotations during upgrade propagates to new Pods but does NOT restart existing Pods. Rolling updates are needed for credential changes to take effect.

### Common Mistakes

1. Forgetting to create the IAM trust relationship for IRSA — Pods fail with access denied.
2. Not specifying `imagePullSecrets` for private registries when using a custom ServiceAccount.
3. Setting `automountServiceAccountToken: false` on a controller that needs to list Pods.
4. Creating a ServiceAccount without corresponding RBAC — Pods can authenticate but have no permissions.

### Production Notes

- Always use IRSA, Workload Identity, or Pod Identity instead of node-level IAM roles.
- Set `automountServiceAccountToken: false` for all application Pods that don't need K8s API access.
- Manage cloud IAM bindings via Terraform or CloudFormation; use Helm only for the ServiceAccount annotation.
- Use `eksctl create iamserviceaccount` or `terraform` to set up the OIDC trust.

---

## 18.17 MutatingWebhook / ValidatingWebhook

Admission webhooks intercept API requests to validate or mutate objects before persistence.

### Overview

Webhooks are critical path components — if a webhook is down, Kubernetes API calls may fail. Helm must manage both the Webhook configuration AND the webhook service correctly.

### Template Example (ValidatingWebhook)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: {{ include "mychart.fullname" . }}
  annotations:
    cert-manager.io/inject-ca-from: {{ .Release.Namespace }}/{{ include "mychart.fullname" . }}-serving-cert
webhooks:
  - name: {{ .Values.webhook.validatingName }}
    failurePolicy: {{ .Values.webhook.failurePolicy }}
    timeoutSeconds: {{ .Values.webhook.timeoutSeconds }}
    matchPolicy: Equivalent
    sideEffects: None
    admissionReviewVersions: ["v1"]
    clientConfig:
      service:
        name: {{ include "mychart.fullname" . }}
        namespace: {{ .Release.Namespace }}
        path: /validate
        port: 443
    rules:
      {{- toYaml .Values.webhook.rules | nindent 6 }}
```

### Template Example (MutatingWebhook)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: {{ include "mychart.fullname" . }}
  annotations:
    cert-manager.io/inject-ca-from: {{ .Release.Namespace }}/{{ include "mychart.fullname" . }}-serving-cert
webhooks:
  - name: {{ .Values.webhook.mutatingName }}
    failurePolicy: {{ .Values.webhook.failurePolicy }}
    timeoutSeconds: {{ .Values.webhook.timeoutSeconds }}
    matchPolicy: Equivalent
    sideEffects: NoneOnDryRun
    reinvocationPolicy: Never
    admissionReviewVersions: ["v1"]
    clientConfig:
      service:
        name: {{ include "mychart.fullname" . }}
        namespace: {{ .Release.Namespace }}
        path: /mutate
        port: 443
    rules:
      {{- toYaml .Values.webhook.rules | nindent 6 }}
```

### Values Example

```yaml
webhook:
  validatingName: "myapp-validator.example.com"
  mutatingName: "myapp-mutator.example.com"
  failurePolicy: Fail
  timeoutSeconds: 10
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
      scope: "Namespaced"
```

### Management Considerations

**Webhook Ordering:** Kubernetes calls mutating webhooks first, then validating webhooks. Within each type, webhooks are called alphabetically by name. Name your webhooks accordingly (e.g., `010-mutate.example.com`, `020-mutate.example.com`).

**`failurePolicy`:**
- `Fail`: Webhook failure rejects the API request. Use for mandatory validation.
- `Ignore`: Webhook failure allows the request. Use for non-critical features.

**Helm's `crds/` Handling with Webhooks:** If your CRD has a conversion webhook, the webhook service must exist before the CRD. Place the Service and Deployment in `templates/` and the CRD in `crds/` — Helm installs `crds/` first, so the ordering works.

**Certificate Management:** Webhooks require TLS. Use cert-manager with `cert-manager.io/inject-ca-from` annotation. Alternatively, use an init container to generate self-signed certificates.

**`timeoutSeconds`:** API server waits this long for a webhook response. Default is 10. Set based on webhook latency. Values 1–30 seconds.

### Common Mistakes

1. Webhook service not yet running when CRD is installed — CRD installation fails if it references a conversion webhook.
2. Setting `failurePolicy: Fail` without high-availability webhook deployment — single point of failure for the entire API.
3. Not including `admissionReviewVersions: ["v1"]` — v1beta1 is removed in Kubernetes 1.22+.
4. Forgetting `sideEffects: None` — required for dry-run support.

### Production Notes

- Deploy webhooks with at least 2 replicas and a PodDisruptionBudget.
- Use `failurePolicy: Ignore` during initial deployment, then switch to `Fail` after validation.
- Monitor webhook latency — slow webhooks add latency to all API calls.
- Use cert-manager for automatic TLS certificate rotation.
- Test webhooks thoroughly in a non-production cluster before enabling `failurePolicy: Fail`.

---

## 18.18 PodDisruptionBudgets (PDBs)

PDBs limit the number of Pods that can be voluntarily disrupted simultaneously.

### Overview

PDBs work with the Eviction API to prevent too many Pods from going down during voluntary disruptions (node drains, rolling updates, `kubectl drain`). Helm creates PDBs like any other resource.

### Template Example

```yaml
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  {{- if .Values.pdb.minAvailable }}
  minAvailable: {{ .Values.pdb.minAvailable }}
  {{- end }}
  {{- if .Values.pdb.maxUnavailable }}
  maxUnavailable: {{ .Values.pdb.maxUnavailable }}
  {{- end }}
  {{- if .Values.pdb.unhealthyPodEvictionPolicy }}
  unhealthyPodEvictionPolicy: {{ .Values.pdb.unhealthyPodEvictionPolicy }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
{{- end }}
```

### Values Example

```yaml
pdb:
  enabled: true
  minAvailable: 2
  unhealthyPodEvictionPolicy: AlwaysAllow
```

Or:

```yaml
pdb:
  enabled: true
  maxUnavailable: 1
  unhealthyPodEvictionPolicy: IfHealthyBudget
```

### Management Considerations

**`minAvailable` vs `maxUnavailable`:** Mutually exclusive. `minAvailable: 2` means at least 2 Pods must remain available. `maxUnavailable: 1` means at most 1 Pod can be disrupted. Use `minAvailable` for fixed-size clusters; `maxUnavailable` for auto-scaled.

**`unhealthyPodEvictionPolicy` (Kubernetes 1.26+):**
- `AlwaysAllow`: Unhealthy Pods (not ready, unschedulable) can always be evicted regardless of budget.
- `IfHealthyBudget`: Unhealthy Pods are only evicted if the PDB budget is still satisfied by healthy Pods.
- Default (unset): Unhealthy Pods cannot be evicted unless `minAvailable/maxUnavailable` allows.

**Selector Matching:** The PDB selector must match the labels in the Pod template. If mismatched, the PDB has no effect.

**Helm Upgrade Behavior:** PDBs are patched in place. Changing `minAvailable` takes effect immediately for future evictions.

### Common Mistakes

1. Setting `maxUnavailable: 0` — blocks all voluntary disruptions, including regular rolling updates.
2. PDB selector not matching the Deployment/StatefulSet selector — PDB protects nothing.
3. Using `minAvailable` equal to `replicas` — prevents any voluntary eviction, including node drains.
4. Forgetting PDBs for StatefulSets — critical for database clusters.

### Production Notes

- Always define PDBs for production workloads with >1 replica.
- For StatefulSets running databases, set `maxUnavailable: 1` to protect quorum.
- For stateless Deployments with 3 replicas, `maxUnavailable: 1` allows one at a time.
- Use `unhealthyPodEvictionPolicy: AlwaysAllow` (K8s 1.26+) to prevent stuck node drains from unhealthy Pods.

---

## 18.19 HPA (HorizontalPodAutoscaler)

HPA automatically scales the number of Pods based on observed metrics.

### Overview

Helm templates HPA resources that reference the target workload's scale target. The HPA must not set `replicas` in the target workload manifest — conflict occurs.

### Template Example

```yaml
{{- if .Values.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "mychart.fullname" . }}
  minReplicas: {{ .Values.hpa.minReplicas }}
  maxReplicas: {{ .Values.hpa.maxReplicas }}
  metrics:
    {{- toYaml .Values.hpa.metrics | nindent 4 }}
  {{- if .Values.hpa.behavior }}
  behavior:
    {{- toYaml .Values.hpa.behavior | nindent 4 }}
  {{- end }}
{{- end }}
```

### Values Example

```yaml
hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
```

### Management Considerations

**Metrics:** Can use `Resource` (cpu/memory), `Pods` (custom pod metrics), `Object` (metrics from a specific object), or `External` (metrics from outside the cluster).

**`minReplicas` / `maxReplicas`:** Define the bounds. HPA overrides the `replicas` field in the Deployment. Do NOT set `replicas` in both the Deployment and HPA — it causes fights between the controllers.

**Behavior Section:** Controls how fast the HPA scales up and down. `stabilizationWindowSeconds` prevents flapping. `selectPolicy` (`Max`, `Min`, `Disabled`) chooses which scaling policy to apply.

**Multiple Metrics:** HPA scales based on the highest ratio among all metrics.

**Helm Upgrade Behavior:** HPA is patched in place. Changes to `minReplicas` or `maxReplicas` take effect immediately. The current target scale is not affected unless the new bounds constrain it.

### Common Mistakes

1. Setting both `spec.replicas` in Deployment AND HPA — controllers fight.
2. Forgetting to install the metrics-server — HPA cannot get resource metrics.
3. `maxReplicas` too low — HPA caps scaling even when demand is high.
4. No `stabilizationWindowSeconds` in `scaleDown` — HPA can thrash (scale down immediately after scaling up).

### Production Notes

- Always set `scaleDown.stabilizationWindowSeconds` to at least 300 (5 minutes).
- Set `scaleUp.stabilizationWindowSeconds: 0` for responsive scaling.
- Use `behavior.scaleUp.selectPolicy: Max` with two policies — percentage AND pods — for burst handling.
- Install metrics-server or Prometheus Adapter before deploying HPAs.
- For custom metrics, use Prometheus Adapter or KEDA (Section 18.21).

---

## 18.20 VPA (VerticalPodAutoscaler)

VPA automatically adjusts container resource requests based on usage.

### Overview

VPA requires the VPA controller (from kubernetes/autoscaler). Its CRD `VerticalPodAutoscaler` is not built into Kubernetes. Helm templates it like any custom resource.

### Template Example

```yaml
{{- if .Values.vpa.enabled }}
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "mychart.fullname" . }}
  updatePolicy:
    updateMode: {{ .Values.vpa.updateMode }}
  {{- if .Values.vpa.resourcePolicy }}
  resourcePolicy:
    {{- toYaml .Values.vpa.resourcePolicy | nindent 4 }}
  {{- end }}
{{- end }}
```

### Values Example

```yaml
vpa:
  enabled: true
  updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: 2
          memory: 4Gi
        controlledResources: ["cpu", "memory"]
```

### Management Considerations

**`updateMode`:**
- `Off`: VPA only provides recommendations, no changes applied.
- `Initial`: VPA sets resource requests only at Pod creation.
- `Recreate`: VPA evicts Pods when resource recommendations change.
- `Auto`: Like `Recreate`, but with no resource limits constraints.

**Resource Policy:** Controls which resources VPA manages and sets bounds via `minAllowed` and `maxAllowed`.

**Controlled Resources:** `controlledValues: RequestsOnly` (default) or `RequestsAndLimits`.

**Helm Interaction:** VPA is a CRD. The VPA controller must be installed separately (via its own Helm chart). VPA and HPA can coexist but should target different metrics (vertical vs horizontal).

### Common Mistakes

1. Using `updateMode: Auto` on a single-replica deployment — causes downtime on updates.
2. VPA and HPA both using CPU/memory — they can conflict.
3. Not setting `maxAllowed` — VPA can recommend excessively large resources.
4. Forgetting to install the VPA CRD and controller before applying the VPA resource.

### Production Notes

- Start with `updateMode: Off` to observe VPA recommendations before enabling automatic updates.
- Set `minAllowed` to prevent VPA from setting too-low values that cause OOMKills.
- Use VPA + HPA together: VPA sets the base resource request; HPA handles horizontal scaling.
- VPA `Recreate` mode causes Pod restarts — ensure your app is stateless or has proper shutdown/startup.
- Monitor VPA recommendations via Prometheus metrics exposed by the VPA recommender.

---

## 18.21 KEDA ScaledObject

KEDA provides event-driven autoscaling beyond CPU/memory metrics.

### Overview

KEDA (Kubernetes Event-Driven Autoscaling) creates `ScaledObject` CRDs that drive HPA based on external event sources (Kafka, RabbitMQ, Prometheus, cron schedules, etc.).

### Template Example

```yaml
{{- if .Values.keda.enabled }}
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "mychart.fullname" . }}
  minReplicaCount: {{ .Values.keda.minReplicaCount }}
  maxReplicaCount: {{ .Values.keda.maxReplicaCount }}
  pollingInterval: {{ .Values.keda.pollingInterval }}
  cooldownPeriod: {{ .Values.keda.cooldownPeriod }}
  {{- if .Values.keda.idleReplicaCount }}
  idleReplicaCount: {{ .Values.keda.idleReplicaCount }}
  {{- end }}
  triggers:
    {{- toYaml .Values.keda.triggers | nindent 4 }}
{{- end }}
```

### Values Example (Kafka Trigger)

```yaml
keda:
  enabled: true
  minReplicaCount: 1
  maxReplicaCount: 20
  pollingInterval: 30
  cooldownPeriod: 300
  idleReplicaCount: 0
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: "kafka-broker:9092"
        consumerGroup: "myapp-consumer"
        topic: "events"
        lagThreshold: "50"
        offsetResetPolicy: "latest"
```

### Values Example (Prometheus Trigger)

```yaml
keda:
  enabled: true
  minReplicaCount: 2
  maxReplicaCount: 15
  pollingInterval: 15
  cooldownPeriod: 120
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-server.monitoring.svc:9090
        metricName: http_requests_per_second
        threshold: "100"
        query: |
          sum(rate(http_requests_total{app="myapp"}[2m]))
```

### Values Example (Cron Trigger for Scheduled Scaling)

```yaml
keda:
  enabled: true
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: cron
      metadata:
        timezone: "America/New_York"
        start: "0 8 * * 1-5"
        end: "0 18 * * 1-5"
        desiredReplicas: "10"
```

### Management Considerations

**Triggers:** KEDA supports 50+ event sources. Each trigger has its own `metadata` fields. The `type` field specifies the scaler (e.g., `kafka`, `rabbitmq`, `prometheus`, `cron`, `cpu`, `memory`).

**Scaling Behavior:** KEDA creates a backing HPA internally. It scales to 0 when no events are present (unlike standard HPA), and scales up when events arrive.

**`idleReplicaCount`:** If set, KEDA scales to this number when no events are pending. Set to `0` to scale to zero.

**`pollingInterval` and `cooldownPeriod`:** Control how frequently KEDA checks trigger metrics and how long to wait before scaling down.

**Helm Interaction:** `ScaledObject` is a CRD. The KEDA operator must be installed first (usually via its own Helm chart).

### Common Mistakes

1. Not installing KEDA before applying ScaledObjects — resources remain unprocessed.
2. Setting `minReplicaCount: 0` on a Deployment with `replicas: 1` — KEDA may scale to zero unexpectedly.
3. Incorrect trigger metadata field names — each scaler has specific required fields; check KEDA docs.
4. Too-low `cooldownPeriod` — causes rapid scale-up/scale-down cycles.

### Production Notes

- Start with `cooldownPeriod: 300` (5 minutes) to prevent flapping.
- Use `idleReplicaCount: 1` for latency-sensitive workloads that cannot tolerate cold starts.
- For Kafka consumers, set `offsetResetPolicy: latest` and tune `lagThreshold` based on processing rate.
- Monitor KEDA metrics: `keda_scaler_metrics_value` and `keda_scaler_errors_total`.
- Combine KEDA cron triggers for predictable scaling (business hours) with metric triggers for burst handling.

---

## 18.22 Karpenter NodePool

Karpenter automatically provisions and manages node lifecycle.

### Overview

Karpenter's `NodePool` resource (replaces Provisioner in v1beta1+) defines node provisioning rules. Helm can manage NodePools for infrastructure-as-code workflows.

### Template Example

```yaml
{{- if .Values.karpenter.enabled }}
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  template:
    metadata:
      labels:
        {{- toYaml .Values.karpenter.nodeLabels | nindent 8 }}
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: {{ .Values.karpenter.nodeClassName }}
      requirements:
        {{- toYaml .Values.karpenter.requirements | nindent 8 }}
      taints:
        {{- toYaml .Values.karpenter.taints | nindent 8 }}
      terminationGracePeriod: {{ .Values.karpenter.terminationGracePeriod }}
  limits:
    cpu: "{{ .Values.karpenter.limits.cpu }}"
    memory: "{{ .Values.karpenter.limits.memory }}"
  disruption:
    consolidationPolicy: {{ .Values.karpenter.consolidationPolicy }}
    consolidateAfter: {{ .Values.karpenter.consolidateAfter }}
    expireAfter: {{ .Values.karpenter.expireAfter }}
{{- end }}
```

### Values Example

```yaml
karpenter:
  enabled: true
  nodeClassName: "default"
  nodeLabels:
    team: platform
    environment: production
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]
    - key: kubernetes.io/arch
      operator: In
      values: ["amd64", "arm64"]
    - key: karpenter.k8s.aws/instance-category
      operator: In
      values: ["c", "m", "r"]
    - key: karpenter.k8s.aws/instance-generation
      operator: Gt
      values: ["5"]
  taints:
    - key: workload-type
      value: batch
      effect: NoSchedule
  terminationGracePeriod: 1h
  limits:
    cpu: "1000"
    memory: "1000Gi"
  consolidationPolicy: WhenUnderutilized
  consolidateAfter: 1m
  expireAfter: 720h
```

### Management Considerations

**Consolidation:** Karpenter can consolidate workloads onto fewer nodes to reduce cost. `consolidationPolicy: WhenUnderutilized` enables this. `WhenEmpty` only removes empty nodes.

**`expireAfter`:** Maximum node age. Nodes are terminated after this period to ensure fresh AMIs and security patches. Recommended: 720h (30 days).

**Limits:** Caps total resources (cpu/memory) Karpenter can provision via this NodePool. Prevents runaway costs.

**Helm as Infrastructure Code:** Managing Karpenter NodePools via Helm is common in GitOps setups. Changes to NodePool specs are patched in place; Karpenter handles node transitions gracefully.

### Common Mistakes

1. Too-low `expireAfter` — causes excessive node churn (minimum 24h recommended).
2. Not setting `limits` — Karpenter can provision unlimited resources.
3. Restrictive `requirements` that exclude all instance types — no nodes can be provisioned.
4. Forgetting `terminationGracePeriod` — Pods may not have enough time to drain before node termination.

### Production Notes

- Set `consolidationPolicy: WhenUnderutilized` for cost optimization in non-production first.
- Use `expireAfter: 720h` (30 days) to balance security patching with stability.
- Define `limits` based on cluster budget to prevent cost overruns.
- Include spot instances with fallback to on-demand using capacity-type requirements.
- Monitor Karpenter logs and metrics: `karpenter_nodes_created`, `karpenter_nodes_terminated`, `karpenter_consolidation_decisions_total`.

---

## 18.23 Service

Services expose Pods via stable IP addresses and DNS names.

### Overview

Helm templates Services of all types: ClusterIP, NodePort, LoadBalancer, ExternalName, and headless. Services are fully mutable.

### Template Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  {{- if eq .Values.service.type "ClusterIP" }}
  {{- if .Values.service.clusterIP }}
  clusterIP: {{ .Values.service.clusterIP }}
  {{- end }}
  {{- end }}
  {{- if eq .Values.service.type "ExternalName" }}
  externalName: {{ .Values.service.externalName }}
  {{- end }}
  {{- if .Values.service.sessionAffinity }}
  sessionAffinity: {{ .Values.service.sessionAffinity }}
  {{- end }}
  ports:
    {{- range .Values.service.ports }}
    - port: {{ .port }}
      targetPort: {{ .targetPort }}
      protocol: {{ .protocol | default "TCP" }}
      name: {{ .name }}
      {{- if and (eq $.Values.service.type "NodePort") .nodePort }}
      nodePort: {{ .nodePort }}
      {{- end }}
    {{- end }}
  selector:
    {{- include "mychart.selectorLabels" . | nindent 4 }}
```

### Values Example (ClusterIP)

```yaml
service:
  type: ClusterIP
  sessionAffinity: None
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
```

### Values Example (LoadBalancer)

```yaml
service:
  type: LoadBalancer
  sessionAffinity: ClientIP
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  ports:
    - port: 443
      targetPort: 8443
      protocol: TCP
      name: https
```

### Values Example (Headless)

```yaml
service:
  type: ClusterIP
  clusterIP: "None"
  ports:
    - port: 7946
      targetPort: 7946
      name: gossip
```

### Management Considerations

**Service Types:**
- `ClusterIP`: Internal-only, accessible within the cluster (default).
- `NodePort`: Exposes the service on each node's IP at a static port (30000–32767).
- `LoadBalancer`: Provisions an external load balancer (cloud-provider-specific).
- `ExternalName`: Maps a Service to a DNS name (no selectors, no endpoints).
- Headless (`clusterIP: None`): No cluster IP; DNS returns Pod IPs directly. Required for StatefulSets.

**`sessionAffinity:`** `ClientIP` routes all requests from a client to the same Pod. `None` uses random load balancing.

**Changing Service Type:** `helm upgrade` can change the Service type. Changing from `LoadBalancer` to `ClusterIP` deallocates the external load balancer (check cloud provider behavior).

**Selector Changes:** The selector must match the Pod labels. If changed, the Service stops routing traffic to existing Pods.

### Common Mistakes

1. Changing `clusterIP` after creation — immutable field.
2. LoadBalancer type on a cluster without a cloud provider or MetalLB — service stays Pending forever.
3. Port conflicts when multiple Services share the same NodePort.
4. Forgetting headless Service for StatefulSets — Pod DNS names won't resolve.

### Production Notes

- Use `LoadBalancer` with cloud provider annotations for external access; prefer Ingress for HTTP.
- For internal services, use `ClusterIP` with explicit port naming for clarity.
- Set `sessionAffinity: ClientIP` only for stateful applications that require sticky sessions.
- Monitor Service endpoints: `kube_endpoint_address_available` and `kube_endpoint_address_not_ready`.

---

## 18.24 Prometheus ServiceMonitor / PodMonitor

ServiceMonitor and PodMonitor are CRDs from the Prometheus Operator that define scrape targets.

### Overview

Prometheus Operator discovers scrape targets via ServiceMonitors and PodMonitors instead of `prometheus.yml` scrape configs. Helm templates these CRDs for automated monitoring.

### Template Example (ServiceMonitor)

```yaml
{{- if .Values.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
    {{- with .Values.serviceMonitor.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  {{- if .Values.serviceMonitor.namespaceSelector }}
  namespaceSelector:
    {{- toYaml .Values.serviceMonitor.namespaceSelector | nindent 4 }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  endpoints:
    {{- toYaml .Values.serviceMonitor.endpoints | nindent 4 }}
{{- end }}
```

### Template Example (PodMonitor)

```yaml
{{- if .Values.podMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  podMetricsEndpoints:
    {{- toYaml .Values.podMonitor.endpoints | nindent 4 }}
{{- end }}
```

### Values Example

```yaml
serviceMonitor:
  enabled: true
  labels:
    release: prometheus
  namespaceSelector:
    any: true
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
      scrapeTimeout: 10s
      honorLabels: true
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
```

### Management Considerations

**ServiceMonitor vs PodMonitor:**
- `ServiceMonitor`: Discovers Pods through Services. Use when a Service exists (most common).
- `PodMonitor`: Discovers Pods directly. Use when no Service is needed (e.g., sidecars, raw Pod workloads).

**Selector Matching:** The `selector.matchLabels` must match the Service or Pod labels. The common pattern: use the same `mychart.selectorLabels` used by the Deployment and Service.

**Relabelings:** Modify or add labels before scraping. Critical relabelings: `__meta_kubernetes_pod_name` -> `pod`, `__meta_kubernetes_namespace` -> `namespace`.

**Namespace Selectors:** `any: true` allows Prometheus to scrape across all namespaces. By default, only the ServiceMonitor's namespace is allowed.

**Helm Interaction:** Prometheus Operator must be installed first. The ServiceMonitor is a CRD; Helm creates it only if Prometheus Operator CRDs exist.

### Common Mistakes

1. Forgetting the `release: prometheus` label — Prometheus Operator only discovers ServiceMonitors that match its `serviceMonitorSelector`.
2. Mismatched port name between the Service and the ServiceMonitor endpoint.
3. Not setting `honorLabels: true` when application metrics already have well-defined labels.
4. Creating ServiceMonitors before installing Prometheus Operator — CRD missing.

### Production Notes

- Use `release: prometheus` label (or match your Prometheus Operator's `serviceMonitorSelector`).
- Set `interval` based on metric cardinality: 30s for high-cardinality, 60s for moderate, 120s for low-priority.
- Always include relabelings for `pod` and `namespace` to enable multi-dimensional queries.
- For multi-tenant setups, constrain namespace selectors to specific `matchNames`.
- Test scrape configuration with `promtool` before deployment.

---

## 18.25 GrafanaDashboard

Grafana dashboards can be managed via ConfigMap or CRD (Grafana Operator).

### Overview

Two approaches: (1) ConfigMap with a specific label (Grafana sidecar pattern), (2) `GrafanaDashboard` CRD (Grafana Operator). Helm can create either.

### Template Example (ConfigMap-based Dashboard Provisioning)

```yaml
{{- if .Values.grafana.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}-dashboard
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
    {{ .Values.grafana.sidecar.label }}: "1"
data:
  {{ (.Files.Glob "dashboards/*.json").AsConfig | nindent 2 }}
{{- end }}
```

### Template Example (CRD-based via Grafana Operator)

```yaml
{{- if .Values.grafana.enabled }}
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
    {{- with .Values.grafana.dashboardLabels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  instanceSelector:
    matchLabels:
      {{- toYaml .Values.grafana.instanceSelector | nindent 6 }}
  folder: {{ .Values.grafana.folder | quote }}
  allowCrossNamespaceImport: {{ .Values.grafana.allowCrossNamespaceImport }}
  json: |
    {{- .Files.Get "dashboards/app-dashboard.json" | nindent 4 }}
{{- end }}
```

### Values Example (ConfigMap approach)

```yaml
grafana:
  enabled: true
  sidecar:
    label: "grafana_dashboard"
```

### Values Example (CRD approach)

```yaml
grafana:
  enabled: true
  folder: "Applications"
  allowCrossNamespaceImport: true
  dashboardLabels:
    app: grafana
  instanceSelector:
    dashboards: "grafana"
```

### Management Considerations

**ConfigMap vs CRD:**
- **ConfigMap:** Simpler, uses Grafana sidecar container to watch for dashboard file changes. Dashboard JSON is stored as ConfigMap `data` entries.
- **CRD (Grafana Operator):** More powerful. Supports cross-namespace imports, folder organization, and lifecycle management. Requires Grafana Operator.

**Dashboard JSON Size:** ConfigMap max size is ~1 MiB. Large dashboards may hit this limit. CRDs have no such restriction but may hit etcd object size limits.

**Grafana Sidecar Label:** The sidecar (e.g., `k8s-sidecar`) watches ConfigMaps with a specific label and writes dashboard JSON files to Grafana's provisioning directory.

**Helm Interaction:** ConfigMaps are patched normally. Dashboard changes propagate when the sidecar syncs (typically every 60s). For CRDs, the Grafana Operator watches for changes.

### Common Mistakes

1. Exceeding 1 MiB ConfigMap limit with large dashboard JSON.
2. Wrong sidecar label — Grafana never picks up the dashboard.
3. JSON syntax errors in dashboard — sidecar writes invalid files, Grafana fails to load.
4. Forgetting to add `{{ .Files.Get }}` for dashboard JSON files — template renders empty.

### Production Notes

- Store dashboard JSON files in a `dashboards/` directory at the chart root.
- Use `.Files.Glob` + `.AsConfig` to load all JSON files from the `dashboards/` directory.
- Validate dashboard JSON before packaging with `helm lint` or `jq`.
- For GitOps, prefer CRD-based dashboards with version-controlled JSON.
- Tag dashboards with `grafana_folder` annotation to organize them in Grafana.

---

## 18.26 Istio VirtualService / DestinationRule

Istio VirtualService and DestinationRule control traffic routing within the service mesh.

### Overview

Istio CRDs enable advanced traffic management: canary deployments, fault injection, retries, timeouts, and circuit breaking. Helm templates generate these CRDs for service mesh configurations.

### Template Example (VirtualService)

```yaml
{{- if .Values.istio.enabled }}
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  hosts:
    - {{ include "mychart.fullname" . }}
    {{- with .Values.istio.additionalHosts }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
  gateways:
    {{- toYaml .Values.istio.gateways | nindent 4 }}
  http:
    - match:
        {{- toYaml .Values.istio.match | nindent 8 }}
      route:
        {{- toYaml .Values.istio.route | nindent 8 }}
      {{- if .Values.istio.retries }}
      retries:
        {{- toYaml .Values.istio.retries | nindent 8 }}
      {{- end }}
      {{- if .Values.istio.fault }}
      fault:
        {{- toYaml .Values.istio.fault | nindent 8 }}
      {{- end }}
      {{- if .Values.istio.timeout }}
      timeout: {{ .Values.istio.timeout }}
      {{- end }}
{{- end }}
```

### Template Example (DestinationRule)

```yaml
{{- if .Values.istio.enabled }}
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  host: {{ include "mychart.fullname" . }}
  {{- if .Values.istio.trafficPolicy }}
  trafficPolicy:
    {{- toYaml .Values.istio.trafficPolicy | nindent 4 }}
  {{- end }}
  subsets:
    {{- toYaml .Values.istio.subsets | nindent 4 }}
{{- end }}
```

### Values Example (Canary Deployment)

```yaml
istio:
  enabled: true
  gateways:
    - istio-system/public-gateway
  match:
    - uri:
        prefix: /api
  route:
    - destination:
        host: myapp
        port:
          number: 8080
        subset: stable
      weight: 90
    - destination:
        host: myapp
        port:
          number: 8080
        subset: canary
      weight: 10
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
  retries:
    attempts: 3
    perTryTimeout: 2s
    retryOn: "5xx,connect-failure,refused-stream"
```

### Values Example (Fault Injection)

```yaml
istio:
  enabled: true
  route:
    - destination:
        host: myapp
        port:
          number: 8080
  fault:
    delay:
      percentage:
        value: 10
      fixedDelay: 5s
    abort:
      percentage:
        value: 5
      httpStatus: 503
```

### Management Considerations

**VirtualService:** Defines routing rules for a host. Controls which traffic goes to which service subset. Works with DestinationRule to implement canary, A/B testing, and blue/green.

**DestinationRule:** Defines subsets (named versions of a service) and traffic policies (load balancing, connection pool, circuit breaker).

**Subsets for Canary:** Create subsets with labels matching different Deployment versions. VirtualService splits traffic between subsets.

**Fault Injection:** Inject delays or aborts to test application resilience. Use `percentage.value` to control blast radius.

**Retries and Timeouts:** Configure retry behavior per-route. Set `perTryTimeout` to cap individual request time.

**Helm Interaction:** Istio CRDs must be installed before VirtualService/DestinationRule. Helm creates these as standard CRD instances.

### Common Mistakes

1. Subset labels not matching Pod labels — traffic goes nowhere.
2. Forgetting DestinationRule when using VirtualService subsets — subsets must be defined in DestinationRule.
3. Too-high fault injection percentages in production — can cascade failures.
4. Route destination host not matching an existing service.

### Production Notes

- Start canary at 5% traffic, monitor, then increase gradually.
- Use `retries.retryOn` to define retryable status codes explicitly.
- Set `connectionPool` limits in DestinationRule's `trafficPolicy` to prevent cascading failures.
- Use `outlierDetection` for circuit breaking — eject unhealthy hosts automatically.
- Implement mirroring/traffic-shadowing for testing new versions with production traffic.

---

## 18.27 Cert-Manager Certificate

Cert-Manager automates TLS certificate issuance and renewal.

### Overview

Cert-Manager uses CRDs: `Issuer` (namespace-scoped), `ClusterIssuer` (cluster-scoped), and `Certificate`. Helm can create all three.

### Template Examples

**Issuer (Let's Encrypt HTTP01):**

```yaml
{{- if .Values.certmanager.enabled }}
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: {{ include "mychart.fullname" . }}-issuer
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  acme:
    server: {{ .Values.certmanager.server }}
    email: {{ .Values.certmanager.email }}
    privateKeySecretRef:
      name: {{ include "mychart.fullname" . }}-issuer-key
    solvers:
      - http01:
          ingress:
            class: {{ .Values.ingress.className }}
{{- end }}
```

**ClusterIssuer (Let's Encrypt DNS01):**

```yaml
{{- if .Values.certmanager.enabled }}
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: {{ include "mychart.fullname" . }}-cluster-issuer
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  acme:
    server: {{ .Values.certmanager.server }}
    email: {{ .Values.certmanager.email }}
    privateKeySecretRef:
      name: {{ include "mychart.fullname" . }}-cluster-issuer-key
    solvers:
      - dns01:
          route53:
            region: {{ .Values.certmanager.route53Region }}
            hostedZoneID: {{ .Values.certmanager.hostedZoneID }}
{{- end }}
```

**Certificate:**

```yaml
{{- if .Values.certmanager.enabled }}
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: {{ include "mychart.fullname" . }}-tls
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  secretName: {{ include "mychart.fullname" . }}-tls
  issuerRef:
    name: {{ include "mychart.fullname" . }}-issuer
    kind: Issuer
  {{- if .Values.certmanager.commonName }}
  commonName: {{ .Values.certmanager.commonName }}
  {{- end }}
  dnsNames:
    {{- toYaml .Values.certmanager.dnsNames | nindent 4 }}
  duration: {{ .Values.certmanager.duration | default "2160h" }}
  renewBefore: {{ .Values.certmanager.renewBefore | default "720h" }}
{{- end }}
```

### Values Example

```yaml
certmanager:
  enabled: true
  server: "https://acme-v02.api.letsencrypt.org/directory"
  email: "ops@example.com"
  dnsNames:
    - app.example.com
    - "*.app.example.com"
  commonName: "app.example.com"
  duration: "2160h"
  renewBefore: "720h"
```

### Management Considerations

**Issuer vs ClusterIssuer:**
- `Issuer`: Namespace-scoped. Certificates can only be issued in the same namespace.
- `ClusterIssuer`: Cluster-scoped. Any namespace can reference it. Use for wildcard or shared certificates.

**DNS01 vs HTTP01 Challenges:**
- `HTTP01`: Proves domain ownership by placing a file at `/.well-known/acme-challenge/`. Requires publicly accessible Ingress. Cannot issue wildcard certificates.
- `DNS01`: Proves domain ownership via DNS TXT record. Can issue wildcards. Requires DNS provider API access (Route53, Cloud DNS, etc.).

**`duration` vs `renewBefore`:**
- `duration`: How long the certificate is valid (default: 90 days, max: 90 days for Let's Encrypt).
- `renewBefore`: How long before expiry to renew (default: 30 days).

**Helm Interaction:** Cert-Manager CRDs must be installed first. Helm creates Issuer/Certificate; Cert-Manager controller handles the ACME workflow.

### Common Mistakes

1. Using HTTP01 without a publicly reachable Ingress — challenge fails.
2. Forgetting to open port 80 for HTTP01 challenges — ACME server cannot reach the challenge endpoint.
3. DNS01 without providing cloud credentials — Cert-Manager cannot create DNS records.
4. ClusterIssuer referenced with `kind: Issuer` in Certificate — reference mismatch, certificate stuck pending.

### Production Notes

- Use `ClusterIssuer` for shared certificates; `Issuer` for namespace-isolated apps.
- For wildcard certificates, DNS01 is required (HTTP01 does not support wildcards).
- Set `renewBefore: 720h` (30 days) for Let's Encrypt to allow ample renewal window.
- Monitor `certmanager_certificate_ready_status` metric for expiring certificates.
- For Route53 DNS01, use IRSA (IAM Roles for Service Accounts) instead of static credentials.

---

## 18.28 ExternalDNS

ExternalDNS synchronizes Kubernetes Services/Ingresses with DNS providers.

### Overview

ExternalDNS watches for annotated resources and automatically creates/updates DNS records. Helm can manage ExternalDNS annotations on resources.

### Template Example (Ingress with ExternalDNS Annotations)

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
  annotations:
    {{- with .Values.ingress.annotations }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
    external-dns.alpha.kubernetes.io/hostname: {{ .Values.ingress.host | quote }}
    external-dns.alpha.kubernetes.io/ttl: {{ .Values.externalDns.ttl | quote | default "300" }}
    {{- if .Values.externalDns.ingressHostnameSource }}
    external-dns.alpha.kubernetes.io/ingress-hostname-source: {{ .Values.externalDns.ingressHostnameSource }}
    {{- end }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  # ... Ingress spec
{{- end }}
```

### Template Example (Service with ExternalDNS Annotations)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  annotations:
    external-dns.alpha.kubernetes.io/hostname: {{ .Values.externalDns.hostname | quote }}
    {{- if .Values.externalDns.targetHostname }}
    external-dns.alpha.kubernetes.io/target: {{ .Values.externalDns.targetHostname }}
    {{- end }}
    external-dns.alpha.kubernetes.io/ttl: "{{ .Values.externalDns.ttl }}"
spec:
  type: LoadBalancer
  # ... Service spec
```

### Values Example

```yaml
externalDns:
  hostname: "app.example.com"
  ttl: "300"
  ingressHostnameSource: "annotation-only"
```

### Management Considerations

**Sources:** ExternalDNS watches:
- `service`: Creates DNS records for LoadBalancer Services.
- `ingress`: Creates DNS records for Ingresses.
- `istio-gateway`, `crd`, and others.

**Annotation-Based Registration:** The annotations `external-dns.alpha.kubernetes.io/hostname` and `external-dns.alpha.kubernetes.io/target` control DNS record creation. ExternalDNS watches for these.

**Zone Management:** ExternalDNS uses the `--domain-filter` flag to specify which zones to manage. Ensure the hostnames resolve to zones managed by ExternalDNS.

**Helm Interaction:** ExternalDNS itself is typically deployed via Helm. Application charts add the `external-dns.alpha.kubernetes.io/hostname` annotation to Ingresses/Services.

### Common Mistakes

1. Annotating a Service that has no external IP yet — ExternalDNS creates DNS pointing to nothing.
2. Hostname not matching any managed zone (`--domain-filter`).
3. Forgetting RBAC for ExternalDNS to watch Services/Ingresses in the namespace.
4. DNS record propagation delay — TTL too high; old IP cached after changes.

### Production Notes

- Set TTL to 60–300 seconds for records that change frequently (blue/green deployments).
- Use `external-dns.alpha.kubernetes.io/target` to point to a specific load balancer.
- For multi-cluster setups, use `external-dns.alpha.kubernetes.io/set-identifier` for ownership management.
- ExternalDNS + cert-manager + Ingress form the standard triad for public-facing Kubernetes services.

---

## 18.29 ArgoCD Application

ArgoCD Application CRD defines what to deploy and where.

### Overview

Helm can create ArgoCD `Application` resources (the "App of Apps" pattern). This is how Helm bootstraps GitOps workflows: a single Helm release creates the root ArgoCD Application, which then deploys everything else.

### Template Example (App of Apps)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{ include "mychart.fullname" . }}
  namespace: {{ .Values.argocd.namespace | default "argocd" }}
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  project: {{ .Values.argocd.project }}
  source:
    repoURL: {{ .Values.argocd.source.repoURL }}
    targetRevision: {{ .Values.argocd.source.targetRevision }}
    path: {{ .Values.argocd.source.path }}
    helm:
      {{- toYaml .Values.argocd.source.helm | nindent 6 }}
  destination:
    server: {{ .Values.argocd.destination.server }}
    namespace: {{ .Values.argocd.destination.namespace }}
  syncPolicy:
    {{- toYaml .Values.argocd.syncPolicy | nindent 4 }}
```

### Values Example

```yaml
argocd:
  project: "default"
  source:
    repoURL: "https://github.com/org/infra-repo.git"
    targetRevision: "main"
    path: "charts/root-app"
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "v1.2.3"
  destination:
    server: "https://kubernetes.default.svc"
    namespace: "myapp"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 3
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m
```

### Management Considerations

**App of Apps Pattern:** A single Helm chart creates an ArgoCD `Application` that points to a Git repository containing more `Application` CRDs. ArgoCD then deploys the actual workloads.

**Managing ArgoCD Applications with Helm:** The chart creates the `Application` CRD. Once created, ArgoCD takes over — any divergence between the Application manifest and Git is reconciled by ArgoCD (if `selfHeal: true`).

**Sync Policies:**
- `automated.prune`: Delete resources in the cluster that are not in Git.
- `automated.selfHeal`: Revert manual changes to match Git state.
- `syncOptions.CreateNamespace=true`: ArgoCD creates the namespace if it doesn't exist.

**Helm Upgrade vs ArgoCD Sync:** `helm upgrade` on the App-of-Apps chart changes the ArgoCD Application manifest, which ArgoCD then uses to sync child applications. This creates a two-tier management model.

### Common Mistakes

1. Enabling `selfHeal` without understanding it reverts ALL manual changes — including emergency hotfixes.
2. Not including `resources-finalizer.argocd.argoproj.io` — deleting the Application doesn't clean up deployed resources.
3. Using `syncOptions: CreateNamespace=true` without proper namespace-scoped RBAC.
4. Circular dependencies: ArgoCD Application managed by ArgoCD itself (use `argocd-autopilot` or bootstrap pattern).

### Production Notes

- Use separate ArgoCD projects for different environments (dev, staging, production).
- Set `automated.prune: true` only after verifying the sync strategy.
- Use `PrunePropagationPolicy: foreground` to ensure proper deletion order.
- Implement Progressive Sync with ArgoCD Rollouts for canary deployments.
- Monitor `argocd_app_sync_status` and `argocd_app_health_status` metrics.

---

## 18.30 ResourceQuota and LimitRange

ResourceQuota and LimitRange enforce resource constraints at the namespace level.

### Overview

These resources constrain resource consumption in a namespace. They are typically managed by platform teams via infrastructure Helm releases, not per-application charts.

### Template Example (ResourceQuota)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {{ include "mychart.fullname" . }}-quota
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  hard:
    {{- toYaml .Values.quota.hard | nindent 4 }}
```

### Template Example (LimitRange)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: {{ include "mychart.fullname" . }}-limits
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  limits:
    {{- toYaml .Values.limitRange.limits | nindent 4 }}
```

### Values Example

```yaml
quota:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "10"
    requests.storage: "500Gi"
    count/deployments.apps: "20"
    count/services: "10"
    count/secrets: "50"
    count/configmaps: "50"

limitRange:
  limits:
    - type: Container
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
    - type: PersistentVolumeClaim
      max:
        storage: "100Gi"
      min:
        storage: "1Gi"
```

### Management Considerations

**ResourceQuota:** Enforces total resource consumption per namespace. When the quota is exceeded, new resource creation is rejected (HTTP 403: Forbidden).

**LimitRange:** Constrains individual Pod/Container resource requests and limits. Sets defaults for containers that don't specify resources.

**Helm Interaction:** These should be managed by platform-level charts (e.g., `namespace-bootstrap`), not individual app charts. Application charts should not create ResourceQuotas — that would constrain the namespace for all users.

**Enforcement:** ResourceQuotas and LimitRanges are enforced by the API server admission controller (`ResourceQuota` admission plugin must be enabled).

### Common Mistakes

1. App chart creating ResourceQuotas — constrains the namespace and breaks other deployments.
2. Not setting `defaultRequest` in LimitRange — Pods without resource requests fail to schedule.
3. Forgetting that ResourceQuotas apply across ALL Pods in the namespace, not just the current release.
4. Setting quota limits too low — prevents valid operations like rolling updates (which need extra resources).

### Production Notes

- Use a dedicated infrastructure Helm chart for namespace provisioning + ResourceQuota + LimitRange.
- Set ResourceQuotas at 120% of typical usage to allow rolling updates (extra Pods temporarily).
- Always define `default` and `defaultRequest` in LimitRange to prevent unbounded Pods.
- Use `scopeSelector` for targeted quotas (e.g., only BestEffort, NotBestEffort, Terminating, NotTerminating).

---

## 18.31 PriorityClass

PriorityClass assigns priority to Pods — higher priority Pods can preempt lower ones.

### Overview

PriorityClasses are cluster-scoped resources that Helm can create. They affect scheduling order and eviction behavior during resource pressure.

### Template Example

```yaml
{{- if .Values.priorityClass.create }}
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: {{ include "mychart.fullname" . }}-priority
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
value: {{ .Values.priorityClass.value }}
globalDefault: {{ .Values.priorityClass.globalDefault | default false }}
description: {{ .Values.priorityClass.description | quote }}
{{- end }}
```

### Values Example

```yaml
priorityClass:
  create: true
  value: 100000
  globalDefault: false
  description: "Critical production workload"
```

### Usage in Pod Spec

```yaml
spec:
  priorityClassName: myapp-priority
  containers:
    - name: myapp
      # ...
```

### Management Considerations

**Value Range:** Kubernetes reserves values > 1,000,000,000 for system-critical components. User-defined priority classes should be <= 1,000,000,000.

**Preemption:** If a high-priority Pod cannot be scheduled, the scheduler may evict (preempt) lower-priority Pods to make room.

**`globalDefault: true`:** Only ONE PriorityClass can be the global default. It applies to Pods without an explicit `priorityClassName`.

**Helm Interaction:** PriorityClass is cluster-scoped and should be managed by infrastructure charts, not application charts — unless the application is the sole user of that priority class.

### Common Mistakes

1. Setting `globalDefault: true` on multiple PriorityClasses — only one is respected.
2. Too-high priority value preempts system Pods — can destabilize the cluster.
3. Application chart creating PriorityClass — conflicts when multiple releases try to manage the same resource.
4. Forgetting `preemptionPolicy: Never` for Pods that should never be preempted (Kubernetes 1.24+).

### Production Notes

- Define PriorityClasses at cluster bootstrap; reference them by name in application charts.
- Use a three-tier scheme: `high` (100000) for production, `medium` (50000) for staging, `low` (10000) for dev/tools.
- Never set user-defined priorities above 1,000,000,000 — reserved for `system-cluster-critical` and `system-node-critical`.
- Combine PriorityClass with PodDisruptionBudget to protect critical workloads from both preemption and voluntary disruption.

---

## 18.32 PodSecurity Admission (PSA) / PodSecurityPolicy

PodSecurity Admission (PSA) replaced PodSecurityPolicy (PSP) in Kubernetes 1.25. PSA labels are applied at the namespace level.

### Overview

PSA enforces Pod security standards (privileged, baseline, restricted) via namespace labels. Helm charts should include PSA labels in namespace manifests or the chart's namespace metadata templates (if it creates namespaces).

### Template Example (Namespace with PSA Labels)

```yaml
{{- if .Values.namespace.create }}
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
    pod-security.kubernetes.io/enforce: {{ .Values.podSecurity.enforce }}
    pod-security.kubernetes.io/enforce-version: {{ .Values.podSecurity.enforceVersion | default "latest" }}
    pod-security.kubernetes.io/audit: {{ .Values.podSecurity.audit }}
    pod-security.kubernetes.io/audit-version: {{ .Values.podSecurity.auditVersion | default "latest" }}
    pod-security.kubernetes.io/warn: {{ .Values.podSecurity.warn }}
    pod-security.kubernetes.io/warn-version: {{ .Values.podSecurity.warnVersion | default "latest" }}
{{- end }}
```

### Values Example

```yaml
namespace:
  create: true
podSecurity:
  enforce: "restricted"
  enforceVersion: "latest"
  audit: "restricted"
  auditVersion: "latest"
  warn: "restricted"
  warnVersion: "latest"
```

### Exemptions for Specific Labels (per-namespace)

```yaml
podSecurity:
  enforce: "privileged"
```

Or for a monitoring namespace that needs host-level access:

```yaml
podSecurity:
  enforce: "privileged"
```

### Management Considerations

**Three Standards:**
- `privileged`: No restrictions. Unrestricted policy with the widest level of permissions.
- `baseline`: Minimally restrictive. Prevents known privilege escalations. Allows default (minimally specified) Pod config.
- `restricted`: Heavily restricted. Follows Pod hardening best practices at the cost of some compatibility.

**PSA Labels:**
- `enforce`: Policy violations cause Pod rejection.
- `audit`: Policy violations are recorded in audit logs but Pods are allowed.
- `warn`: Policy violations produce user-facing warnings but Pods are allowed.

**Helm Interaction:** PSA labels are namespace-level. If the chart creates a namespace (via `namespace.create`), include PSA labels. If deploying into an existing namespace, the namespace must already have PSA labels — they cannot be set by a Helm release deploying to that namespace.

**PodSecurityPolicy (Deprecated):** PSP is removed in Kubernetes 1.25. Migrate to PSA or a policy engine like OPA/Gatekeeper or Kyverno.

### Common Mistakes

1. Setting `enforce: restricted` without adapting Pod specs — all Pods fail to create.
2. Applying PSA labels to an existing namespace with running Pods — existing Pods are not validated, but new Pods are.
3. Not testing with `warn` mode before switching to `enforce` — production outages from rejected Pods.
4. Forgetting that `restricted` requires explicit `securityContext` settings (non-root user, read-only root filesystem, etc.).

### Production Notes

- Rollout PSA gradually: start with `warn: restricted`, then `audit: restricted`, finally `enforce: restricted`.
- Use `enforce-version: latest` to always enforce the latest security standard version.
- For legacy apps that cannot meet `restricted`, use `baseline` and document the exceptions.
- Combine PSA with OPA/Gatekeeper for fine-grained policy enforcement beyond the three standards.
- Test your Pod specs against `restricted` with `kubectl label --dry-run=server -n test-ns pod-security.kubernetes.io/enforce=restricted`.

---

## Chapter Summary

This chapter covered Helm's interaction with 32 major Kubernetes resource types. Key takeaways:

1. **Immutability awareness:** Several fields (Deployment selector, PVC storageClassName, volumeClaimTemplate, Secret/ConfigMap immutable flag) cannot be changed after creation. Helm upgrades that attempt to change these will fail.

2. **Special Helm behaviors:** CRDs in `crds/` are never updated or deleted. Hooks (`helm.sh/hook`) control Job lifecycle. `helm.sh/resource-policy: keep` preserves resources during uninstall.

3. **Cluster-scoped vs namespace-scoped:** Cluster-scoped resources (PV, StorageClass, ClusterRole, ClusterRoleBinding, IngressClass, PriorityClass, CRDs) require unique naming conventions and careful management to avoid release conflicts.

4. **Checksum annotation pattern:** The `{{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}` pattern is essential for triggering Pod rolling updates when ConfigMaps change.

5. **External secret management:** Never hardcode secrets in Helm values. Use External Secrets Operator, Sealed Secrets, or Vault.

6. **Infrastructure-as-Code separation:** Platform resources (StorageClass, NetworkPolicy, ResourceQuota, PriorityClass, IngressClass) should be managed by separate infrastructure Helm releases, not per-application charts.

7. **Controller dependencies:** CRDs from operators (cert-manager, KEDA, Karpenter, Istio, Prometheus Operator) must be installed before their custom resource instances. Helm's `crds/` directory handles this for bundled CRDs.

