# Chapter 11: Helm Hooks

## What Are Helm Hooks?

Hooks are Kubernetes resources (Jobs, Pods, ConfigMaps, etc.) that Helm executes at specific points in the release lifecycle. They enable operations like database migrations, cleanup tasks, smoke tests, notifications, and any custom logic that must run before or after Helm installs, upgrades, deletes, or rolls back a release.

**Key concept:** Hooks are not part of the standard Helm release. They run at designated lifecycle phases and are tracked separately from the release's core resources.

## How Hooks Work

Hooks are ordinary Kubernetes templates annotated with `helm.sh/hook` (and optionally `helm.sh/hook-weight` and `helm.sh/hook-delete-policy`). When Helm reaches the phase specified by the annotation, it:

1. Executes (creates) all resources annotated for that hook phase.
2. Waits for those resources to reach completion (for Jobs: `Complete`; for Pods: `Succeeded`).
3. Optionally deletes the hook resources based on the delete policy.
4. Continues with the release lifecycle.

**Note:** If a hook resource fails (Job fails, Pod exits non-zero), Helm blocks the release and reports the failure. The release is not marked as successful until all hooks for that phase complete.

### Hook Resource Flow

```
helm install myapp ./chart
     │
     ▼
┌──────────────────┐
│  pre-install     │  ← hooks annotated with "helm.sh/hook: pre-install"
│  hooks execute   │
└──────┬───────────┘
       │ (block until complete)
       ▼
┌──────────────────┐
│  core resources  │  ← Deployments, Services, ConfigMaps, etc.
│  created         │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  post-install    │  ← hooks annotated with "helm.sh/hook: post-install"
│  hooks execute   │
└──────┬───────────┘
       │
       ▼
     Release marked as deployed
```

## All 8 Hook Types

Helm provides 9 annotations for lifecycle hooks (8 lifecycle phases + 1 special test phase):

### 1. `pre-install`

Executes **after** templates are rendered but **before** any release resources are created.

**Annotation:** `helm.sh/hook: pre-install`

**Use cases:**
- Creating required namespaces or RBAC before deployment
- Validating prerequisites (e.g., CRDs must exist)
- Running database schema backups before a new install
- Setting up external dependencies (message queues, API keys)

**Example:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-pre-install-check
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: check-prereqs
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Checking prerequisites for {{ .Release.Name }} v{{ .Chart.Version }}"
              echo "Namespace: {{ .Release.Namespace }}"
              echo "Pre-install validation passed."
```

### 2. `post-install`

Executes **after** all resources in the release have been created and are in a ready state (when `--wait` is used) or immediately after creation (default behavior).

**Annotation:** `helm.sh/hook: post-install`

**Use cases:**
- Running smoke tests or health checks
- Sending deployment notifications (Slack, email, PagerDuty)
- Populating initial data (seed data for databases)
- Triggering external CI/CD integrations
- Updating configuration management databases (CMDB)

**Example:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-smoke-test
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: smoke-test
          image: curlimages/curl:8.4.0
          command:
            - sh
            - -c
            - |
              echo "Running smoke test against {{ .Release.Name }}"
              curl -f -s -o /dev/null http://{{ .Release.Name }}-service:{{ .Values.service.port }}/health
              echo "Smoke test passed."
```

### 3. `pre-delete`

Executes **before** any resources in the release are deleted.

**Annotation:** `helm.sh/hook: pre-delete`

