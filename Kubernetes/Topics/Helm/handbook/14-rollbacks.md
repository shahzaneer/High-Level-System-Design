# Chapter 14: Rollbacks

Helm rollbacks are the primary mechanism for reverting a release to a previous working state. This chapter covers the internals, commands, strategies, and best practices for performing safe and effective rollbacks in production environments.

---

## What Is a Rollback

A rollback reverts a Helm release to a previously deployed revision. When an upgrade introduces regressions, misconfigurations, or failures, a rollback restores the cluster to a known-good state by re-applying the manifest from an earlier revision.

```
+------------------+         +------------------+         +------------------+
|   Revision 1     |         |   Revision 2     |         |   Revision 3     |
|   (working)      |-------->|   (broken)       |-------->|   (rollback to   |
|   deployed       | upgrade |   deployed       | rollback|   revision 1)    |
+------------------+         +------------------+         +------------------+
                                    |                            ^
                                    |   helm rollback            |
                                    +----------------------------+
```

**Note:** A rollback creates a *new* revision in the release history. It does not delete the broken revision. Rolling back from revision 3 to revision 1 produces revision 4, whose manifest is identical to revision 1.

---

## How Rollback Works Internally

Helm rollback follows a precise internal process:

```
Step 1: User runs "helm rollback my-release 1"
         |
Step 2: Helm reads the release secret for revision 1
         |    sh.helm.release.v1.my-release.v1
         |    -> decodes base64 -> decompresses gzip -> parses JSON
         |
Step 3: Extracts the stored manifest from revision 1
         |    (This is the multi-document YAML that was applied at revision 1)
         |
Step 4: Re-renders the current chart with original values (revision 1 values)
         |    OR uses the stored manifest directly, depending on Helm version
         |
Step 5: Computes a three-way strategic merge patch:
         |    Old = revision 1 manifest (desired)
         |    New = revision 3 manifest (current)
         |    Live = actual cluster state
         |
Step 6: Applies the patch to the cluster via Kubernetes API
         |
Step 7: Creates a new release secret (revision 4)
         |    Contains: manifest from revision 1, current chart metadata,
         |    timestamp, and status "deployed"
         |
Step 8: Marks revision 4 as deployed, supersedes revision 3
```

**Key insight:** The rollback does not "undo" the broken upgrade by applying the reverse diff. It re-applies the full desired state from the target revision as a new revision, creating a clean forward movement in history.

---

## helm rollback — Full Syntax and Flags

```bash
helm rollback <RELEASE> [REVISION] [flags]
```

### All Flags

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `RELEASE` | string (arg) | (required) | The name of the release to roll back. |
| `REVISION` | int (arg) | 0 | The revision number to roll back to. If 0, rolls back to the previous revision (current - 1). |
| `--cleanup-on-fail` | bool | false | Delete newly created resources if the rollback fails. Prevents partial rollback state. |
| `--dry-run` | bool | false | Simulate the rollback. Renders and validates manifests but does not apply them. |
| `--force` | bool | false | Force resource updates through delete/recreate when needed (e.g., immutable field changes between revisions). |
| `--history-max` | int | 10 | Maximum number of revisions retained per release. Older revisions beyond this limit are deleted during the rollback. |
| `--no-hooks` | bool | false | Prevent pre-rollback and post-rollback hooks from executing. |
| `--recreate-pods` | bool | false | Perform a pod recreate instead of a rolling update (causes downtime). |
| `--timeout` | duration | 5m0s | Maximum time to wait for Kubernetes operations to complete during the rollback. |
| `--wait` | bool | false | Wait until all Pods, PVCs, and Services are in a ready state before marking the rollback as complete. |
| `--wait-for-jobs` | bool | false | Also wait for Jobs to complete when `--wait` is used. |
| `--namespace` | string | current | The namespace of the release. |
| `--kube-context` | string | current | The Kubernetes context to use. |

### Examples

```bash
# Rollback to the previous revision
helm rollback my-release

# Rollback to a specific revision
helm rollback my-release 5

# Dry-run a rollback to preview what will happen
helm rollback my-release 3 --dry-run

# Rollback with cleanup on failure (safe for production)
helm rollback my-release 2 --cleanup-on-fail --wait --timeout 10m

# Rollback but skip hooks (hooks may also be broken)
helm rollback my-release 1 --no-hooks

# Rollback and force pod recreation if needed
helm rollback my-release 1 --force --recreate-pods

# Rollback with history limit to clean old revisions
helm rollback my-release 1 --history-max 5
```

