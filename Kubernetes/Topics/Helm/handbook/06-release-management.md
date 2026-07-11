# Chapter 6: Release Management

## 6.1 The Release Concept

A Helm **release** is the fundamental unit of deployment in Helm. It is not merely a chart or a set of Kubernetes resources—it is the union of four things:

```
Release = Chart + Values + Namespace + Release Name
```

- **Chart**: A packaged Helm chart (`.tgz` or unpacked directory) containing templates, default values, and metadata.
- **Values**: Configuration data merged into templates, drawn from `values.yaml`, user-supplied `--values` files, `--set` overrides, and previous release state.
- **Namespace**: The Kubernetes namespace where resources are instantiated.
- **Release Name**: A unique identifier within the namespace (or cluster, depending on the storage backend).

When you run `helm install`, Helm records the release in its **release storage** (a Kubernetes Secret, ConfigMap, or external SQL store). Each subsequent `helm upgrade` creates a new **revision** of that release, tracked sequentially (revision 1, 2, 3, …).

**Note:** The same chart can be installed multiple times under different release names or in different namespaces. Each installation is an independent release.

---

## 6.2 Release Lifecycle

A release passes through several states during its lifetime:

```
install → deployed
   ↓
upgrade → deployed (revision N+1)
   ↓
rollback → deployed (revision N-1)
   ↓
uninstall → uninstalled
```

Intermediate and terminal states:

| State | Description |
|---|---|
| `unknown` | Release is in an undefined state (rare) |
| `deployed` | Successfully installed or upgraded; all resources exist |
| `uninstalled` | Release has been removed (`helm uninstall` succeeded) |
| `superseded` | A former revision replaced by a newer upgrade |
| `failed` | Install/upgrade/rollback did not complete successfully |
| `uninstalling` | Uninstall is in progress |
| `pending-install` | Install is in progress |
| `pending-upgrade` | Upgrade is in progress |
| `pending-rollback` | Rollback is in progress |

---

## 6.3 `helm install`

### 6.3.1 Basic Syntax

```
helm install [NAME] [CHART] [flags]
```

- `NAME`: The release name. If omitted, `--generate-name` or `--name-template` must be supplied.
- `CHART`: A chart reference, which can be a chart archive (`.tgz`), an unpacked chart directory, a fully qualified URL, or a chart reference in a repository (e.g., `stable/nginx-ingress`).

**Example:**

```bash
helm install my-release oci://registry-1.docker.io/bitnamicharts/nginx
```

### 6.3.2 Every Flag Explained