**Use cases:**
- Draining traffic from load balancers before pod termination
- Taking database backups before data is deleted
- Unregistering from service discovery / Consul / etcd
- Sending pending notifications to external systems
- Cleaning up external resources (DNS records, cloud load balancers that Terraform won't manage)

**Example:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-pre-delete-backup
  annotations:
    "helm.sh/hook": pre-delete
    "helm.sh/hook-weight": "-10"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: backup
          image: postgres:16-alpine
          env:
            - name: PGPASSWORD
              value: {{ .Values.database.password }}
          command:
            - sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              pg_dump -h {{ .Values.database.host }} -U {{ .Values.database.user }} \
                -d {{ .Values.database.name }} \
                > /backup/backup-${TIMESTAMP}.sql
              echo "Backup completed: backup-${TIMESTAMP}.sql"
```

### 4. `post-delete`

Executes **after** all resources in the release have been deleted.

**Annotation:** `helm.sh/hook: post-delete`

**Use cases:**
- Cleaning up persistent data (volumes, snapshots) that `helm uninstall` leaves behind
- Removing orphaned DNS records
- Sending deletion confirmation notifications
- Auditing / logging the deletion for compliance

**Example:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-post-delete-cleanup
  annotations:
    "helm.sh/hook": post-delete
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: cleanup
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Release {{ .Release.Name }} has been deleted."
              echo "Sending deletion notification..."
              # Example: curl -X POST https://hooks.slack.com/... \
              #   -d '{"text":"Release {{ .Release.Name }} deleted from {{ .Release.Namespace }}"}'
```

### 5. `pre-upgrade`

Executes **before** any resources are modified during an upgrade.

**Annotation:** `helm.sh/hook: pre-upgrade`

**Use cases:**
- Running database schema migrations before the new application version deploys
- Validating that the new chart values are correct
- Taking pre-upgrade snapshots of state
- Draining traffic from the current deployment

**Example:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: {{ .Values.migration.image }}:{{ .Values.migration.tag }}
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-db-secret
                  key: url
          command:
            - /app/migrate
            - up
```

### 6. `post-upgrade`

Executes **after** all resources have been upgraded (and are ready, if `--wait` is used).

**Annotation:** `helm.sh/hook: post-upgrade`

**Use cases:**
- Running post-migration data validation
- Sending upgrade notifications
- Invalidating CDN caches
- Running integration tests against the upgraded release
- Verifying that the new version is serving traffic correctly

**Example:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-post-upgrade-verify
  annotations:
    "helm.sh/hook": post-upgrade
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: verify
          image: curlimages/curl:8.4.0
          command:
            - sh
            - -c
            - |
              echo "Verifying upgraded release {{ .Release.Name }}"
              HEALTH_URL="http://{{ .Release.Name }}-service:{{ .Values.service.port }}/health"
              RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" $HEALTH_URL)
              if [ "$RESPONSE" != "200" ]; then
                echo "Health check failed with status $RESPONSE"
                exit 1
              fi
              echo "Verification passed."
```

### 7. `pre-rollback`

Executes **before** any resources are rolled back to a previous revision.

**Annotation:** `helm.sh/hook: pre-rollback`

**Use cases:**
- Backing up the current (failing) state for forensic analysis
- Notifying on-call of impending rollback
- Pausing traffic before the rollback begins

**Example:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-pre-rollback-snapshot
  annotations:
    "helm.sh/hook": pre-rollback
    "helm.sh/hook-weight": "-10"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: snapshot
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Capturing pre-rollback state for {{ .Release.Name }}"
              echo "Current revision before rollback: $(date --iso-8601=seconds)"
              kubectl get all -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }} > /tmp/state.txt
              echo "State snapshot captured."
```

### 8. `post-rollback`

Executes **after** all resources have been rolled back.

**Annotation:** `helm.sh/hook: post-rollback`

**Use cases:**
- Verifying that the rollback restored correct functionality
- Sending rollback notifications
- Running health checks against the restored version
- Logging the rollback event for audit

**Example:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-post-rollback-check
  annotations:
    "helm.sh/hook": post-rollback
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: verify-rollback
          image: curlimages/curl:8.4.0
          command:
            - sh
            - -c
            - |
              echo "Verifying rollback for {{ .Release.Name }}"
              HEALTH_URL="http://{{ .Release.Name }}-service:{{ .Values.service.port }}/health"
              for i in $(seq 1 10); do
                STATUS=$(curl -s -o /dev/null -w "%{http_code}" $HEALTH_URL)
                if [ "$STATUS" = "200" ]; then
                  echo "Rollback verified: service healthy."
                  exit 0
                fi
                echo "Attempt $i: status $STATUS, retrying..."
                sleep 3
              done
              echo "Rollback verification failed after 10 attempts."
              exit 1
```