**Production Note:** Always use `--cleanup-on-fail --wait --timeout` together for production rollbacks. This ensures the rollback either fully succeeds or leaves no partial state, and you know whether the operation completed within a reasonable time.

---

## Revision History

### How Revisions Are Stored

Every Helm release stores its revision history as Kubernetes Secrets in the release's namespace:

```bash
# List all revisions for a release
kubectl get secrets -l owner=helm,name=my-release -n default

# Sample output:
# NAME                                     TYPE                 DATA   AGE
# sh.helm.release.v1.my-release.v1         helm.sh/release.v1   1      7d
# sh.helm.release.v1.my-release.v2         helm.sh/release.v1   1      5d
# sh.helm.release.v1.my-release.v3         helm.sh/release.v1   1      2d
# sh.helm.release.v1.my-release.v4         helm.sh/release.v1   1      1h
```

### Viewing History

```bash
# Standard history view
helm history my-release

# Extended output in YAML
helm history my-release --output yaml

# Limit the number of revisions displayed
helm history my-release --max 20

# Filter to a specific namespace
helm history my-release --namespace production
```

### Revision Numbering

Revisions are monotonic integers starting at 1:

| Operation | New Revision | Old Revisions |
|-----------|-------------|---------------|
| `helm install` | 1 | None |
| `helm upgrade` | Previous + 1 | All previous retained (up to history-max) |
| `helm rollback` | Previous + 1 | All previous retained (including the rolled-back-from revision) |

**Note:** Revisions are never reused. Even if you delete old revision secrets, the counter continues incrementing. This ensures uniqueness and prevents ambiguity.

---

## Revision Storage Format

The `sh.helm.release.v1` secrets contain a base64-encoded, gzipped JSON blob inside another base64 encoding.

### Inspecting a Revision Secret

```bash
# Decode and inspect a revision
kubectl get secret sh.helm.release.v1.my-release.v3 -n default \
  -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip | jq .

# The decoded JSON structure:
```

```json
{
  "name": "my-release",
  "info": {
    "first_deployed": "2024-01-15T10:30:00Z",
    "last_deployed": "2024-01-15T11:30:00Z",
    "deleted": "",
    "description": "Upgrade complete",
    "status": "deployed"
  },
  "chart": {
    "metadata": {
      "name": "mychart",
      "version": "1.2.0",
      "appVersion": "1.26.0",
      "apiVersion": "v2"
    },
    "values": {},
    "templates": [...],
    "files": [...]
  },
  "config": {
    "replicaCount": 3,
    "image": {
      "repository": "nginx",
      "tag": "1.25"
    },
    "service": {
      "type": "ClusterIP",
      "port": 80
    }
  },
  "manifest": "---\n# Source: mychart/templates/deployment.yaml\n...",
  "version": 1
}
```

| Field | Description |
|-------|-------------|
| `name` | Release name. |
| `info.status` | Current status: `deployed`, `superseded`, `failed`, `pending-*`, `uninstalling`. |
| `info.first_deployed` | Timestamp of the initial install. Same across all revisions of a release. |
| `info.last_deployed` | Timestamp of this revision's operation. |
| `info.description` | Human-readable result: "Install complete", "Upgrade complete", "Rollback to revision X". |
| `chart` | The Chart.yaml metadata, default values, all template files, and chart files. |
| `config` | The full merged values used to render this revision (user overrides + defaults). |
| `manifest` | The multi-document YAML manifest that was applied to the cluster. |
| `version` | Internal Helm release object version (not the revision number). |

**Warning:** Release secrets contain the full chart (all files) and all values. If your values contain secrets, they are stored **unencrypted** in the release secret. Use Kubernetes Secrets or external secret management (e.g., Vault, Sealed Secrets) for sensitive data instead of passing them via `--set` or values files.

---

## Setting History Limits