| Flag | Type | Default | Description |
|---|---|---|---|
| `--name-template` | string | `""` | Specify a Go template for the release name. Useful for naming conventions (e.g., `"{{ .Release.Name }}-{{ .Release.Namespace }}"`). Mutually exclusive with `--generate-name` and a positional name argument. |
| `--generate-name` | bool | `false` | Generate a release name by appending a random suffix to the chart name (e.g., `nginx-ingress-1691427345`). Mutually exclusive with `--name-template` and a positional name argument. |
| `--create-namespace` | bool | `false` | Create the release namespace if it does not already exist. Helm will not create a namespace by default; the namespace must pre-exist unless this flag is set. |
| `--dependency-update` | bool | `false` | Run `helm dependency update` before installing the chart. Downloads any subcharts listed in `Chart.yaml` dependencies and updates `Chart.lock`. |
| `--description` | string | `""` | A human-readable description string added to the release metadata. Visible in `helm list` and `helm history`. |
| `--dry-run` | bool | `false` | Simulate the installation without actually creating resources. Templates are rendered and validated, and the output is printed. Combine with `--debug` for verbose template output. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS certificate verification when fetching the chart from a repository URL. **Warning:** This disables a critical security check. Use only in trusted development environments. |
| `--no-hooks` | bool | `false` | Prevent hooks (pre-install, post-install, etc.) from running during the installation. Resources defined by hooks are still rendered and applied, but the hook annotations are ignored. |
| `--output` | string | `table` | Output format for the install summary. Valid values: `table`, `json`, `yaml`. |
| `--password` | string | `""` | Chart repository password. Used for authentication when the chart is fetched from a password-protected repository. |
| `--post-renderer` | string | `""` | Path to an executable that Helm will pipe rendered manifests through before applying them. The script receives YAML on stdin and must emit YAML on stdout (e.g., a Kustomize binary). |
| `--render-subchart-notes` | bool | `false` | If set, includes subchart NOTES.txt output in the final install notes. By default, only the parent chart's notes are shown. |
| `--replace` | bool | `false` | Reuse the given release name even if it already exists (reinstall). Deletes the existing release first, then re-installs. Equivalent to `helm uninstall` followed by `helm install`. |
| `--repo` | string | `""` | Chart repository URL where the chart is located. Allows you to reference a chart by name and specify the repo URL directly without adding it via `helm repo add` first. |
| `--set` | stringArray | `[]` | Set values on the command line using dot-notation. Can be specified multiple times. Example: `--set service.port=8080,replicaCount=3`. |
| `--set-file` | stringArray | `[]` | Set values from files on the command line. The file's contents are injected as the value. Example: `--set-file config.data=./config.yaml`. |
| `--set-json` | stringArray | `[]` | Set JSON values on the command line. The value is parsed as JSON (not a string). Example: `--set-json 'resources.limits={"cpu":"500m","memory":"256Mi"}'`. |
| `--set-string` | stringArray | `[]` | Set STRING values on the command line. Useful when a value looks numeric but should be treated as a string. Example: `--set-string app.port="8080"`. |
| `--skip-crds` | bool | `false` | If the chart contains CRDs in the `crds/` directory, skip installing them. CRDs are installed only on `helm install`, not on `helm upgrade`, by default. |
| `--timeout` | duration | `5m0s` | Maximum time to wait for the Kubernetes operation to complete. Uses Go duration format (e.g., `10m`, `300s`, `1h`). |
| `--username` | string | `""` | Chart repository username. Used with `--password` for basic authentication against the chart repository. |
| `--values` / `-f` | stringArray | `[]` | Specify YAML values files. Can be specified multiple times. The order matters: later files override earlier ones for overlapping keys. |
| `--verify` | bool | `false` | Verify the chart's provenance file (`.prov`) before installation. Requires a valid GPG keychain and that the chart was signed with `helm package --sign`. |
| `--version` | string | `""` | Specify a chart version constraint. If omitted, the latest version is used. Examples: `--version 1.2.3`, `--version ">=1.0.0 <2.0.0"`. |
| `--wait` | bool | `false` | Wait for all Pods, PVCs, Services, and minimum required Deployments/StatefulSets to reach a ready state before marking the release as successful. |
| `--wait-for-jobs` | bool | `false` | Wait for all Jobs to complete before marking the release as successful. If a Job fails, the install is marked as failed. Used with `--wait`. |
| `--atomic` | bool | `false` | If set, the installation process will roll back all changes on failure. Equivalent to using `--wait` and automatically rolling back if the install fails. |

### 6.3.3 Common Install Patterns

```bash
# Simple install
helm install my-nginx oci://registry-1.docker.io/bitnamicharts/nginx

# Install with values file and set overrides
helm install my-app ./my-chart \
  -f values/production.yaml \
  --set replicaCount=3 \
  --set image.tag=v1.2.3

# Install with atomic (rollback on failure) and wait for jobs
helm install my-app ./my-chart \
  --atomic \
  --wait-for-jobs \
  --timeout 10m \
  --namespace production \
  --create-namespace

# Dry-run to inspect rendered manifests
helm install my-app ./my-chart \
  --dry-run \
  --debug \
  -f values/staging.yaml
```

**Production Note:** Always use `--atomic` in CI/CD pipelines to ensure failed deploys roll back cleanly. Pair it with `--wait-for-jobs` if your chart includes pre/post-install Jobs.

---

## 6.4 `helm upgrade`

### 6.4.1 Basic Syntax

```
helm upgrade [RELEASE] [CHART] [flags]
```

`helm upgrade` takes an existing release and upgrades it to a new version of the chart and/or new values. Each upgrade creates a new **revision**, which is stored in the release history.

### 6.4.2 Every Flag Explained