### 9. `test`

Reserved for **Helm tests**. Unlike lifecycle hooks, `test` hooks are only executed when `helm test` is explicitly invoked — they do not run during `install`, `upgrade`, `delete`, or `rollback`.

**Annotation:** `helm.sh/hook: test`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-connectivity-test
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Testing connectivity to {{ .Release.Name }}"
          wget -q -O - http://{{ .Release.Name }}-service:{{ .Values.service.port }}/health
```

**Exam Tip:** `test` hooks are fundamentally different from lifecycle hooks. They are not triggered by `install`, `upgrade`, `rollback`, or `delete`. They only run when `helm test <release-name>` is executed.

## Hook Weights

When multiple hooks exist for the same lifecycle phase, Helm uses the `helm.sh/hook-weight` annotation to determine execution order.

### Rules

- Weights are **strings** parsed as integers.
- **Lower numbers execute first.** Think of it as ascending sort order.
- **Default weight is `"0"`** if not specified.
- **Hooks with the same weight execute in parallel.**
- **Negative weights are valid** (e.g., `"-10"`) and execute before default-weight hooks.

### Weight Ordering Example

```yaml
# Executes FIRST (weight -10)
metadata:
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-10"

# Executes SECOND (weight 0, default)
metadata:
  annotations:
    "helm.sh/hook": pre-install
    # "helm.sh/hook-weight" omitted → defaults to "0"

# Executes THIRD (weight 5)
metadata:
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "5"

# Executes in PARALLEL with the one above
metadata:
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "5"
```

### Visual Example

```
pre-install phase:
  Weight -10 → Job A executes
  Weight   0 → Job B executes (after A completes)
  Weight   5 → Job C and Job D execute in parallel (after B completes)

post-install phase: (same rules, independent ordering)
  Weight   0 → Job E executes
  Weight  10 → Job F executes (after E completes)
```

**Production Note:** Use negative weights for prerequisites (e.g., migrations before deployment) and positive weights for follow-up tasks (e.g., notifications after verification). This makes the relative ordering intuitive.

## Delete Policies

Hook resources are NOT automatically cleaned up. By default, they persist in the cluster until manually deleted. The `helm.sh/hook-delete-policy` annotation controls cleanup behavior.

### 1. `before-hook-creation`

Deletes the **previous** instance of the hook before creating a new one. If no previous hook exists, nothing is deleted.

**Use case:** Ensuring only one instance of a hook resource exists at any time.

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation
```

**Warning:** If a hook is still running when a new release operation begins, `before-hook-creation` deletes the running hook and starts a new one. This can leave operations in an incomplete state.

### 2. `hook-succeeded`

Deletes the hook resource **after** it completes successfully.

**Use case:** Cleaning up successful hook resources to reduce cluster clutter. Failed hooks are preserved for debugging.

```yaml
metadata:
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-delete-policy": hook-succeeded
```

### 3. `hook-failed`

Deletes the hook resource **after** it fails. Successful hooks are preserved.