```bash
# Set history limit at install time
helm install my-release ./mychart --history-max 5

# Set history limit at upgrade time
helm upgrade my-release ./mychart --history-max 5

# Set history limit at rollback time
helm rollback my-release 2 --history-max 5

# Set history limit as a default via environment variable
export HELM_MAX_HISTORY=10
```

### Cleanup Behavior

When a new revision is created and the total exceeds `--history-max`:
1. Helm identifies the oldest revision(s) beyond the limit.
2. Those release secrets are deleted from the cluster.
3. The chart's `charts/` directory cleanup also respects this limit.

```
Before (history-max=5):
  v1, v2, v3, v4, v5, v6 (6 revisions)

After v7 is created:
  v2, v3, v4, v5, v6, v7 (v1 deleted)
```

**Note:** The currently deployed revision and the immediately preceding revision are **never** deleted, even if they exceed `history-max`. This guarantees that a rollback to the previous revision is always possible as long as the previous revision's secret exists.

---

## Rollback Process Step-by-Step

```
+-----------------------------------------------------------------------------------------+
|                           HELM ROLLBACK -- END-TO-END FLOW                              |
+-----------------------------------------------------------------------------------------+
|                                                                                          |
|  USER: helm rollback my-release 3                                                        |
|     |                                                                                    |
|     v                                                                                    |
|  +----------------------------------------+                                              |
|  | 1. VALIDATE REVISION                   |                                              |
|  |    - Check revision 3 exists           |                                              |
|  |    - Verify release secret is present  |                                              |
|  |    - Verify release is not in          |                                              |
|  |      pending state (blocked)           |                                              |
|  +-------------------+--------------------+                                              |
|                      |                                                                   |
|                      v                                                                   |
|  +----------------------------------------+                                              |
|  | 2. EXTRACT REVISION DATA              |                                              |
|  |    - Get sh.helm.release.v1.<name>.v3 |                                              |
|  |    - Decode base64 -> decompress gzip  |                                              |
|  |    - Parse manifest and config         |                                              |
|  +-------------------+--------------------+                                              |
|                      |                                                                   |
|                      v                                                                   |
|  +----------------------------------------+                                              |
|  | 3. EXECUTE PRE-ROLLBACK HOOKS         |                                              |
|  |    (if --no-hooks is not set)          |                                              |
|  |    - Run hooks with "pre-rollback"     |                                              |
|  |    - Execute in weight order            |                                              |
|  |    - Wait for completion/timeout        |                                              |
|  +-------------------+--------------------+                                              |
|                      |                                                                   |
|                      |  HOOK FAILED?  -----> Rollback fails (unless --no-hooks)          |
|                      |                                                                   |
|                      v                                                                   |
|  +----------------------------------------+                                              |
|  | 4. APPLY MANIFEST TO CLUSTER          |                                              |
|  |    - Three-way merge:                  |                                              |
|  |      Old = v3 manifest (current)       |                                              |
|  |      New = target manifest (v3 data)   |                                              |
|  |      Live = actual cluster state       |                                              |
|  |    - Apply patch via Kubernetes API    |                                              |
|  +-------------------+--------------------+                                              |
|                      |                                                                   |
|                      |  APPLY FAILED?  -----> Rollback fails                             |
|                      |                        (--cleanup-on-fail removes new resources)  |
|                      v                                                                   |
|  +----------------------------------------+                                              |
|  | 5. WAIT FOR RESOURCES (--wait)        |                                              |
|  |    - Poll Deployments, StatefulSets    |                                              |
|  |    - Check Pod readiness               |                                              |
|  |    - Wait for Jobs (--wait-for-jobs)   |                                              |
|  |    - Respect --timeout                  |                                              |
|  +-------------------+--------------------+                                              |
|                      |                                                                   |
|                      |  TIMEOUT?  -----> Rollback fails                                  |
|                      |                                                                   |
|                      v                                                                   |
|  +----------------------------------------+                                              |
|  | 6. EXECUTE POST-ROLLBACK HOOKS        |                                              |
|  |    (if --no-hooks is not set)          |                                              |
|  +-------------------+--------------------+                                              |
|                      |                                                                   |
|                      v                                                                   |
|  +----------------------------------------+                                              |
|  | 7. PERSIST NEW REVISION               |                                              |
|  |    - Create sh.helm.release.v1.<n>.vN  |                                              |
|  |    - Mark as "deployed"                 |                                              |
|  |    - Supersede previous revisions       |                                              |
|  |    - Enforce --history-max cleanup      |                                              |
|  +----------------------------------------+                                              |
|                                                                                          |
+-----------------------------------------------------------------------------------------+
```

