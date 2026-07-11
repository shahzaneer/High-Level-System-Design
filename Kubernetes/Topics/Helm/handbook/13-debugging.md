# Chapter 13: Debugging Helm

Helm debugging requires a systematic approach combining Helm's built-in tools, Kubernetes inspection commands, and methodical troubleshooting. This chapter covers every major debugging scenario you will encounter in production and development.

---

## Systematic Debugging Approach

When a Helm operation fails, do not guess. Follow a five-step methodology:

```
+----------------+     +----------------+     +----------------+     +----------------+     +----------------+
|  1. IDENTIFY   |---->|  2. REPRODUCE  |---->|  3. ISOLATE    |---->|  4. FIX         |---->|  5. VERIFY     |
|  Understand    |     |  Can you make  |     |  Narrow down   |     |  Apply the      |     |  Confirm the    |
|  the error     |     |  it happen     |     |  the root      |     |  correction     |     |  fix resolves   |
|  message       |     |  again?        |     |  cause         |     |                 |     |  the issue      |
+----------------+     +----------------+     +----------------+     +----------------+     +----------------+
```

| Phase | Actions | Tools |
|-------|---------|-------|
| Identify | Read the full error message. Check exit codes. Note the affected resource, revision, and timestamp. | `helm status`, `helm history`, `kubectl get events` |
| Reproduce | Run with `--dry-run --debug`. Check if the issue is deterministic. | `helm template`, `helm install --dry-run` |
| Isolate | Remove subcharts, simplify values, test individual templates. | `helm template --show-only`, `fail` function |
| Fix | Apply the minimal change needed. Update values, templates, or RBAC. | Editor, `helm upgrade`, `kubectl edit` |
| Verify | Reinstall or upgrade. Run tests. Confirm resource state. | `helm test`, `kubectl describe`, `helm get manifest` |

**Exam Tip:** For CKA/CKAD/Helm certifications, always start troubleshooting with `helm status` and `helm history` before attempting fixes. This demonstrates the systematic approach examiners expect.

---

## helm lint — Template-Level Validation

`helm lint` validates a chart for issues before you ever touch a cluster. It is the first line of defense.

### What helm lint Checks

| Check | Description |
|-------|-------------|
| YAML syntax | Ensures `Chart.yaml`, `values.yaml`, and templates are valid YAML. Catches missing colons, bad indentation, and unquoted special characters. |
| Required fields | Verifies `Chart.yaml` has mandatory fields: `apiVersion`, `name`, `version`. |
| Template rendering | Renders all templates with default values. Reports template errors (undefined functions, missing values causing nil pointer panics). |
| Naming conventions | Flags release names that violate Kubernetes naming rules (RFC 1123 label: lowercase, alphanumeric, hyphens, max 63 chars). |
| Version format | Validates that `version` in `Chart.yaml` is a valid SemVer 2 string. |
| Deprecated APIs | Warns when templates reference Kubernetes API versions scheduled for removal (e.g., `extensions/v1beta1`). |
| Dependency references | Validates that dependencies listed in `Chart.yaml` resolve correctly. |
| Icon URLs | Warns if the `icon` field has an unreachable or malformed URL. |

### Usage and Flags

```bash
# Basic lint
helm lint ./mychart

# Strict mode — treats warnings as errors (exit code 1 on warnings)
helm lint --strict ./mychart

# Lint with specific values
helm lint ./mychart --values ./prod-values.yaml

# Lint multiple value files
helm lint ./mychart -f ./base.yaml -f ./overrides.yaml

# Quiet mode — only show errors (suppress "1 chart(s) linted" messages)
helm lint --quiet ./mychart

# Lint a chart with subcharts (recursive)
helm lint ./mychart --with-subcharts
```

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All charts linted successfully, no issues found. |
| 1 | Linting error detected (syntax error, missing required field, template render failure). |
| 2 | Non-linting error (file not found, network issue) — Helm 3.12+. |

**Production Note:** Always run `helm lint --strict --with-subcharts` in CI/CD pipelines before packaging. A chart that lints with warnings but passes may fail at deploy time due to deprecated APIs rejected by newer Kubernetes versions.

### Example Output