**Use case:** Cleaning up failed hook resources when the failure is expected and the resource does not need debugging. Rarely used alone.

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-delete-policy": hook-failed
```

### 4. Combining Policies

Multiple policies can be specified as a comma-separated list:

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

**This means:** Delete the previous hook before creating the new one; if the new hook succeeds, delete it too.

### Delete Policy Summary Table

| Policy | When Deletion Occurs | Best For |
|---|---|---|
| `before-hook-creation` | Before new hook runs | Avoiding duplicate hook resources |
| `hook-succeeded` | After successful completion | Keeping cluster clean |
| `hook-failed` | After failure | Auto-cleaning expected failures |
| Comma-separated combo | At each matching trigger | Full lifecycle management |

**Production Note:** Always use `before-hook-creation` for hooks that may run multiple times (e.g., on repeated upgrades). Without it, each upgrade creates a new hook resource and the cluster accumulates stale Jobs. Combine it with `hook-succeeded` for full cleanup.

## Hook Resource Types

Not all Kubernetes resources behave the same way as hooks. Helm treats hooks by waiting for them to reach a terminal state.

### Supported Resource Types

| Resource Type | Terminal State | Behavior |
|---|---|---|
| **Job** | `Complete` or `Failed` | **Recommended.** Helm waits for the Job to finish and checks its status. |
| **Pod** | `Succeeded` or `Failed` | Supported. Helm waits for the Pod to reach a terminal phase. RestartPolicy must be `Never` or `OnFailure`. |
| **ConfigMap** | N/A (immutable) | Creates the ConfigMap but does NOT wait for any process. Use for providing configuration to other hook resources, not for executing logic. |
| **Secret** | N/A (immutable) | Same as ConfigMap — created but not waited on. |

### What Does NOT Work as a Hook

- **Deployments / StatefulSets / DaemonSets:** These never reach a terminal state; Helm waits indefinitely.
- **Services / Ingresses:** No terminal state; Helm does not wait.
- **PersistentVolumeClaims:** No terminal state.
- **Custom Resources (CRDs):** Depends on the operator; Helm cannot determine a terminal state for arbitrary CRDs.

**Note:** Jobs are the recommended resource type for hooks because they have clear success/failure semantics and Helm can correctly determine when they complete.

## Full Production Scenarios

### Scenario 1: Database Migration (pre-upgrade)

A common pattern: run schema migrations before a new application version deploys.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate-{{ .Release.Revision }}
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  ttlSecondsAfterFinished: 300
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: {{ .Release.Name }}-migration-sa
      containers:
        - name: migrate
          image: {{ .Values.migration.image.repository }}:{{ .Values.migration.image.tag }}
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.database.secretName }}
                  key: host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.database.secretName }}
                  key: password
          command: ["/app/migrate", "up"]
          resources:
            limits:
              cpu: 500m
              memory: 256Mi
            requests:
              cpu: 250m
              memory: 128Mi
```

**Why this pattern works:**
- `pre-upgrade` ensures migration runs before the new application version starts.
- Negative weight (`-5`) means it runs before other pre-upgrade hooks.
- `before-hook-creation` cleans up the previous migration Job.
- `ttlSecondsAfterFinished` auto-deletes the completed Job after 5 minutes.
- `backoffLimit: 3` prevents infinite retry loops on failure.

### Scenario 2: Cleanup Before Deletion (pre-delete)

Ensures no data is lost when uninstalling a stateful application.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-pre-delete-drain
  annotations:
    "helm.sh/hook": pre-delete
    "helm.sh/hook-weight": "-10"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: drain
          image: bitnami/kubectl:1.28
          command:
            - sh
            - -c
            - |
              echo "Draining connections from {{ .Release.Name }}"
              kubectl label svc {{ .Release.Name }}-service -n {{ .Release.Namespace }} \
                app.kubernetes.io/drain=true --overwrite
              sleep 30  # Allow traffic to drain
              echo "Drain complete."
---
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-pre-delete-backup
  annotations:
    "helm.sh/hook": pre-delete
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: backup
          image: amazon/aws-cli:2.15.0
          env:
            - name: AWS_REGION
              value: "us-east-1"
          command:
            - sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              echo "Taking final backup before deletion..."
              kubectl exec -n {{ .Release.Namespace }} deploy/{{ .Release.Name }} \
                -- pg_dump -U app > /tmp/backup-${TIMESTAMP}.sql
              aws s3 cp /tmp/backup-${TIMESTAMP}.sql \
                s3://my-backups/{{ .Release.Name }}/backup-${TIMESTAMP}.sql
              echo "Backup stored in S3."