| Flag | Type | Default | Description |
|---|---|---|---|
| `--atomic` | bool | `false` | Automatically roll back the release to the previous successful revision on upgrade failure. Implies `--wait`. |
| `--cleanup-on-fail` | bool | `false` | Delete newly created resources if the upgrade fails. Resources that were updated (not newly created) are left as-is. |
| `--create-namespace` | bool | `false` | Create the release namespace if it does not already exist. |
| `--dependency-update` | bool | `false` | Run `helm dependency update` before upgrading. |
| `--description` | string | `""` | Human-readable description added to the revision. |
| `--devel` | bool | `false` | Include development (pre-release) versions when selecting chart versions. Useful for testing beta or alpha releases. |
| `--dry-run` | bool | `false` | Simulate the upgrade without applying any changes. Renders templates against proposed values and prints the diff. Combine with `--debug` for detailed output. |
| `--force` | bool | `false` | Force resource updates through a delete-and-recreate strategy. Not the same as `kubectl apply --force`. Use when a resource's immutable fields need to change (e.g., `spec.selector` on a Deployment). |
| `--history-max` | int | `10` | Maximum number of revisions to retain. Older revisions are deleted. Increase this if you need to roll back beyond the last 10 revisions. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS certificate verification when fetching the chart. |
| `--install` / `-i` | bool | `false` | If the release does not already exist, run `helm install` instead. **This is the most commonly used flag for upgrade.** Makes deployments idempotent. |
| `--no-hooks` | bool | `false` | Skip running hooks during the upgrade. |
| `--output` | string | `table` | Output format: `table`, `json`, or `yaml`. |
| `--password` | string | `""` | Chart repository password. |
| `--post-renderer` | string | `""` | Path to a post-renderer executable. |
| `--render-subchart-notes` | bool | `false` | Include subchart notes in the output. |
| `--repo` | string | `""` | Chart repository URL. |
| `--reset-then-reuse-values` | bool | `false` | (Helm 3.11+) Reset values to chart defaults, then re-apply the previously deployed values on top. This is effectively a "staged reset": first clear everything, then layer on old values, then apply any new `--set`/`-f` values on top *of those*. |
| `--reset-values` | bool | `false` | Reset values to the chart's built-in defaults before applying any `-f` or `--set` values. Previously deployed values are discarded entirely. |
| `--reuse-values` | bool | `false` | When upgrading, reuse the last release's values and merge any new `-f` or `--set` values on top. New chart defaults are *not* picked up unless explicitly set. |
| `--set` | stringArray | `[]` | Set values via dot-notation (same as install). |
| `--set-file` | stringArray | `[]` | Set values from file contents (same as install). |
| `--set-json` | stringArray | `[]` | Set JSON values (same as install). |
| `--set-string` | stringArray | `[]` | Set string values (same as install). |
| `--skip-crds` | bool | `false` | Skip CRD installation (CRDs are not upgraded by default on upgrade anyway). |
| `--timeout` | duration | `5m0s` | Timeout for the upgrade operation. |
| `--username` | string | `""` | Chart repository username. |
| `--values` / `-f` | stringArray | `[]` | Values files to merge into the release. |
| `--verify` | bool | `false` | Verify the chart's provenance file. |
| `--version` | string | `""` | Specific chart version to upgrade to. |
| `--wait` | bool | `false` | Wait for resources to reach a ready state before marking the upgrade successful. |
| `--wait-for-jobs` | bool | `false` | Wait for Jobs to complete. Used with `--wait`. |

### 6.4.3 The `--install` Pattern (Idempotent Deployment)

The `helm upgrade --install` pattern is the de facto standard for CI/CD pipelines:

```bash
helm upgrade --install my-app ./my-chart \
  --namespace production \
  --create-namespace \
  --atomic \
  --wait-for-jobs \
  --timeout 10m \
  -f values/production.yaml
```

This single command:
1. Installs the release if it does not exist (`helm install` behavior).
2. Upgrades the release if it already exists (`helm upgrade` behavior).
3. Rolls back automatically on failure (`--atomic`).
4. Creates the namespace if needed (`--create-namespace`).

**Production Note:** This is the recommended pattern for all CI/CD deployments. It is idempotent—you can run it repeatedly and it will either install or upgrade as needed.

### 6.4.4 `--force` Deep Dive

`--force` is frequently misunderstood. It performs a **delete-and-recreate** for resources where the patch/update fails. Specifically:

- For **Deployments**: If the `spec.selector` has changed (an immutable field), Helm issues a `DELETE` followed by a `CREATE`. This causes pod churn and momentary downtime.
- For **DaemonSets**: Same delete-and-recreate behavior.
- For **StatefulSets**: Same behavior, but PVCs are preserved.

**Warning:** `--force` causes pod restarts. Do not use it casually in production. If you need zero-downtime changes to immutable fields, consider a blue-green or canary strategy instead.

---

## 6.5 `helm uninstall`

### 6.5.1 Basic Syntax

```
helm uninstall [RELEASE] [flags]
```

### 6.5.2 Every Flag Explained