```
$ helm lint ./mychart
==> Linting ./mychart
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

$ helm lint --strict ./mychart
==> Linting ./mychart
[ERROR] Chart.yaml: version is required
[INFO] Chart.yaml: icon is recommended
[ERROR] templates/: template: mychart/templates/deployment.yaml:25:16: executing "mychart/templates/deployment.yaml" at <.Values.image.repo>: nil pointer evaluating interface {}.repo

Error: 1 chart(s) linted, 1 chart(s) failed
```

---

## helm template — Local Rendering Without a Cluster

`helm template` renders chart templates locally and prints the resulting Kubernetes manifests to stdout. It does **not** contact any cluster, making it ideal for CI/CD validation and template debugging.

### Core Usage

```bash
# Render entire chart with default values
helm template my-release ./mychart

# Render with specific values
helm template my-release ./mychart --values ./staging.yaml

# Render and write to file for inspection
helm template my-release ./mychart > rendered.yaml

# Compare two value sets
helm template my-release ./mychart -f values-a.yaml > output-a.yaml
helm template my-release ./mychart -f values-b.yaml > output-b.yaml
diff output-a.yaml output-b.yaml
```

### Key Flags

| Flag | Description |
|------|-------------|
| `--show-only <path>` | Render only the specified template file(s). Accepts glob patterns. Example: `--show-only templates/deployment.yaml` or `--show-only templates/*.yaml` |
| `--validate` | Validate rendered manifests against the target Kubernetes version's schema. Requires connectivity to a cluster or specified `--kube-version`. |
| `--api-versions <versions>` | Comma-separated list of Kubernetes API versions available for `.Capabilities.APIVersions`. Critical for templates that use `{{ .Capabilities.APIVersions.Has }}`. |
| `--kube-version <version>` | Specify a Kubernetes version for `.Capabilities.KubeVersion`. Example: `--kube-version 1.29` |
| `--include-crds` | Include CRD manifests from the `crds/` directory in the output. By default, `helm template` skips CRDs. |
| `--skip-tests` | Exclude test pod templates from the output. |
| `--is-upgrade` | Render as if performing an upgrade (affects `.Release.IsUpgrade` and `.Release.IsInstall` in templates). |
| `--debug` | Print debug logs alongside the rendered output. Shows the server-side values being used. |
| `--no-hooks` | Exclude hook templates from the output. |

**Warning:** `helm template` output is **not** identical to `helm install --dry-run`. Template does not interact with the Kubernetes API, so it cannot validate API schemas, check resource quotas, or resolve admission webhook behavior. Always follow local template testing with `--dry-run` against a real or ephemeral cluster.

### Debug Template Logic

```bash
# Render a single template to debug its output
helm template my-release ./mychart --show-only templates/deployment.yaml

# Check what .Release looks like
helm template my-release ./mychart --show-only templates/_helpers.tpl --debug 2>&1 | head -50

# Test conditional branches by providing different values
helm template my-release ./mychart \
  --set ingress.enabled=true \
  --show-only templates/ingress.yaml
```

---

## helm install --dry-run — Cluster-Aware Validation

`--dry-run` renders templates and submits them to the Kubernetes API server for validation, but does **not** create any resources.

```bash
# Validate an install against the cluster
helm install my-release ./mychart --dry-run

# Verbose mode: prints the full rendered manifest + server debug logs
helm install my-release ./mychart --dry-run --debug
```

**Production Note:** `--dry-run --debug` is the most powerful pre-deployment check. It reveals:
- Template rendering errors with exact line numbers
- Kubernetes API validation errors (schema violations)
- Admission webhook rejections
- RBAC permission errors (if the user lacks CREATE access)

The output includes `[debug]` lines showing the computed values and the rendered YAML for each resource.

### What --dry-run Does NOT Catch

- Resource quota limits (quotas are checked at creation time, but dry-run skips creation)
- Actual pod scheduling failures
- Image pull issues
- Runtime application errors

---

## helm upgrade --dry-run — Preview Upgrade Changes

Before upgrading, preview exactly what will change:

```bash
helm upgrade my-release ./mychart --dry-run --debug
```

This reveals:
- Resources that will be **created** (not present in previous revision)
- Resources that will be **updated** (present in both, manifest differs)
- Resources that will be **deleted** (present in previous revision, absent in new chart)
- Kubernetes-specific validation errors on the updated manifests