```

### Scenario 3: Smoke Test (post-install)

Validates that the deployment is healthy and serving traffic before considering it successful.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-smoke-test
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  backoffLimit: 5
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: smoke-test
          image: curlimages/curl:8.4.0
          command:
            - sh
            - -c
            - |
              set -e
              BASE_URL="http://{{ .Release.Name }}-service:{{ .Values.service.port }}"

              echo "=== Smoke Test: {{ .Release.Name }} v{{ .Chart.Version }} ==="

              echo "Test 1: Health endpoint"
              curl -f -s -o /dev/null -w "  Status: %{http_code}\n" $BASE_URL/health || exit 1

              echo "Test 2: Readiness endpoint"
              curl -f -s -o /dev/null -w "  Status: %{http_code}\n" $BASE_URL/ready || exit 1

              echo "Test 3: API version endpoint"
              RESPONSE=$(curl -f -s $BASE_URL/api/version)
              echo "  Response: $RESPONSE"
              echo "$RESPONSE" | grep -q '"status":"ok"' || exit 1

              echo "=== All smoke tests passed ==="
```

### Scenario 4: Notification (post-install / post-upgrade)

Sends deployment notifications to a chat platform.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-notify
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "10"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: notify
          image: curlimages/curl:8.4.0
          command:
            - sh
            - -c
            - |
              ACTION="{{ if .Release.IsInstall }}installed{{ else }}upgraded{{ end }}"
              VERSION="{{ .Chart.Version }}"
              REVISION="{{ .Release.Revision }}"
              NAMESPACE="{{ .Release.Namespace }}"

              curl -X POST {{ .Values.notifications.webhookUrl }} \
                -H "Content-Type: application/json" \
                -d "{
                  \"text\": \"Helm release \`${RELEASE}\` ${ACTION} (v${VERSION}, rev ${REVISION}) in \`${NAMESPACE}\`\"
                }"
          env:
            - name: RELEASE
              value: "{{ .Release.Name }}"
```

**Exam Tip:** Notice the hook is annotated with **two** hook types: `post-install,post-upgrade`. This means the same hook resource runs on both install and upgrade events. This is a common pattern for notifications.

## Hook Debugging

### Checking Hook Status

```bash
# List all hook resources for a release
kubectl get jobs,pods -n <namespace> -l helm.sh/chart=<chart-name>

# Describe a specific hook Job
kubectl describe job <release-name>-<hook-name> -n <namespace>

# Check hook pod logs
kubectl logs job/<release-name>-<hook-name> -n <namespace>
```

### Common Debugging Commands

```bash
# Get all hook pods with their status
kubectl get pods -n <namespace> \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,HOOK:.metadata.annotations.helm\.sh/hook

# View events for a stuck hook
kubectl get events -n <namespace> --field-selector involvedObject.name=<hook-job-name>