| Flag | Type | Default | Description |
|---|---|---|---|
| `--description` | string | `""` | Human-readable description added to the release history (visible in `helm list --uninstalled`). |
| `--dry-run` | bool | `false` | Simulate the uninstall. Shows which resources would be deleted without actually removing anything. |
| `--keep-history` | bool | `false` | Retain the release history (all revisions) after uninstall. The release moves to `uninstalled` state but its revision data is preserved. Useful for audit trails. |
| `--no-hooks` | bool | `false` | Prevent hooks from running during uninstall. Pre-delete and post-delete hooks are skipped. |
| `--timeout` | duration | `5m0s` | Maximum time to wait for the uninstall to complete. |
| `--wait` | bool | `false` | Wait for all resources to be fully deleted before returning. Ensures cleanup is complete. |

### 6.5.3 Common Uninstall Patterns

```bash
# Simple uninstall
helm uninstall my-release

# Uninstall but keep history for auditing
helm uninstall my-release --keep-history

# Dry-run to see what gets deleted
helm uninstall my-release --dry-run

# Uninstall with timeout
helm uninstall my-release --timeout 10m --wait
```

---

## 6.6 `helm list`

### 6.6.1 Basic Syntax

```
helm list [flags]
```

Lists all releases in the current namespace (or all namespaces with `-A`).

### 6.6.2 Every Flag Explained

| Flag | Type | Default | Description |
|---|---|---|---|
| `--all` / `-a` | bool | `false` | Show all releases regardless of status. By default, only `deployed` releases are shown. |
| `--all-namespaces` / `-A` | bool | `false` | List releases across all namespaces. |
| `--date` / `-d` | bool | `false` | Sort releases by date (most recent first). |
| `--deployed` | bool | `false` | Show only releases in `deployed` state. |
| `--failed` | bool | `false` | Show only releases in `failed` state. |
| `--filter` / `-f` | string | `""` | A regular expression to filter release names. Case-insensitive. Example: `--filter "nginx|redis"`. |
| `--max` / `-m` | int | `256` | Maximum number of releases to return. |
| `--namespace` / `-n` | string | `default` | Namespace scope. |
| `--offset` / `-o` | int | `0` | Offset for pagination (skip first N releases). |
| `--pending` | bool | `false` | Show only releases in `pending-install`, `pending-upgrade`, or `pending-rollback` states. |
| `--reverse` / `-r` | bool | `false` | Reverse the sort order. |
| `--selector` / `-l` | string | `""` | Label selector to filter releases. Uses Kubernetes label selector syntax. Example: `--selector "app.kubernetes.io/name=nginx"`. |
| `--short` / `-q` | bool | `false` | Output only release names (one per line). Useful for scripting. |
| `--superseded` | bool | `false` | Show only superseded releases (old revisions). |
| `--time-format` | string | `""` | Format for timestamps using Go time layout (e.g., `"2006-01-02 15:04:05"`). |
| `--uninstalled` | bool | `false` | Show only `uninstalled` releases. Only works if `--keep-history` was used during uninstall. |
| `--uninstalling` | bool | `false` | Show only releases in `uninstalling` state. |

### 6.6.3 Common Query Patterns

```bash
# List all releases in current namespace
helm list

# List across all namespaces
helm list -A

# List all releases regardless of status
helm list -a

# List only failed releases
helm list --failed

# Filter by name regex
helm list -f "web"

# Filter by label
helm list -l "app=frontend"

# Short output for scripting
helm list -q

# Show releases with custom time format
helm list --time-format "2006-01-02 15:04:05"

# Pagination: skip first 20, show 50
helm list --offset 20 --max 50
```

---

## 6.7 `helm history`

### 6.7.1 Basic Syntax

```
helm history RELEASE [flags]
```

Displays all revisions for a release, with status, description, and timestamps.

### 6.7.2 Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--max` | int | `256` | Maximum number of revisions to show. |
| `--namespace` / `-n` | string | `default` | Namespace scope. |
| `--output` / `-o` | string | `table` | Output format: `table`, `json`, `yaml`. |

### 6.7.3 Output Format (Table)

```
REVISION  UPDATED                    STATUS       CHART            APP VERSION  DESCRIPTION
1         Mon Jan  1 12:00:00 2024   superseded   my-app-1.0.0    1.16.0       Install complete
2         Mon Jan  1 12:05:00 2024   superseded   my-app-1.1.0    1.17.0       Upgrade complete
3         Mon Jan  1 12:10:00 2024   deployed     my-app-1.2.0    1.18.0       Upgrade complete
```

### 6.7.4 Scripting with Helm History

```bash
# Get the current revision number
CURRENT_REV=$(helm history my-release -o json | jq '.[-1].revision')

# Get the last successful revision (non-failed)
LAST_GOOD=$(helm history my-release -o json | \
  jq '[.[] | select(.status == "deployed" or .status == "superseded")] | last | .revision')
```

---

## 6.8 `helm status`