**Note:** Helm 3 uses a three-way strategic merge patch (old manifest, new manifest, live state) to compute changes. `--dry-run` for upgrades shows the *desired* new manifest, not the merge result. To see the actual merge, use `helm diff upgrade` from the `helm-diff` plugin.

---

## helm get — Inspecting Deployed State

The `helm get` family of commands retrieves information about deployed releases.

### helm get manifest

Retrieves the Kubernetes manifest that was applied for a given revision:

```bash
# Get manifest of the latest deployed revision
helm get manifest my-release

# Get manifest of a specific revision
helm get manifest my-release --revision 3

# Get manifest of a release in another namespace
helm get manifest my-release --namespace production
```

Use this to see exactly what Kubernetes resources exist, verify that a previous deployment applied correctly, or compare the rendered template output against what the cluster received.

### helm get values

Shows computed values (defaults merged with user overrides) used for the release:

```bash
# Show all computed values
helm get values my-release

# Show all values including defaults (--all)
helm get values my-release --all

# Show values as YAML (default is a flat table)
helm get values my-release --output yaml

# Show values for a specific revision
helm get values my-release --revision 4
```

**Exam Tip:** `helm get values --all` reveals the full merged values, including defaults from `values.yaml` that were not explicitly overridden. This is essential for debugging "where did this value come from?" scenarios.

### helm get notes

Displays the rendered `NOTES.txt` for the release:

```bash
helm get notes my-release
```

### helm get hooks

Lists all hooks defined for the release and their status:

```bash
helm get hooks my-release
```

---

## helm history — Revision Tracking

```bash
# View revision history
helm history my-release

# Sample output:
# REVISION  UPDATED                   STATUS      CHART           APP VERSION  DESCRIPTION
# 1         Mon Jan 15 10:30:00 2024  superseded  mychart-1.0.0   1.0.0        Install complete
# 2         Mon Jan 15 11:00:00 2024  superseded  mychart-1.1.0   1.1.0        Upgrade complete
# 3         Mon Jan 15 11:30:00 2024  deployed    mychart-1.2.0   1.2.0        Upgrade complete

# Show history in YAML for parsing
helm history my-release --output yaml

# Maximum revisions to display
helm history my-release --max 10
```

The STATUS column reveals the revision state:

| Status | Meaning |
|--------|---------|
| `deployed` | This revision is currently active in the cluster. |
| `superseded` | A newer revision has replaced this one. |
| `failed` | The install/upgrade/rollback operation failed. Resources may still exist. |
| `pending-install` | Installation was initiated but not completed (e.g., timeout). |
| `pending-upgrade` | Upgrade was initiated but not completed. |
| `pending-rollback` | Rollback was initiated but not completed. |
| `uninstalling` | `helm uninstall` is in progress. |
| `unknown` | The release state could not be determined. |

---

## Debugging Failed Hooks

Helm hooks are Kubernetes resources annotated with `helm.sh/hook` that run at specific lifecycle points. When hooks fail, the install/upgrade/rollback fails.

### Common Hook Issues

```bash
# Check hook pod logs
kubectl logs -l helm.sh/hook=pre-install -n <namespace>

# Describe the hook resource to see events
kubectl describe job my-release-hook-name

# Check hook pod status
kubectl get pods -l helm.sh/hook=pre-install -n <namespace>

# Delete a stuck hook pod so Helm retries
kubectl delete pod -l helm.sh/hook=pre-install -n <namespace>
```

### Hook Weight Problems

Hooks execute in weight order (ascending). If hook B depends on hook A completing, ensure `helm.sh/hook-weight: "0"` is lower for hook A:

```yaml
annotations:
  "helm.sh/hook": pre-install
  "helm.sh/hook-weight": "0"   # Runs first
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

### Hook Delete Policy Issues

| Policy | Behavior | Debugging Tip |
|--------|----------|---------------|
| `before-hook-creation` | Delete previous hook before creating new one | Ensures a fresh hook run. Use when hooks are idempotent. |
| `hook-succeeded` | Delete hook after success | Hook pods cleaned automatically. Check logs before deletion. |
| `hook-failed` | Delete hook after failure | The default when no policy is set. Deleted on failure, making logs hard to retrieve. **Add `before-hook-creation` for debugging.** |
| `before-hook-creation,hook-succeeded` | Combined policy | The recommended safe default. Hooks are cleaned between runs and after success, but retained on failure for debugging. |

**Production Note:** Always include `hook-delete-policy: before-hook-creation,hook-succeeded` for production hooks. This keeps failed hook pods available for `kubectl logs` inspection while preventing accumulation of old hook resources.

---

## Debugging Immutable Field Errors

Kubernetes rejects updates to certain immutable fields. The most common: `spec.selector` in a Deployment.

```
Error: UPGRADE FAILED: Deployment.apps "my-app" is invalid: 
spec.selector: Invalid value: ... field is immutable
```

### Solutions

```bash
# Option 1: Delete and recreate (--force)
helm upgrade my-release ./mychart --force