---

## Revision Cleanup Strategies

### Automatic Cleanup via --history-max

```bash
# Conservative: keep 10 revisions (default)
helm install my-release ./mychart --history-max 10

# Aggressive: keep only 2 revisions (current + previous)
helm install my-release ./mychart --history-max 2

# Per-upgrade override
helm upgrade my-release ./mychart --history-max 20
```

### Manual Cleanup

```bash
# Delete all superseded secrets for a release
kubectl delete secret -l owner=helm,name=my-release,status=superseded \
  --namespace default

# Delete a specific revision's secret (dangerous: breaks rollback to that revision)
kubectl delete secret sh.helm.release.v1.my-release.v5 -n default

# Delete ALL release secrets for a release (clean slate, but lose all history)
kubectl delete secret -l owner=helm,name=my-release -n default
```

**Warning:** Deleting the secret for the currently deployed revision will prevent future rollbacks. Helm cannot revert to a revision whose secret has been deleted.

---

## Recovering from Failed Upgrades

```
+---------------------------+     +---------------------------+     +---------------------------+
| 1. DETECT                 |     | 2. IDENTIFY               |     | 3. DECIDE                 |
| helm status shows failed  |---->| helm history shows which  |---->| Rollback or fix-forward?  |
| kubectl shows CrashLoop   |     | revision introduced issue |     |                           |
+---------------------------+     +---------------------------+     +-----------+---------------+
                                                                               |
                                                          +--------------------+--------------------+
                                                          |                                         |
                                                          v                                         v
                                               +---------------------+                   +----------------------+
                                               | ROLLBACK            |                   | FIX-FORWARD          |
                                               | helm rollback       |                   | Fix chart, then      |
                                               | to previous rev     |                   | helm upgrade again   |
                                               +---------------------+                   +----------------------+
                                                          |                                         |
                                                          v                                         v
                                               +---------------------+                   +----------------------+
                                               | VERIFY              |                   | VERIFY               |
                                               | helm test, describe |                   | helm test, describe  |
                                               +---------------------+                   +----------------------+
```

### Recovery Commands

```bash
# 1. Check the current state
helm status my-release
helm history my-release

# 2. Check what the failed revision contains
helm get manifest my-release --revision 3
helm get values my-release --revision 3

# 3. Rollback
helm rollback my-release 2 --wait --timeout 5m

# 4. Verify
helm test my-release
helm status my-release
kubectl get pods -l app.kubernetes.io/instance=my-release
```

---

## Production Rollback Strategy — Decision Matrix