### 6.8.1 Basic Syntax

```
helm status RELEASE [flags]
```

Displays the current state of a release.

### 6.8.2 Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--namespace` / `-n` | string | `default` | Namespace scope. |
| `--output` / `-o` | string | `table` | Output format: `table`, `json`, `yaml`. |
| `--revision` | int | `0` | Show the status of a specific revision. If `0`, the latest revision is shown. |
| `--show-desc` | bool | `false` | Show the description of each revision (requires `--revision`). |

### 6.8.3 Output Sections

```
NAME: my-release
LAST DEPLOYED: Mon Jan  1 12:10:00 2024
NAMESPACE: default
STATUS: deployed
REVISION: 3
TEST SUITE: None
NOTES:
1. Get the application URL by running:
   kubectl port-forward svc/my-release 8080:80
```

- **NAME**: The release name.
- **LAST DEPLOYED**: Timestamp of the most recent successful deploy.
- **NAMESPACE**: Kubernetes namespace.
- **STATUS**: The release state (`deployed`, `failed`, `pending-*`, etc.).
- **REVISION**: Current revision number.
- **TEST SUITE**: Results of any Helm tests (`helm test`) that have been run.
- **NOTES**: The rendered output from `templates/NOTES.txt` in the chart.

### 6.8.4 Inspecting a Specific Revision

```bash
helm status my-release --revision 2
```

---

## 6.9 `helm get` Commands

Helm provides six subcommands to inspect release data.

### 6.9.1 `helm get values`

Retrieves the **override values** that were supplied for a release—the combined result of all `--set`, `--set-*`, and `--values` flags. It does **not** show chart defaults.

```bash
helm get values my-release
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--all` / `-a` | bool | `false` | Show all computed values, including chart defaults. This merges chart `values.yaml` with user overrides. |
| `--namespace` / `-n` | string | `default` | Namespace. |
| `--output` / `-o` | string | `table` | Format: `table`, `json`, `yaml`. |
| `--revision` | int | `0` | Retrieve values for a specific revision. Default (`0`) means latest. |

```bash
# See all computed values (chart defaults + overrides)
helm get values my-release --all -o yaml

# See values from a specific revision
helm get values my-release --revision 2
```

### 6.9.2 `helm get manifest`

Retrieves the **rendered Kubernetes manifests** for a release—the exact YAML that was sent to the Kubernetes API.

```bash
helm get manifest my-release
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--namespace` / `-n` | string | `default` | Namespace. |
| `--revision` | int | `0` | Specific revision. |

```bash
# Pipe diff to see what changed between revisions
diff <(helm get manifest my-release --revision 2) \
     <(helm get manifest my-release --revision 3)
```

### 6.9.3 `helm get hooks`

Retrieves the **hooks** associated with a release—the hook resources that were executed during install, upgrade, or delete.

```bash
helm get hooks my-release
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--namespace` / `-n` | string | `default` | Namespace. |
| `--revision` | int | `0` | Specific revision. |

**Example output:**

```yaml
---
# Source: my-chart/templates/hooks.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "my-release-pre-upgrade"
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "5"
```

### 6.9.4 `helm get notes`

Retrieves the **NOTES.txt** output for a release. This is the same content shown by `helm status`.

```bash
helm get notes my-release
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--namespace` / `-n` | string | `default` | Namespace. |
| `--revision` | int | `0` | Specific revision. |

### 6.9.5 `helm get metadata`

Retrieves release metadata including name, namespace, status, revision, and chart info.

```bash
helm get metadata my-release
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--namespace` / `-n` | string | `default` | Namespace. |
| `--revision` | int | `0` | Specific revision. |

### 6.9.6 `helm get all`

Combined output of values, manifest, hooks, notes, and metadata.

```bash
helm get all my-release
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--namespace` / `-n` | string | `default` | Namespace. |
| `--revision` | int | `0` | Specific revision. |
| `--template` | string | `""` | Go template to format the output. This is **not** a chart template—it formats the `helm get all` output. Example: `--template '{{ .Name }}:{{ .Namespace }}'`. |

---

## 6.10 Release Naming

### 6.10.1 Naming Conventions

A release name must be a valid DNS-1123 subdomain:

- Maximum 53 characters.
- Lowercase letters, digits, and hyphens only.
- Must start with a letter.
- Must end with a letter or digit.

**Recommended conventions:**

```
<project>-<environment>          # my-app-prod, my-app-staging
<service>-<region>-<env>         # api-us-east-prod
<team>-<service>                 # platform-auth-service
```