# Option 2: Recreate pods (--recreate-pods)
helm upgrade my-release ./mychart --recreate-pods

# Option 3: Manually delete the Deployment, then upgrade
kubectl delete deployment my-app
helm upgrade my-release ./mychart

# Option 4: Use recreate strategy in the Deployment spec
```

**Warning:** `--force` deletes and recreates resources. This causes **downtime** unless you have multiple replicas and a proper rollout strategy. For stateful workloads, prefer a rolling update or manual migration.

The `--force` flag triggers the Kubernetes `Recreate` update strategy, which deletes all Pods before creating new ones. For zero-downtime immutable field changes, consider:
- Changing the resource name (blue/green deployment)
- Using a migration controller
- Accepting a brief outage during the recreate

---

## Debugging CRDs

Custom Resource Definitions have a special lifecycle in Helm.

### CRD Installation Order

CRDs must exist before any custom resources can be created. Helm installs CRDs in the `crds/` directory **before** any templates, but with important limitations:

```
Helm CRD Lifecycle:
  +------------------+
  | crds/*.yaml      |----> Applied FIRST (before templates)
  +------------------+
           |
           v
  +------------------+
  | templates/*.yaml |----> Applied SECOND (after CRDs exist)
  +------------------+
```

### CRD Gotchas

```bash
# CRDs are NOT managed by Helm after initial install
# Upgrades do NOT update CRDs — this is by design
# Use kubectl apply for CRD updates, or the crds/ directory for initial install only

# Skip CRD installation (useful when CRDs are managed separately)
helm install my-release ./mychart --skip-crds

# CRD too large error (> 1MB manifests)
# Solution: Split CRDs into separate chart or use kubectl apply directly
```

**Warning:** Helm does not manage the lifecycle of CRDs installed from `crds/`. They are not tracked in release history, not updated on `helm upgrade`, and not deleted on `helm uninstall`. For production CRD management, consider a dedicated CRD controller or separate CRD installation step.

---

## Debugging RBAC Issues

### Insufficient Permissions

```
Error: INSTALLATION FAILED: clusterroles.rbac.authorization.k8s.io is forbidden: 
User "system:serviceaccount:default:my-sa" cannot create resource "clusterroles"
```

| Issue | Diagnosis | Solution |
|-------|-----------|----------|
| Missing ClusterRole | `kubectl auth can-i create clusterroles --as=system:serviceaccount:ns:sa` | Create the ClusterRole or use an existing one. |
| ServiceAccount not bound | `kubectl get clusterrolebinding -o wide \| grep <serviceaccount>` | Create a ClusterRoleBinding. |
| Namespace-scoped role for cluster resource | Check that ClusterRole (not Role) is used for cluster-scoped resources. | Use ClusterRole + ClusterRoleBinding. |
| Missing verbs | Error message specifies the exact verb and resource. | Add the missing verb to the Role/ClusterRole rules. |

```bash
# Check what permissions a ServiceAccount has
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>

# Test a specific permission
kubectl auth can-i create deployments --as=system:serviceaccount:<ns>:<sa> -n <ns>

# See the effective RBAC for a SA
kubectl get rolebindings,clusterrolebindings --all-namespaces -o json | \
  jq '.items[] | select(.subjects[]?.name=="<sa-name>")'
```

---

## Debugging Dependencies

### Missing Dependencies

```
Error: found in Chart.yaml, but missing in charts/ directory: <dependency-name>
```

```bash
# Always run before installing a chart with dependencies
helm dependency update ./mychart

# List current dependencies and their status
helm dependency list ./mychart

# Build dependencies into charts/ directory
helm dependency build ./mychart

# Force update even if Chart.lock exists
helm dependency update ./mychart --skip-refresh
```

### Version Conflicts

When a dependency's subchart also has dependencies and versions conflict:

```bash
# Check the dependency tree
helm dependency list ./mychart

# Use --debug to see resolution details
helm dependency update ./mychart --debug

# Override subchart repository
# In Chart.yaml:
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
```

### Condition and Alias Not Resolving

If a dependency is not being installed despite being listed:

```bash
# Verify the condition/tag references a valid values key
# Chart.yaml:
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    condition: redis.enabled     # Must exist in values.yaml

# Debug with:
helm template my-release ./mychart --set redis.enabled=true --show-only charts/
```

---

## Debugging Templates

### The `fail` Function — Assertions

Use `fail` to guard against invalid combinations:

```yaml
{{- if and .Values.persistence.enabled (eq .Values.persistence.type "hostPath") }}
  {{- if not .Values.persistence.hostPath }}
    {{- fail "persistence.hostPath must be set when persistence.type is hostPath" }}
  {{- end }}
{{- end }}
```

### The `required` Function — Mandatory Values

```yaml
# In deployment.yaml:
image: "{{ .Values.image.registry }}/{{ required "image.repository is required" .Values.image.repository }}:{{ .Values.image.tag }}"
```

If `.Values.image.repository` is empty, Helm halts with:
```
Error: execution error at (mychart/templates/deployment.yaml:25:11): image.repository is required
```

**Production Note:** Use `required` for every value that would cause a broken deployment if missing. Common candidates: image repository, database connection strings, TLS certificate paths, service ports.

### The `tpl` Function — Dynamic Rendering

`tpl` evaluates a string as a template, enabling dynamic configuration:

```yaml
# values.yaml:
extraConfig: |
  log_level: {{ .Values.logLevel }}
  endpoint: {{ .Values.service.endpoint }}

# In a ConfigMap:
data:
  extra.conf: |
    {{- tpl .Values.extraConfig . | nindent 4 }}
```

Debugging `tpl` issues:

```bash
# Isolate the tpl expression
{{- $rendered := tpl .Values.extraConfig . }}
{{- if not $rendered }}
  {{- fail "extraConfig rendered empty" }}
{{- end }}
```

---

## Debugging YAML

### Multi-Document YAML

Ensure `---` separators are on their own line without trailing whitespace:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-a
data:
  key: value
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-b
data:
  key: value
```

**Note:** The `---` separator must have a newline before and after it. Trailing whitespace after `---` can cause YAML parsers to treat it as a content line rather than a document separator.

### Indentation Issues

```bash
# Use helm template with --debug to see exact YAML output
helm template my-release ./mychart --debug 2>&1 | grep -A5 "Error"

# Validate YAML separately
helm template my-release ./mychart | yamllint -

# Check for common indentation mistakes
# - nindent vs indent: nindent adds a newline before indentation
# - Missing pipe (|) on multiline strings
# - Forgetting to use quote on values that look like numbers
```

---

## Debugging Rendering Issues

### Values Not Appearing

| Symptom | Likely Cause | Solution |
|---------|-------------|----------|
| Value renders empty | Typo in value key name | Compare exact key path in values.yaml vs template. Use `helm get values --all` to see merged values. |
| Value shows `<no value>` | Referencing a key that does not exist | Use `{{ .Values.key | default "fallback" }}` or add `required`. |
| Wrong value appears | Value overridden by higher-precedence source | Check merge order: defaults < parent chart values < user values file < `--set` < `--set-string` < `--set-file` |
| Value is type `string` but expected `int` | YAML/CLI `--set` type coercion | Use `--set-string` for strings, cast in template with `{{ int .Values.port }}` or `{{ .Values.port | int }}`. |

### Type Issues

```yaml
# values.yaml — these are the correct types:
replicas: 3           # int
enabled: true         # bool
timeout: 30s          # string (duration)
resources:
  limits:
    memory: "512Mi"   # string (quoted to prevent YAML scientific notation issues)

# Template type casting:
{{ .Values.replicas | int }}
{{ .Values.enabled | toString }}
{{ .Values.timeout | float64 }}
```

**Warning:** Integer values in `values.yaml` that look like scientific notation (e.g., `1e6`) will be parsed as floats by YAML. Always quote string values that could be misinterpreted. Use `--set-string` to force string coercion from the CLI.

---

## kubectl Integration for Debugging

### Inspecting Release Secrets

Every Helm release is stored as one or more Kubernetes Secrets:

```bash
# List all Helm release secrets
kubectl get secrets -l owner=helm -n <namespace>

# Get a specific release's secrets across all namespaces
kubectl get secrets -l owner=helm,name=<release-name> --all-namespaces

# Inspect a specific revision's secret
kubectl get secret sh.helm.release.v1.<release-name>.v<revision> \
  -n <namespace> -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip | jq .

# The decoded output contains:
# - name: release name
# - info: status, first_deployed, last_deployed, description
# - chart: Chart.yaml content
# - config: merged values used
# - manifest: rendered Kubernetes manifests
# - version: Helm version used
```

### Release Secret Structure

```
sh.helm.release.v1.<release-name>.v<revision>
    ├── type: "helm.sh/release.v1"
    ├── labels:
    │     ├── owner: "helm"
    │     ├── name: "<release-name>"
    │     ├── version: "<revision>"
    │     ├── status: "deployed"
    │     └── modified_at: "<timestamp>"
    └── data:
          └── release: <base64(base64(gzip(json)))>
```

**Production Note:** Direct manipulation of release secrets is dangerous and unsupported. Never edit these secrets manually. Use `helm` commands for all release management operations.

---

## Debugging Upgrade Failures

### Three-Way Merge Issues

Helm 3 uses a three-way strategic merge patch:

```
         Old Manifest (stored in release secret)
                    |
        +-----------+-----------+
        |                       |
   Current Live           New Manifest
   (from API server)      (rendered from chart)
                    |
        +-----------+-----------+
                    |
              Patch Applied
```

If the live state has drifted from the old manifest (e.g., manual `kubectl edit`), the merge may produce unexpected results or conflicts.

```bash
# Detect drift between stored manifest and live state
helm get manifest my-release > stored.yaml
kubectl get deployment my-app -o yaml > live.yaml
diff stored.yaml live.yaml

# If drift exists, decide:
# 1. Reconcile: update the chart to match live changes (recommended)
# 2. Overwrite: use --force to recreate resources
# 3. Manual fix: kubectl edit to align live state with chart
```

### Orphaned Resources

Resources that were in a previous revision but removed from the chart become orphaned:

```bash
# List all resources with the release label
kubectl get all -l app.kubernetes.io/instance=<release-name> -n <namespace>

# Check if any are not in the current manifest
helm get manifest <release-name> | kubectl get -f - --dry-run=client -o name \
  | sort > current.txt
kubectl get all -l app.kubernetes.io/instance=<release-name> -o name \
  | sort > actual.txt
comm -13 current.txt actual.txt   # Shows resources in cluster but not in manifest
```

### cleanup-on-fail

When an upgrade fails partially, some resources may have been created while others failed:

```bash
# Rollback the failed upgrade and clean up partially created resources
helm rollback my-release --cleanup-on-fail
```

---

## Debugging helm install Timeouts

```bash
# Increase timeout for slow-starting pods
helm install my-release ./mychart --timeout 10m

# Wait for all resources to be ready
helm install my-release ./mychart --wait

# Wait for Jobs to complete
helm install my-release ./mychart --wait --wait-for-jobs

# Debug why a resource is not becoming ready
kubectl describe deployment <name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Check pod status and recent events
kubectl get pods -n <namespace> -l app.kubernetes.io/instance=<release>
kubectl describe pod <pod-name> -n <namespace>
```

**Note:** `--wait` blocks until all Deployments, StatefulSets, DaemonSets, and PVCs report ready. It does **not** wait for custom resources or Jobs by default. Use `--wait-for-jobs` for Jobs.

---

## helm-diff Plugin — Preview Change Deltas

The `helm-diff` plugin shows the actual differences between revisions before applying:

```bash
# Install the plugin
helm plugin install https://github.com/databus23/helm-diff

# Preview changes between current release and proposed upgrade
helm diff upgrade my-release ./mychart --values ./new-values.yaml

# Preview what an install would create
helm diff install my-new-release ./mychart

# Show differences from a specific revision
helm diff revision my-release 2 3

# Diff with detailed context
helm diff upgrade my-release ./mychart --context 5

# Suppress secrets in the diff output (redacts Secret data)
helm diff upgrade my-release ./mychart --suppress-secrets

# Output as JSON for programmatic use
helm diff upgrade my-release ./mychart --output json
```

### Sample Output

```diff
default, my-app, Deployment (apps) has changed:
  # Source: mychart/templates/deployment.yaml
...
          image: nginx:1.25
-         resources:
-           limits:
-             cpu: 500m
-             memory: 512Mi
+         resources:
+           limits:
+             cpu: 1000m
+             memory: 1Gi
...
```

Integrate into CI/CD:

```bash
# In a pipeline: fail if there are unexpected changes
CHANGES=$(helm diff upgrade my-release ./mychart --output json | jq 'length')
if [ "$CHANGES" -gt 0 ]; then
  echo "Changes detected:"
  helm diff upgrade my-release ./mychart
fi
```

---

## Post-Renderer Debugging

The `--post-renderer` flag passes rendered manifests through an external program before applying:

```bash
# Use kustomize as a post-renderer
helm install my-release ./mychart --post-renderer ./kustomize-wrapper.sh

# The wrapper script receives manifests on stdin and must write results to stdout
# Example wrapper:
#!/bin/bash
cat <&0 > /tmp/rendered.yaml
kustomize build /tmp/kustomize-overlay
```

```bash
# Debug post-renderer output
helm template my-release ./mychart --post-renderer ./my-script.sh > output.yaml

# Use a simple pass-through for debugging
helm install my-release ./mychart --post-renderer /bin/cat --dry-run --debug
```

**Note:** Post-renderers allow arbitrary YAML manipulation (adding labels, patching annotations, injecting sidecars) but also introduce a debugging surface. Always test post-renderer scripts with `helm template` before applying.

---

## Common Errors — Causes and Solutions

| Error Message | Root Cause | Solution |
|---------------|------------|----------|
| `Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest: error validating "": error validating data: [ValidationError(Deployment.spec.selector): missing required field "matchLabels"...]` | Missing required Kubernetes fields in the rendered manifest. | Fix the template to include all required fields. Use `helm template` to inspect output. |
| `Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists. Unable to continue with install: ...` | A resource with the same name exists from a different release or was manually created. | Use a different release name, delete the conflicting resource, or use `--force`. |
| `Error: UPGRADE FAILED: "my-release" has no deployed releases` | The release exists but has no deployed revision (all failed or were purged). | Rollback to a known working revision or reinstall with a new name. |
| `Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress` | A concurrent Helm operation is running (pending state). | Wait for the operation to complete. If stuck, delete the pending release secret: `kubectl delete secret -l owner=helm,name=<name>,status=pending-upgrade`. |
| `Error: create: failed to create: Secret "sh.helm.release.v1.my-release.v1" already exists` | Release name already exists in the namespace. | Use a different release name or `helm upgrade` instead of `helm install`. |
| `Error: execution error at (<chart>/templates/deployment.yaml:25): <message>` | Template rendering error at a specific line. | Navigate to the line in the template file. Check for nil values, wrong function signatures, or syntax errors. |
| `Error: no deployed release with the given name` | Release name doesn't exist or all revisions are failed. | Check spelling, namespace, and release history. Use `helm list -A` to see all releases. |
| `Error: failed to download "<chart>"` | Repository index not updated, wrong URL, or network issue. | Run `helm repo update`. Verify the repository URL. Check network connectivity. |
| `Error: looks like "<url>" is not a valid chart repository or cannot be reached` | Repository URL is malformed or the server is down. | Verify the URL with `curl`. Re-add the repo if the URL changed. |
| `Error: UPGRADE FAILED: Deployment.apps "..." is invalid: spec.selector: field is immutable` | Attempting to change an immutable field. | Use `--force` to recreate, or adopt a migration strategy. |
| `Error: configuration: apiVersion <v> for <resource> is not available in this Kubernetes version` | Template uses a deprecated or too-new API version. | Update the template to use a supported API version. Check `.Capabilities.APIVersions`. |
| `Error: context deadline exceeded` | Helm timed out waiting for resources to become ready. | Increase `--timeout`. Check if pods are stuck in Pending/CrashLoopBackOff. Use `kubectl describe` and `kubectl logs`. |
| `Error: Failed to render chart: exit status 1` | Template engine error, often from a `fail` or `required` call. | Run `helm lint --strict` and inspect the error message for the exact cause. |

---

## Debugging Workflow Diagram

```
+------------------------------------------------------------------------------------------+
|                              HELM DEBUGGING WORKFLOW                                      |
+------------------------------------------------------------------------------------------+
|                                                                                           |
|  Error encountered                                                                        |
|       |                                                                                   |
|       v                                                                                   |
|  +-----------------------+    NO     +---------------------------+                        |
|  | Is the error message  |---------->| helm lint --strict        |                        |
|  | self-explanatory?     |           | (catch YAML/template err) |                        |
|  +-----------------------+           +------------+--------------+                        |
|       | YES                                     |                                        |
|       v                                         v                                        |
|  +-----------------------+           +---------------------------+                        |
|  | Fix the issue        |           | helm template --debug     |                        |
|  | directly             |           | (render locally)          |                        |
|  +-----------------------+           +------------+--------------+                        |
|       |                                        |                                        |
|       v                                        v                                        |
|  +-----------------------+           +---------------------------+                        |
|  | Verify fix with       |           | Error found?              |                        |
|  | helm template         |           +-----+---------------------+                        |
|  +-----------------------+                 |                                          |
|       |                          YES       |       NO                                      |
|       v                           |               |                                       |
|  +-----------------------+        |               v                                       |
|  | helm install --dry-run<--------+  +---------------------------+                        |
|  | --debug (cluster check)|           | Check values:             |                        |
|  +-----------------------+           | helm get values --all     |                        |
|       |                              | (verify merged values)    |                        |
|       v                              +------------+--------------+                        |
|  +-----------------------+                        |                                       |
|  | Dry-run passes?       |          NO            v                                       |
|  +-----+-----------------+          |    +---------------------------+                    |
|        | YES                        |    | Values wrong?             |                    |
|        |                            |    +-----+---------------------+                    |
|        v                            |     YES  |      NO                                 |
|  +-----------------------+          |          |       |                                 |
|  | Apply fix for real    |<---------+          v       v                                 |
|  +-----------------------+          |  Fix values  +---------------------------+         |
|       |                            |              | Check cluster state:      |         |
|       v                            |              | kubectl get events        |         |
|  +-----------------------+          |              | kubectl describe          |         |
|  | helm test              |         |              | kubectl logs              |         |
|  | helm status            |         |              | kubectl get secrets       |         |
|  | kubectl describe       |         |              |   -l owner=helm           |         |
|  +-----------------------+          |              +------------+--------------+         |
|       |                            |                           |                        |
|       v                            |                           v                        |
|  +-----------------------+          |              +---------------------------+         |
|  | Issue resolved?       |----------+              | RBAC issue?              |         |
|  +-----------------------+                         +-----+---------------------+         |
|       | YES                                             |    |                           |
|       v                                           YES   |    |  NO                       |
|  +-----------------------+                              |    v                           |
|  | Document the fix      |                              |  +-------------------------+  |
|  | (runbook entry)       |                              |  | Hook failure?           |  |
|  +-----------------------+                              |  +-----+-------------------+  |
|                                                         |        |    |                 |
|                                                         |   YES  |    |  NO             |
|                                                         |        |    v                 |
|                                                         |        |  Check CRD,          |
|                                                         |        |  dependency,         |
|                                                         |        |  version issues      |
|                                                         v        |                      |
|                                                       Fix       v                      |
|                                                       RBAC    +-------------------+    |
|                                                               | Escalate to SME   |    |
|                                                               +-------------------+    |
+------------------------------------------------------------------------------------------+
```

---

## Summary

| Debugging Method | When to Use | Key Advantage |
|------------------|-------------|---------------|
| `helm lint` | Before any deployment | Catches template/YAML errors without a cluster |
| `helm template --debug` | Template logic issues | Full local rendering visibility |
| `helm install --dry-run --debug` | Pre-deployment validation | Kubernetes API validation |
| `helm get manifest/values` | Post-deployment inspection | See what was actually deployed |
| `helm history` | Tracking changes over time | Identify when issues were introduced |
| `helm test` | Verifying functionality | Automated post-deployment checks |
| `helm-diff plugin` | Previewing changes | Understand exactly what will change |
| `kubectl logs/describe/events` | Runtime issues | Deep cluster-level diagnostics |
| Release Secrets inspection | Release corruption | Forensic analysis of release data |