# Follow logs of a running or stuck hook
kubectl logs -f job/<release-name>-<hook-name> -n <namespace>
```

### Hook That Never Completes

If a hook is stuck:

1. **Check the pod status:**
   ```bash
   kubectl get pods -n <namespace> | grep <release-name>
   ```

2. **Describe for scheduling issues:**
   ```bash
   kubectl describe pod <hook-pod-name> -n <namespace>
   ```
   Look for: `ImagePullBackOff`, `CrashLoopBackOff`, `Pending` (resource constraints), `ErrImagePull`.

3. **Check resource limits:** Hooks are Jobs/Pods that need CPU and memory. If cluster resources are exhausted, hooks stay in `Pending`.

4. **Check RBAC:** The hook's ServiceAccount may lack permissions for the operations it tries to perform.

5. **Check network policies:** Hooks may be unable to reach the application service or external APIs due to NetworkPolicy restrictions.

**Note:** A stuck hook blocks the entire Helm operation. `helm install --timeout 5m` will eventually fail with a timeout error if the hook doesn't complete.

## Hook Limitations

1. **No guarantees during a failed rollback:** If `helm rollback` itself fails (e.g., because release secrets are corrupted), hooks defined for `pre-rollback` and `post-rollback` will NOT execute. Helm does not run hooks when the rollback operation cannot be initiated.

2. **Hooks are not retried automatically.** If a pre-upgrade hook fails, Helm does NOT retry it. The operator must fix the issue and re-run `helm upgrade`.

3. **Hook resources are not included in `helm template` output.** They are only evaluated during actual release operations.

4. **No hook for `pre-template` or `post-template`.** Hooks only fire during Helm's lifecycle operations, not during template rendering or linting.

5. **Hooks cannot depend on other hooks within the same weight.** All hooks with the same weight execute in parallel with no ordering guarantee.

6. **Secret and ConfigMap hooks do not block.** Helm creates them and continues immediately. Use Jobs for any logic that requires waiting.

7. **Helm does not delete hook secrets automatically.** If hooks use `imagePullSecrets`, those secrets persist after the hook completes unless managed by a delete policy.

8. **Cross-namespace hooks are not supported.** Hooks execute in the release namespace.

## Common Mistakes

### 1. Hooks That Never Complete

**Problem:** A Job hook runs indefinitely or never reaches a terminal state.

**Root cause:** Missing `restartPolicy: Never` in the Pod spec, an infinite loop in the container command, or a container that hangs waiting for a resource that never becomes available.

**Fix:**
```yaml
spec:
  template:
    spec:
      restartPolicy: Never   # Required for Jobs
      containers:
        - name: my-hook
          command: ["/app/hook-script.sh"]
          command: # Script includes a timeout mechanism
```

Add a timeout inside the script:
```bash
timeout 300 /app/hook-script.sh || { echo "Hook timed out"; exit 1; }
```

### 2. Missing Delete Policies

**Problem:** Hook resources accumulate in the cluster after each install/upgrade.

**Root cause:** The `helm.sh/hook-delete-policy` annotation is omitted.

**Fix:**
```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

### 3. Wrong Weight Ordering

**Problem:** A database migration (`pre-upgrade, weight 0`) runs AFTER a configuration update (`pre-upgrade, weight -5`) that depends on the new schema.

**Fix:**
```yaml
# Migration: must run first
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-10"  # lower number = runs first

# Config initialization: runs after migration
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
```

### 4. Using Deployments as Hooks

**Problem:** A Deployment annotated with `helm.sh/hook: post-install` never completes, blocking the release.

**Fix:** Replace Deployments with Jobs. Deployments manage long-running processes and never reach `Complete`. Jobs have clear terminal states.

### 5. Not Setting `backoffLimit`

**Problem:** A failing hook retries indefinitely until Helm's `--timeout` is reached.

**Fix:** Set a reasonable `backoffLimit` on hook Jobs:
```yaml
spec:
  backoffLimit: 3   # Only retry 3 times, then fail
```

### 6. Forgetting `--wait` with Post-Install Hooks

**Problem:** A `post-install` hook tests the application's health endpoint, but the application pods are not yet ready.

**Fix:** Use `helm install --wait` so that core resources are ready before post-install hooks execute:
```bash
helm install myapp ./chart --namespace prod --wait --timeout 10m
```

### 7. Hardcoding Names in Hook Templates

**Problem:** Multiple releases of the same chart produce hook name collisions.

**Fix:** Always include `{{ .Release.Name }}` and `{{ .Release.Revision }}` in hook resource names:
```yaml
metadata:
  name: {{ .Release.Name }}-db-migrate-{{ .Release.Revision }}
```

### 8. Assuming Hooks Run During `helm template`

**Problem:** Developers expect hook resources to appear in `helm template` output.

**Reality:** `helm template` renders all templates but does not execute hooks. Hook annotations are rendered in the output, but hooks only execute during `install`, `upgrade`, `delete`, `rollback`, or `test`.