Avoid embedding revision numbers in the name—Helm tracks revisions automatically.

### 6.10.2 `--generate-name`

Appends a random 6-character suffix to the chart name:

```bash
helm install --generate-name ./my-chart
# Creates: my-chart-1691427345
```

Useful for temporary or test installs where uniqueness matters more than predictability.

### 6.10.3 `--name-template`

Uses a Go template to generate the name:

```bash
# Prepend the namespace
helm install --name-template "{{ .Release.Namespace }}-{{ .Release.Name }}" ./my-chart

# Custom prefix
helm install --name-template "myapp-{{ .Release.Name }}" ./my-chart
```

Available template variables include `{{ .Release.Name }}` and `{{ .Release.Namespace }}`.

**Note:** `--name-template` requires that you do NOT provide a positional name argument.

---

## 6.11 Release Namespaces

### 6.11.1 Specifying Namespaces

```bash
# Via --namespace flag
helm install my-release ./chart --namespace production

# Via HELM_NAMESPACE environment variable
export HELM_NAMESPACE=production
helm install my-release ./chart

# Via kubeconfig context (Helm inherits it)
kubectl config set-context --current --namespace=production
helm install my-release ./chart
```

### 6.11.2 `--create-namespace`

By default, Helm **does not** create namespaces. The target namespace must pre-exist or you must pass `--create-namespace`:

```bash
helm install my-release ./chart --namespace staging --create-namespace
```

**Production Note:** In CI/CD, always include `--create-namespace` for new environments. For production, consider managing namespaces separately (e.g., via Terraform or a dedicated RBAC setup) for auditability.

---

## 6.12 Release Ownership

Helm labels every resource it creates with standard labels and annotations:

| Label / Annotation | Purpose |
|---|---|
| `app.kubernetes.io/managed-by: Helm` | Identifies the resource as Helm-managed |
| `release: <release-name>` | Links the resource to its release |
| `helm.sh/chart: <chart-name>-<version>` | Identifies the chart |
| `app.kubernetes.io/instance: <release-name>` | Instance identifier |
| `app.kubernetes.io/version: <app-version>` | Application version |

### 6.12.1 Ownership Transfer

Moving a release between Helm instances (e.g., splitting a monorepo chart) requires transferring release ownership:

1. **Export** the release state from the old instance.
2. **Import** into the new instance using the storage backend (Secrets/ConfigMaps).
3. **Verify** `helm list` shows the release under the new instance.

This is an advanced operation. In most cases, it is simpler to uninstall from the old instance and re-install from the new one.

---

## 6.13 Failed Releases

### 6.13.1 Common Causes

| Cause | Symptom | Resolution |
|---|---|---|
| Invalid YAML in values | `Error: YAML parse error` | Validate values with `yamllint` or `helm lint` |
| Missing CRD | `Error: the server could not find the requested resource` | Install CRDs first, then the chart |
| Resource quota exceeded | `Error: exceeded quota` | Increase quota or reduce resource requests |
| RBAC denied | `Error: User cannot create resource` | Ensure correct RBAC permissions |
| Image pull failure | `ErrImagePull` / `ImagePullBackOff` | Verify image registry and pull secrets |
| Hook failure | `Error: hook pre-install failed` | Inspect hook logs, fix the hook |
| Timeout | `Error: timed out waiting for the condition` | Increase `--timeout` or debug the resource |
| Conflicting resources | `Error: resource already exists` | Remove orphaned resources or use `--force` |
| Helm storage corruption | `Error: "Secret in version "v1" cannot be handled"` | Repair or recreate the release Secret |

### 6.13.2 Debugging Failed Releases

```bash
# 1. Check release status
helm status my-release --show-desc

# 2. View history to see what went wrong
helm history my-release

# 3. Inspect rendered manifests of the failed revision
helm get manifest my-release --revision 5

# 4. Check Kubernetes resources directly
kubectl get all -l "release=my-release" -n my-ns
kubectl describe deployment my-release -n my-ns

# 5. View pod logs
kubectl logs -l "release=my-release" -n my-ns

# 6. For hook failures, check hook resources
kubectl get jobs -l "helm.sh/hook"
kubectl logs job/my-release-pre-install
```

### 6.13.3 Clearing a Failed Release

```bash
# Uninstall the failed release
helm uninstall my-release

# If uninstall fails, delete the release Secret manually
kubectl delete secret -n <namespace> -l "owner=helm,status=failed"

# Or for ConfigMap storage backend
kubectl delete configmap -n <namespace> -l "owner=helm,status=failed"
```