| Scenario | Rollback | Fix-Forward | Rationale |
|----------|----------|-------------|-----------|
| Upgrade fails immediately (pods don't start) | **Yes** | No | No value in keeping broken state. Roll back to restore service. |
| Upgrade succeeds but app has a minor bug | No | **Yes** | Rollback has overhead. Fix the bug and upgrade again. |
| Upgrade causes data corruption | **Yes** (immediately) | After rollback | Restore service first. Then fix the data issue and test before upgrading. |
| Upgrade passes all tests but has a performance regression | **Conditional** | **Preferred** | If performance is tolerable, fix-forward. If SLA is at risk, rollback. |
| Upgrade changes a database schema | **Conditional** | **Depends** | Schema changes may not be reversible. Have a migration rollback plan. |
| Upgrade changes an immutable field | N/A | **Use --force** | Rollback to a revision with different immutable fields also requires `--force`. |
| Upgrade was partially applied (some resources created, some failed) | **Yes** (--cleanup-on-fail) | No | Partial state is unpredictable. Roll back for consistency. |
| Current revision is unknown/corrupted | **Yes** | No | Rollback to the last known working revision. |
| Third-party chart upgrade introduces breaking changes | **Yes** | Fix chart values | Restore service. Then evaluate whether to stay on old version or adapt to the new chart. |

### Decision Flowchart

```
+--------------------------+
| Upgrade deployed         |
+------------+-------------+
             |
             v
+--------------------------+     YES     +-----------------------+
| Is the service down or   |------------>| ROLLBACK immediately  |
| degraded?                |             | (MTTR priority)       |
+------------+-------------+             +-----------------------+
             | NO
             v
+--------------------------+     YES     +-----------------------+
| Is there data loss or    |------------>| ROLLBACK immediately  |
| corruption?              |             | Run data recovery     |
+------------+-------------+             +-----------------------+
             | NO
             v
+--------------------------+     YES     +-----------------------+
| Can the issue be fixed   |------------>| FIX-FORWARD           |
| within SLA window?       |             | helm upgrade with fix |
+------------+-------------+             +-----------------------+
             | NO
             v
+--------------------------+
| ROLLBACK and fix later   |
+--------------------------+
```

---

## Rollback Hooks

Helm supports two hook types for rollbacks:

| Hook | Annotation | Execution Point |
|------|------------|-----------------|
| `pre-rollback` | `"helm.sh/hook": pre-rollback` | Executes **before** the rollback manifest is applied. |
| `post-rollback` | `"helm.sh/hook": post-rollback` | Executes **after** the rollback manifest is applied and resources are ready. |

### Hook Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-pre-rollback"
  annotations:
    "helm.sh/hook": pre-rollback
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pre-rollback
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Pre-rollback: taking a backup before reverting..."
              # Backup critical data
              echo "Backup complete."
```

### Hook Execution During Rollback

- Hooks defined in the **target revision's** chart are executed, not the current revision's hooks.
- If the target revision's chart has a `pre-rollback` hook, it runs before the manifest is applied.
- If the target revision's chart has a `post-rollback` hook, it runs after resources are ready.
- Hooks from the current (failed) revision are **not** executed during rollback.
- `--no-hooks` skips all hook execution.

**Exam Tip:** Hooks execute from the **target** revision, not the broken revision. This means if you've added hooks since the revision you're rolling back to, those newer hooks will not run.

---

## Rollback with Dependencies

### Subchart Rollback Behavior

When rolling back a parent chart, subcharts are rolled back to the versions that were bundled with the target revision:

```bash
# Parent chart at revision 3 has:
#   parent-chart 2.0.0
#   subchart-A 1.5.0
#   subchart-B 3.2.0

# After upgrading (revision 4):
#   parent-chart 3.0.0
#   subchart-A 2.0.0  (upgraded)
#   subchart-B 3.2.0  (unchanged)

# Rolling back to revision 3:
helm rollback my-release 3
# Results:
#   parent-chart 2.0.0
#   subchart-A 1.5.0  (rolled back to revision 3's version)
#   subchart-B 3.2.0  (unchanged)
```

**Warning:** Subchart rollbacks apply the full manifest from the target revision. This means any Kubernetes resources managed by the subchart will be reverted to the state they had at the target revision, including version downgrades.

### Parent Chart Rollback Only

There is no built-in way to rollback only the parent chart while keeping subcharts at their current version. The rollback is all-or-nothing: it restores the entire manifest from the target revision.

If you need to selectively rollback:
1. Rollback the entire release to the target revision.
2. Then upgrade again with updated subchart versions.
3. Or, manually apply changes with `kubectl` (not recommended).

---

## Rollback and Stateful Workloads

### StatefulSets

StatefulSets have special behavior during rollbacks:

```bash
# StatefulSet during rollback:
# - Pods are NOT automatically deleted. They are updated one-by-one
#   in reverse ordinal order (highest index first).
# - If you use --recreate-pods, Pods are deleted and recreated
#   (causes downtime, but can resolve stuck upgrades).
# - The StatefulSet controller ensures at-most-one semantics for
#   Pods with identity.

# Rollback without pod recreation
helm rollback my-release 1 --wait

# Rollback with forced pod recreation
helm rollback my-release 1 --recreate-pods
```

### PVCs During Rollback

```
PersistentVolumeClaims are NEVER deleted during rollback.
  |
  +-- This is by design. PVCs survive upgrades, rollbacks, and
  |   even uninstalls (unless the PVC annotation is set).
  |
  +-- If the rollback target revision used a different PVC template
      (e.g., different storage class or size), the existing PVC is
      NOT modified. You must handle PVC migrations manually.
```

**Production Note:** PVC immutability during rollbacks is a safety feature, not a bug. It prevents accidental data loss. If a rollback requires a different PVC configuration, you must plan a data migration separately from the Helm rollback operation.

### Data Safety Checklist

- [ ] Verify PVCs are not accidentally deleted (check reclaim policy: `Retain` is safest).
- [ ] Ensure `volumeClaimTemplates` in StatefulSets haven't changed fields that trigger rebuilds.
- [ ] Back up critical data before any rollback that involves StatefulSets.
- [ ] Test StatefulSet rollbacks in a staging environment that mirrors production PVC configuration.

---

## Zero-Downtime Rollback

### Strategies

| Strategy | How It Works | Downtime | Complexity |
|----------|-------------|----------|------------|
| Rolling update rollback | StatefulSets and Deployments update pods one at a time. | Minimal (one pod at a time) | Low |
| Blue-green with Helm | Deploy the old version alongside the new, switch traffic via Service selector. | Zero | Medium |
| Canary rollback | Shift traffic back from canary to stable. | Zero | High |
| `--recreate-pods` | Delete all pods, recreate from old manifest. | Full | Low |

### Zero-Downtime Rollback with Blue-Green

```bash
# Maintain two releases: my-app-blue and my-app-green
# my-app-blue = working (older) version
# my-app-green = new (possibly broken) version

# If green is broken:
# 1. Switch Service selector back to blue
kubectl patch service my-app -p '{"spec":{"selector":{"app":"my-app-blue"}}}'

# 2. Then rollback green at your leisure
helm rollback my-app-green 1

# 3. Switch back to green when fixed
kubectl patch service my-app -p '{"spec":{"selector":{"app":"my-app-green"}}}'
```

---

## Automated Rollback

### The --atomic Flag

`--atomic` is an install/upgrade flag that automatically rolls back on failure:

```bash
# Install with automatic rollback on failure
helm install my-release ./mychart --atomic

# Upgrade with automatic rollback on failure
helm upgrade my-release ./mychart --atomic --timeout 5m

# How --atomic works:
# 1. Install/Upgrade is attempted
# 2. If the operation fails:
#    a. All created/updated resources are deleted/reverted
#    b. The release returns to its previous state
#    c. The failed revision is NOT stored (unlike without --atomic)
```

**Note:** `--atomic` is equivalent to `--wait --cleanup-on-fail` plus automatic rollback on failure. It is the safest way to perform upgrades when you want guarantee that a failed upgrade leaves no trace.

### The --cleanup-on-fail Flag

Without `--atomic`, `--cleanup-on-fail` ensures that resources created during a failed install/upgrade/rollback are cleaned up:

```bash
helm upgrade my-release ./mychart --cleanup-on-fail
```

Without `--cleanup-on-fail`, failed operations may leave partially created resources:

```
Upgrade creates: Deployment (success), ConfigMap (success), Service (fails)
  Without --cleanup-on-fail:
    -> Deployment and ConfigMap remain (orphaned in new state)
  With --cleanup-on-fail:
    -> All new resources are deleted
    -> Cluster returns to pre-upgrade state
```

---

## Rollback to a Specific Revision

```bash
# Rollback to the immediately previous revision (revision N-1)
helm rollback my-release 0
# OR equivalently:
helm rollback my-release

# Rollback to a specific revision number
helm rollback my-release 7

# List revisions to identify the target
helm history my-release --max 20

# Dry-run to verify which revision's manifest will be applied
helm rollback my-release 7 --dry-run --debug

# Compare revision manifests before rollback
helm get manifest my-release --revision 7 > rev7.yaml
helm get manifest my-release --revision 3 > rev3.yaml
diff rev3.yaml rev7.yaml
```

**Exam Tip:** `helm rollback my-release 0` is a special case that rolls back to the previous revision (current revision - 1), regardless of the actual revision number. This is a common source of trick exam questions.

---

## Rollback Across Namespaces

Helm releases are namespace-scoped (in Helm 3):

```bash
# Rollback a release in a specific namespace
helm rollback my-release 3 --namespace production

# The release secret resides in that namespace
kubectl get secret -l owner=helm,name=my-release -n production

# Cluster-wide releases are not supported in Helm 3
# Each namespace has its own release namespace
```

---

## Common Rollback Mistakes

| Mistake | Consequence | Prevention |
|---------|------------|------------|
| Rolling back too many revisions (e.g., from 50 to 3) | Large state drift; many resources may need recreation. Potential for unexpected behavior. | Test the target revision's manifest first: `helm get manifest --revision 3`. |
| Forgetting --history-max | Runs out of revision history. Cannot rollback beyond the retained limit. | Set `--history-max` at install time. Default is 10, which is usually sufficient for production. |
| Using --recreate-pods unnecessarily | Causes downtime by deleting all pods before recreating them. | Only use when pods are stuck or the deployment strategy requires it. |
| Rolling back without --wait | The rollback is marked complete before pods are ready. May appear successful but service is down. | Always use `--wait` in production. |
| Not checking hooks | Pre/post-rollback hooks from the target revision may have side effects (e.g., database migrations in reverse). | Review hooks with `--dry-run` first. Use `--no-hooks` if hooks are problematic. |
| Assuming PVCs roll back | PVC spec changes are ignored. Old PVCs persist. | PVCs are immutable from Helm's perspective. Manage PVC changes manually. |
| Rollback of a subchart without understanding parent impact | The entire release (parent + all subcharts) rolls back. | There is no subchart-only rollback. Understand the blast radius. |
| Rolling back with missing RBAC | The target revision's chart may require RBAC that no longer exists or was modified. | Ensure RBAC manifests are present in the target revision. |

---

## Testing After Rollback

### Verification Steps

```bash
# 1. Check release status
helm status my-release

# Output should show "STATUS: deployed" and the correct revision number
# The NOTES.txt will indicate which revision is active

# 2. Run Helm tests
helm test my-release

# 3. Inspect actual resources
kubectl get all -l app.kubernetes.io/instance=my-release -n <namespace>

# 4. Check pod health
kubectl get pods -l app.kubernetes.io/instance=my-release -n <namespace>
kubectl describe deployment <name> -n <namespace>

# 5. Verify specific configurations
# e.g., check that the correct image tag is deployed
kubectl get deployment my-app -o jsonpath='{.spec.template.spec.containers[0].image}'

# 6. Check application logs
kubectl logs -l app.kubernetes.io/instance=my-release --tail=50

# 7. Confirm revision matches expectation
helm history my-release --max 1

# The latest revision should show:
# STATUS: deployed, DESCRIPTION: Rollback to revision X
```

### Automated Verification

```bash
#!/bin/bash
# post-rollback-verify.sh

RELEASE="my-release"
NAMESPACE="production"
TIMEOUT=300

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=${RELEASE} \
  -n ${NAMESPACE} \
  --timeout=${TIMEOUT}s || {
    echo "ERROR: Pods not ready within ${TIMEOUT}s"
    exit 1
}

# Run Helm tests
helm test ${RELEASE} -n ${NAMESPACE} --timeout ${TIMEOUT}s || {
    echo "ERROR: Helm tests failed"
    exit 1
}

# Verify the correct revision is deployed
CURRENT_REV=$(helm history ${RELEASE} -n ${NAMESPACE} --max 1 -o json | jq -r '.[0].revision')
EXPECTED_REV=$1
if [ "${CURRENT_REV}" != "${EXPECTED_REV}" ]; then
    echo "ERROR: Expected revision ${EXPECTED_REV} but found ${CURRENT_REV}"
    exit 1
fi

echo "Rollback to revision ${EXPECTED_REV} verified successfully"
```

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Rollback mechanism | Creates a new revision using the manifest from a previous revision. Not an undo. |
| Revision storage | Kubernetes Secrets (`sh.helm.release.v1.<name>.v<N>`) containing base64/gzipped JSON. |
| Revision numbering | Monotonic increment. Never reused. |
| History limits | Controlled by `--history-max`. Oldest revisions are deleted first. |
| PVC safety | PVCs are never modified or deleted during rollback. |
| Automated rollback | `--atomic` flag performs automatic rollback on install/upgrade failure. |
| Hook execution | Hooks run from the target revision's chart, not the current revision. |
| Zero-downtime | Achievable with rolling updates, blue-green deployments, or canary rollbacks. |
| Verification | Always run `helm test` and check pod readiness after rollback. |