**Note:** Manually deleting a release's storage object (Secret or ConfigMap) will make any Kubnernetes resources created by that release **orphaned**. You must clean up those resources separately.

---

## 6.14 Pending Upgrades

A release can become stuck in `pending-upgrade` state if Helm crashes or is interrupted during an upgrade. In this state, further upgrades are blocked.

### 6.14.1 Symptoms

```bash
$ helm upgrade my-release ./chart
Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
```

### 6.14.2 Resolution

```bash
# Roll back to the last known-good revision
helm rollback my-release

# Or manually clear the pending state
kubectl delete secret -n <namespace> \
  sh.helm.release.v1.my-release.v<revision>
```

After clearing, you can upgrade again normally.

**Warning:** Only delete the pending revision's Secret (typically the highest revision number). Do not delete the base release Secret (lowest revision number) unless you intend to fully remove the release.

---

## 6.15 Superseded Revisions

Every time you upgrade, the previous revision is marked `superseded`. These accumulate and consume storage.

### 6.15.1 Cleaning Up Old Revisions

```bash
# Set --history-max during upgrade
helm upgrade my-release ./chart --history-max 5

# Or use helm rollback --history-max for existing releases
# (Note: helm rollback does not have --history-max; you need to upgrade to change it)

# Old revisions auto-purge; to confirm:
helm history my-release
```

**Production Note:** Set `--history-max` to a reasonable value (10-20) to balance the ability to roll back far enough against storage bloat. For frequently deployed services (multiple times per day), consider a higher value (25-50). Each revision is a single Secret/ConfigMap, so storage cost is minimal.

---

## 6.16 Rollback

### 6.16.1 Basic Syntax

```
helm rollback RELEASE [REVISION] [flags]
```

If `REVISION` is omitted, Helm rolls back to the previous revision.

```bash
# Roll back to revision 3
helm rollback my-release 3

# Roll back to the previous revision
helm rollback my-release
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--cleanup-on-fail` | bool | `false` | Delete newly created resources if the rollback fails. |
| `--dry-run` | bool | `false` | Simulate the rollback. |
| `--force` | bool | `false` | Force resource update through delete-and-recreate. |
| `--history-max` | int | `10` | Maximum revisions to retain. |
| `--namespace` / `-n` | string | `default` | Namespace. |
| `--no-hooks` | bool | `false` | Skip hooks. |
| `--recreate-pods` | bool | `false` | Delete and recreate pods (not just rollout restart). |
| `--timeout` | duration | `5m0s` | Timeout for the operation. |
| `--wait` | bool | `false` | Wait for resources to become ready. |
| `--wait-for-jobs` | bool | `false` | Wait for Jobs to complete. |

### 6.16.2 Rollback in CI/CD

```bash
# Emergency rollback script
#!/bin/bash
RELEASE=$1
NAMESPACE=$2

# Find the last deployed revision (non-failed)
LAST_GOOD=$(helm history $RELEASE -n $NAMESPACE -o json | \
  jq '[.[] | select(.status == "deployed" or .status == "superseded")] | last | .revision')

helm rollback $RELEASE $LAST_GOOD -n $NAMESPACE --wait
```

---

## 6.17 Release Storage Drivers

Helm stores release metadata in **storage objects** (revisions) within the Kubernetes cluster. The storage backend is configurable.

### 6.17.1 Available Drivers

| Driver | Storage Type | Default | Pros | Cons |
|---|---|---|---|---|
| **Secret** | Kubernetes Secrets | **Yes** | Encrypted at rest (if cluster encryption is enabled); base64-encoded by default | Slightly larger than ConfigMaps due to base64 encoding |
| **ConfigMap** | Kubernetes ConfigMaps | No | Human-readable, plain text | Not encrypted at rest |
| **SQL** | External SQL database (PostgreSQL, MySQL) | No | Centralized, cluster-independent | Requires external dependency; additional operational overhead |

### 6.17.2 Switching the Storage Driver

Set the `HELM_DRIVER` environment variable:

```bash
export HELM_DRIVER=configmap     # Use ConfigMaps
export HELM_DRIVER=secret        # Use Secrets (default)
export HELM_DRIVER=sql           # Use SQL (requires --storage-driver-sql-* flags)
```

**SQL driver connection details:**

```bash
export HELM_DRIVER=sql
export HELM_DRIVER_SQL_CONNECTION_STRING="postgresql://helm:password@localhost:5432/helm?sslmode=disable"
export HELM_DRIVER_SQL_DIALECT=postgres

helm list
```

### 6.17.3 Release Secret Structure

For the Secret driver, each revision is stored as a Secret named:

```
sh.helm.release.v1.<release-name>.v<revision>
```

Example secrets in a namespace:

```
sh.helm.release.v1.my-app.v1
sh.helm.release.v1.my-app.v2
sh.helm.release.v1.my-app.v3
sh.helm.release.v1.redis.v1
```

The Secret contains:
- The chart (gzipped, base64-encoded)
- The merged values
- Release metadata (name, namespace, version, status)

**Exam Tip:** The release Secret format is `<prefix>.v<version>.<name>.v<revision>`. Knowing this is useful for troubleshooting and is commonly tested on CKA/CKAD exams.

### 6.17.4 Storage Backend Comparison

| Feature | Secret | ConfigMap | SQL |
|---|---|---|---|
| Encryption at rest | Base64 (cluster-side encryption if enabled) | Plain text | Database-dependent |
| Size limit | 1 MB | 1 MB | Database-dependent |
| External dependency | None | None | Required |
| Multi-cluster support | No | No | Yes (single DB shared across clusters) |
| Visibility | `kubectl get secret` | `kubectl get configmap` | SQL queries |
| Default | Yes | No | No |

---

## 6.18 Complete Release Lifecycle Diagram (ASCII)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           HELM RELEASE LIFECYCLE                             │
└──────────────────────────────────────────────────────────────────────────────┘

                        ┌─────────────────────┐
                        │   helm install       │
                        │                     │
                        │   Chart + Values +   │
                        │   Namespace + Name   │
                        └──────────┬──────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │   Revision 1         │
                        │   STATUS: deployed   │
                        │   (Secret/CM stored) │
                        └──────────┬──────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                    ▼              ▼              ▼
           ┌────────────┐  ┌────────────┐  ┌────────────┐
           │helm upgrade│  │helm upgrade│  │helm upgrade│
           │--install   │  │(new values)│  │(new chart) │
           └──────┬─────┘  └──────┬─────┘  └──────┬─────┘
                  │               │               │
                  ▼               ▼               ▼
           ┌─────────────────────────────────────────┐
           │  Revision N                               │
           │  STATUS: deployed                         │
           │  Previous revision → STATUS: superseded   │
           └──────────────────┬──────────────────────┘
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
       ┌────────────┐  ┌────────────┐  ┌────────────┐
       │helm        │  │helm        │  │helm        │
       │rollback    │  │uninstall   │  │uninstall   │
       │(prev rev)  │  │            │  │--keep-hist │
       └──────┬─────┘  └──────┬─────┘  └──────┬─────┘
              │               │                │
              ▼               ▼                ▼
       ┌──────────────────────────────────────────┐
       │  Revision N+1                             │
       │  STATUS: deployed                         │
       │  (Restored from older revision's values)  │
       └──────────────────────────────────────────┘

              ┌───────────────┐
              │  release is    │
              │  gone from     │
              │  cluster       │
              └───────────────┘

                              ┌───────────────┐
                              │  release in    │
                              │  status:       │
                              │  uninstalled   │
                              │  (history kept)│
                              └───────────────┘
```

### 6.18.1 State Transition Table

| From State | Action | To State |
|---|---|---|
| (none) | `helm install` | `deployed` |
| `deployed` | `helm upgrade` | `deployed` (old rev → `superseded`) |
| `deployed` | `helm rollback` | `deployed` (restored from revision) |
| `deployed` | `helm uninstall` | `uninstalled` |
| `deployed` | `helm uninstall --keep-history` | `uninstalled` (history retained) |
| `pending-upgrade` | (stuck/locked) | `failed` or manually cleared |
| `pending-install` | (stuck/locked) | `failed` or manually cleared |
| `failed` | `helm uninstall` | `uninstalled` |
| `failed` | `helm upgrade` | `deployed` (if upgrade succeeds) |
| `uninstalled` | `helm install` (same name) | `deployed` (revision 1, fresh) |

---

## 6.19 Summary

- A **release** is a deployed instance of a chart in a namespace with a unique name and specific values.
- `helm install` creates a release, `helm upgrade` revises it, `helm rollback` reverts it, and `helm uninstall` removes it.
- The **`--install`** flag on `helm upgrade` is the standard idempotent deployment pattern.
- The **Secret** driver is the default and recommended storage backend for most use cases.
- Use `--atomic` and `--wait` in production to ensure deployment integrity.
- Release history is capped by `--history-max` (default 10); clean up superseded revisions to avoid storage bloat.
- The `helm get` commands provide complete introspection into any revision of any release.
- Understanding **failed releases** and **pending states** is critical for production troubleshooting.
