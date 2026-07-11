# Chapter 21: Helm Cheat Sheets

## Table of Contents

1. [Commands Quick Reference](#commands-quick-reference)
2. [Common Scenarios](#common-scenarios)
3. [Task → Best Command](#task--best-command)
4. [Common Errors → Cause → Fix](#common-errors--cause--fix)
5. [Values Flags Comparison](#values-flags-comparison)
6. [Upgrade Strategy Flags](#upgrade-strategy-flags)
7. [Release State Table](#release-state-table)
8. [Environment Variables](#environment-variables)
9. [Hook Types Reference](#hook-types-reference)
10. [ASCII Reference Cards](#ascii-reference-cards)

---

## Commands Quick Reference

| Command | Purpose | Most Important Flags | CKA Frequency |
|---|---|---|---|
| `helm install` | Deploy a chart as a new release | `-n`, `--set`, `-f`, `--dry-run`, `--debug`, `--atomic`, `--wait`, `--timeout`, `--create-namespace` | **Very High** |
| `helm upgrade` | Modify an existing release | `-n`, `--set`, `-f`, `--install`, `--atomic`, `--wait`, `--reuse-values`, `--dry-run`, `--timeout`, `--force`, `--cleanup-on-fail` | **Very High** |
| `helm rollback` | Revert to a previous revision | `-n`, `--wait`, `--dry-run`, `--timeout` | **High** |
| `helm list` | List releases | `-A`, `-n`, `--all`, `--failed`, `--pending`, `--deployed`, `-q`, `-o json/yaml`, `--filter` | **Very High** |
| `helm history` | Show revision history of a release | `-n`, `--max`, `-o json/yaml` | **High** |
| `helm get values` | Retrieve values used by a release | `-n`, `--all`, `--revision`, `-o json/yaml` | Medium |
| `helm get manifest` | Retrieve the rendered Kubernetes YAML | `-n`, `--revision` | Medium |
| `helm get notes` | Retrieve the release notes | `-n`, `--revision` | Low |
| `helm get all` | Retrieve all information about a release | `-n`, `--revision`, `-o yaml` | Low |
| `helm uninstall` | Remove a release and all its resources | `-n`, `--keep-history`, `--dry-run` | Medium |
| `helm repo add` | Add a chart repository | `--username`, `--password`, `--no-update` | **High** |
| `helm repo update` | Refresh the local repository index | (none) | **High** |
| `helm repo list` | List configured repositories | `-o json/yaml` | Low |
| `helm repo remove` | Remove a configured repository | (none) | Low |
| `helm repo index` | Generate an index.yaml for a chart repo | `--url`, `--merge` | Rare (CKA) |
| `helm search repo` | Search for charts in configured repos | `--versions`, `-o json/yaml`, `--regexp` | **High** |
| `helm search hub` | Search Artifact Hub for charts | `--max-col-width`, `-o json/yaml` | Rare (CKA) |
| `helm status` | Show the current status of a release | `-n`, `--revision`, `-o json/yaml`, `--show-resources`, `--show-desc` | Medium |
| `helm show values` | Show the default values of a chart | `--version` | Medium |
| `helm show chart` | Show the chart definition (Chart.yaml) | `--version` | Low |
| `helm show all` | Show all information about a chart | `--version` | Low |
| `helm show readme` | Show the README of a chart | `--version` | Low |
| `helm template` | Render chart templates locally | `-f`, `--set`, `-n`, `--release-name`, `--debug`, `--validate`, `--skip-tests`, `--show-only` | Low |
| `helm pull` | Download a chart from a repository | `--version`, `--untar`, `--untardir`, `--destination`, `--username`, `--password` | Rare (CKA) |
| `helm test` | Run the tests for a release | `-n`, `--timeout`, `--logs` | Rare (CKA) |
| `helm lint` | Validate a chart for issues | `--strict`, `--quiet`, `--with-subcharts` | Rare (CKA) |
| `helm package` | Package a chart directory into a .tgz | `--destination`, `--sign`, `--key`, `--keyring`, `--version`, `--app-version` | Rare (CKA) |
| `helm dependency list` | List chart dependencies | (none) | Rare (CKA) |
| `helm dependency update` | Download chart dependencies into charts/ | `--skip-refresh`, `--verify` | Rare (CKA) |
| `helm dependency build` | Alias for `helm dependency update` | (same as update) | Rare (CKA) |
| `helm verify` | Verify a signed chart | `--keyring` | Very Rare |
| `helm plugin list` | List installed Helm plugins | (none) | Very Rare |
| `helm plugin install` | Install a Helm plugin | `--version` | Very Rare |
| `helm plugin update` | Update installed Helm plugins | (none) | Very Rare |
| `helm plugin uninstall` | Uninstall a Helm plugin | (none) | Very Rare |
| `helm env` | Show Helm environment variables | (none) | Very Rare |
| `helm version` | Show the Helm client version | `--short`, `--template` | Rare (CKA) |
| `helm completion` | Generate shell autocompletion scripts | `bash`, `zsh`, `fish` | Very Rare |

---

## Common Scenarios

| # | Scenario | Command |
|---|---|---|
| 1 | Install a chart from a repository | `helm install my-release bitnami/nginx -n my-ns` |
| 2 | Install a chart from a local directory | `helm install my-release ./chart -n my-ns` |
| 3 | Install with custom values from CLI | `helm install my-release bitnami/nginx -n my-ns --set replicaCount=3` |
| 4 | Install with multiple custom values | `helm install my-release bitnami/nginx -n my-ns --set replicaCount=3,service.type=NodePort` |
| 5 | Install with nested values | `helm install my-release bitnami/nginx -n my-ns --set resources.limits.cpu=500m` |
| 6 | Install with a values file | `helm install my-release bitnami/nginx -n my-ns -f values-prod.yaml` |
| 7 | Install with multiple values files (last wins) | `helm install my-release bitnami/nginx -n my-ns -f base.yaml -f prod.yaml` |
| 8 | Install and create namespace | `helm install my-release bitnami/nginx -n my-ns --create-namespace` |
| 9 | Install with --set-string (force string) | `helm install my-release bitnami/nginx -n my-ns --set-string nodeSelector.role=web` |
| 10 | Install with --set-json (complex JSON value) | `helm install my-release bitnami/nginx -n my-ns --set-json 'affinity={"nodeAffinity":{}}'` |
| 11 | Install with --set-file (load value from file) | `helm install my-release bitnami/nginx -n my-ns --set-file config.data=/path/to/file` |
| 12 | Install and wait for all resources ready | `helm install my-release bitnami/nginx -n my-ns --wait --timeout 10m` |
| 13 | Install with atomic rollback on failure | `helm install my-release bitnami/nginx -n my-ns --atomic --timeout 10m` |
| 14 | Upgrade a release with new values | `helm upgrade my-release bitnami/nginx -n my-ns --set replicaCount=5` |
| 15 | Upgrade and keep all existing values | `helm upgrade my-release bitnami/nginx -n my-ns --reuse-values --set image.tag=v2` |
| 16 | Upgrade or install if not present (idempotent) | `helm upgrade --install my-release bitnami/nginx -n my-ns --set replicaCount=3` |
| 17 | Upgrade with atomic rollback | `helm upgrade my-release bitnami/nginx -n my-ns --atomic --wait --timeout 10m` |
| 18 | Dry-run an install (validate without applying) | `helm install my-release bitnami/nginx -n my-ns --dry-run --debug` |
| 19 | Dry-run an upgrade | `helm upgrade my-release bitnami/nginx -n my-ns --dry-run` |
| 20 | Rollback to a specific revision | `helm rollback my-release 3 -n my-ns` |
| 21 | Rollback to previous revision (programmatic) | `helm rollback my-release $(helm history my-release -n my-ns -o json \| jq '.[-2].revision') -n my-ns` |
| 22 | List all releases across all namespaces | `helm list -A` |
| 23 | List all releases in a namespace | `helm list -n my-ns` |
| 24 | List only failed releases | `helm list -A --failed` |
| 25 | List only pending releases | `helm list -A --pending` |
| 26 | List releases in JSON format | `helm list -A -o json` |
| 27 | Show history of a release | `helm history my-release -n my-ns` |
| 28 | Show last 3 revisions only | `helm history my-release -n my-ns --max 3` |
| 29 | Get values used by a release (user-supplied only) | `helm get values my-release -n my-ns` |
| 30 | Get all values including chart defaults | `helm get values my-release -n my-ns --all` |
| 31 | Get the rendered Kubernetes manifests | `helm get manifest my-release -n my-ns` |
| 32 | Get the release notes | `helm get notes my-release -n my-ns` |
| 33 | Get status with resource details | `helm status my-release -n my-ns --show-resources` |
| 34 | Uninstall a release | `helm uninstall my-release -n my-ns` |
| 35 | Uninstall but keep history record | `helm uninstall my-release -n my-ns --keep-history` |
| 36 | Add a Helm repository | `helm repo add bitnami https://charts.bitnami.com/bitnami` |
| 37 | Update all repositories | `helm repo update` |
| 38 | List configured repositories | `helm repo list` |
| 39 | Remove a repository | `helm repo remove bitnami` |
| 40 | Search for a chart | `helm search repo nginx` |
| 41 | Search and show all versions | `helm search repo nginx --versions` |
| 42 | Show default values of a chart | `helm show values bitnami/nginx` |
| 43 | Show chart metadata (Chart.yaml) | `helm show chart bitnami/nginx` |
| 44 | Render templates locally | `helm template my-release bitnami/nginx -n my-ns -f values.yaml` |
| 45 | Render only one template file | `helm template my-release ./chart --show-only templates/deployment.yaml` |
| 46 | Render and validate against cluster API | `helm template my-release ./chart --validate` |
| 47 | Pull a chart tarball to local filesystem | `helm pull bitnami/nginx --version 18.0.0` |
| 48 | Pull and extract a chart | `helm pull bitnami/nginx --version 18.0.0 --untar --untardir ./charts/` |
| 49 | Run chart tests | `helm test my-release -n my-ns` |
| 50 | Run chart tests with log output | `helm test my-release -n my-ns --logs` |
| 51 | Lint a chart with strict mode | `helm lint ./chart --strict` |
| 52 | Package a chart into a .tgz archive | `helm package ./chart -d ./packages/` |
| 53 | Check Helm version | `helm version --short` |
| 54 | Generate bash completion | `source <(helm completion bash)` |
| 55 | Install with history-max limit | `helm install my-release bitnami/nginx -n my-ns --history-max 10` |

---

## Task → Best Command

| Task | Best Command | Notes |
|---|---|---|
| Deploy a new application | `helm install` | Use `--create-namespace` if namespace doesn't exist |
| Modify a running deployment | `helm upgrade` | Use `--reuse-values` if you want to keep existing values |
| Both deploy and modify (CI/CD) | `helm upgrade --install` | Idempotent—safe to run repeatedly |
| Revert a bad deployment | `helm rollback` | Find the target revision with `helm history` first |
| Test before applying | `--dry-run --debug` | Applies to `install`, `upgrade`, and `rollback` |
| Find what's running | `helm list -A` | Best bird's-eye view of the cluster |
| Find a chart in a repository | `helm search repo` | Faster than `helm search hub` |
| See what you configured | `helm get values -n NS RELEASE` | Add `--all` for computed values |
| See what's deployed | `helm get manifest -n NS RELEASE` | Full rendered Kubernetes YAML |
| Check deployment health | `helm status -n NS RELEASE` | Shows state, resources, notes |
| Debug a failed deploy | `helm history` + `helm get manifest` + `kubectl describe` | Trace the chain of events |
| Save current values for later | `helm get values -n NS RELEASE --all -o yaml > backup.yaml` | Useful before risky upgrades |
| Export rendered YAML without deploying | `helm template` | Use `--show-only` for specific template files |
| Clean up completely | `helm uninstall -n NS RELEASE` | Optionally add `--keep-history` |
| Validate chart structure | `helm lint ./chart --strict` | Catches template errors, missing files |
| See available chart config options | `helm show values CHART` | Pipe to `less` for browsing |
| Download a chart for inspection | `helm pull CHART --untar` | Extract and review templates locally |
| Test deployed release | `helm test -n NS RELEASE` | Runs test pods defined in `templates/tests/` |
| Compare chart versions | `helm search repo CHART --versions` | See all available versions |
| Is this release managed by Helm? | `helm list -A \| grep NAME` | Helm-managed releases appear in `helm list` |
| Recover from hung upgrade | `helm rollback -n NS RELEASE` | Rolls back to last successful revision |
| Audit who deployed what | `helm history -n NS RELEASE` | Shows timestamps, status, and chart versions |
| Verify values before install | `helm show values CHART` then `--dry-run` | Check required values exist in your config |
| Add a private repository | `helm repo add NAME URL --username U --password P` | Use for private OCI or ChartMuseum repos |
| Force delete stuck release | Delete the release Secret manually | `kubectl delete secret -n NS sh.helm.release.v1.RELEASE.vN` |

---

## Common Errors → Cause → Fix

| # | Error Message | Cause | Solution |
|---|---|---|---|
| 1 | `Error: cannot re-use a name that is still in use` | Attempting `helm install` with an existing release name | Use `helm upgrade --install` or `helm uninstall` the old release first |
| 2 | `Error: "RELEASE" has no deployed releases` | Running `helm history` or `helm status` on a nonexistent release | Check spelling and namespace with `helm list -A` |
| 3 | `Error: unknown flag: --name` | Using Helm 2 syntax on Helm 3 (`--name` was removed) | In Helm 3, the release name is a positional argument: `helm install NAME CHART` |
| 4 | `Error: failed to download "bitnami/nginx"` | Repository not added or local cache is stale | Run `helm repo add bitnami URL` then `helm repo update` |
| 5 | `Error: chart "nginx" version "1.0.0" not found` | Requested version doesn't exist in the repository | Run `helm search repo nginx --versions` to see available versions |
| 6 | `Error: Kubernetes cluster unreachable: connection refused` | `kubectl` context points to a nonexistent or unreachable cluster | Run `kubectl cluster-info`; switch context with `kubectl config use-context` |
| 7 | `Error: release: not found` | Release doesn't exist in the specified namespace | Check with `helm list -A` to find the correct namespace or release name |
| 8 | `Error: UPGRADE FAILED: timed out waiting for the condition` | Pods didn't become ready within `--timeout` | Increase `--timeout`, check `kubectl describe pod` for stuck pods |
| 9 | `Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest` | Template rendering error (syntax error, missing values) | Run `helm template --debug` to see the exact rendering error |
| 10 | `Error: YAML parse error on mychart/templates/deployment.yaml: error converting YAML to JSON` | Invalid YAML in a template (bad indentation, missing quotes) | Run `helm lint --strict`; check line numbers in the error message |
| 11 | `Error: create: failed to create: Secret "sh.helm.release.v1..." already exists` | Corrupted Helm state (bug or manual Secret manipulation) | Delete the stale Secret: `kubectl delete secret -n NS -l owner=helm,name=RELEASE` |
| 12 | `Error: another operation (install/upgrade/rollback) is in progress` | Previous Helm operation is still running or hung | Check `helm history -n NS RELEASE` for pending status; force cleanup of stuck hooks |
| 13 | `Error: UPGRADE FAILED: has no deployed releases, but has a pending upgrade` | Previous upgrade was interrupted, leaving release in pending state | Rollback to clear state: `helm rollback RELEASE -n NS` |
| 14 | `Error: looks like "https://..." is not a valid chart repository or cannot be reached` | Invalid repository URL or network connectivity issue | Verify the URL in a browser/curl; check `helm repo list` for typos |
| 15 | `Error: failed to fetch https://.../index.yaml: 403 Forbidden` | Authentication required for a private repository | Add credentials: `helm repo add NAME URL --username U --password P` |
| 16 | `Error: "values.yaml" is not a valid YAML file` | Malformed YAML in the values file | Validate YAML syntax with `yamllint values.yaml` or `python -c "import yaml; yaml.safe_load(open('values.yaml'))"` |
| 17 | `Error: schema validation failed: replicaCount: Invalid type. Expected: integer, given: string` | `values.schema.json` rejected the provided values | Use correct types: integers without quotes, strings with quotes. Check schema requirements. |
| 18 | `Error: cannot patch "my-deployment" with kind Deployment` | Kubernetes cannot apply the patch; field may be immutable or in conflict | Investigate with `kubectl describe deployment`; sometimes `--force` is needed |
| 19 | `Error: uninstall: Release not loaded: RELEASE: release: not found` | Trying to uninstall a release that doesn't exist | Check with `helm list -A`; verify namespace and release name spelling |
| 20 | `Warning: Merged values are not additive for lists` (not a hard error) | You expected `--set` to append to a list; instead it replaced the entire list | Specify the entire list via `-f` values file or use `--set-json` |
| 21 | `Error: local chart "mychart" has an invalid version: "1.2"` | `version` field in Chart.yaml is not a valid SemVer string | Fix Chart.yaml: `version: 1.2.0` (must be MAJOR.MINOR.PATCH) |
| 22 | `Error: "/path/to/chart" has no templates/ directory` | The chart directory doesn't contain a `templates/` folder | Check the chart structure: must have `Chart.yaml` and `templates/` at minimum |
| 23 | `Error: found in Chart.yaml, but missing in charts/ directory: postgresql` | Dependency declared in Chart.yaml but not downloaded | Run `helm dependency update` to download dependencies into `charts/` |
| 24 | `Error: execution error at (...): can't evaluate field Values in type string` | Template uses `.Values.something` in a context where `.Values` is not available | Check template scoping: inside `range` or `with` blocks, context changes. Use `$.Values` for root scope. |
| 25 | `Error: UPGRADE FAILED: rendered manifests contain a resource that already exists` | Chart creates a resource that already exists from another source | Differentiate names (use `fullname` helper), or pre-delete the conflicting resource |
| 26 | `Error: could not find a ready tiller pod` | Helm 2 error: Tiller is not running (Helm 3 doesn't use Tiller) | Upgrade to Helm 3. If using Helm 2, install Tiller: `helm init` |
| 27 | `Error: context deadline exceeded` | Network timeout talking to the Kubernetes API | Check cluster connectivity; check proxy settings; increase `--timeout` |
| 28 | `Error: UPGRADE FAILED: error validating "": error validating data: unknown object type` | CRD not installed for a custom resource referenced by the chart | Install the CRD first; check the chart's CRDs directory or install separately |
| 29 | `Error: UPGRADE FAILED: cannot re-use a name that is still in use` (on upgrade) | Corrupted release state—Helm thinks the release name is already in use | Delete problematic Secrets: `kubectl delete secret -n NS sh.helm.release.v1.RELEASE.vN` |
| 30 | `Error: error installing: the server could not find the requested resource` | Your cluster version doesn't support a Kubernetes resource used by the chart (e.g., Ingress v1 on an old cluster) | Match chart's Kubernetes version requirements; upgrade the cluster or use an older chart |

---

## Values Flags Comparison

| Flag | Purpose | Type Handling | Example | When to Use |
|---|---|---|---|---|
| `--set key=value` | Set a value from CLI | Auto-detects YAML type (int, bool, list) | `--set replicaCount=3` | Quick overrides of simple values |
| `--set-string key=value` | Force value as string | **Always string** | `--set-string name=12345` | When a value looks like a number/bool but must be a string |
| `--set-json key=json` | Set a value as JSON | Parses JSON literals | `--set-json 'resources={"limits":{"cpu":"500m"}}'` | Complex nested values, arrays, or objects |
| `--set-file key=path` | Load value from file content | Read file as text | `--set-file config.data=./config.ini` | Config file contents, large text blocks, certificates |
| `-f / --values file` | Load values from YAML file | Parses YAML file | `-f values-prod.yaml` | Environment-specific configs, production values |
| `--set-literal key=value` | Set with special characters preserved | Literal string (no parsing) | `--set-literal password='p@ss$$w0rd!'` | Values containing `$`, `\`, or other shell-special chars |

### `--set` Type Resolution

| Input | YAML Parsed As | To Force String |
|---|---|---|
| `--set key=3` | integer `3` | `--set-string key=3` or `--set key="3"` |
| `--set key=3.14` | float `3.14` | `--set-string key=3.14` |
| `--set key=true` | boolean `true` | `--set-string key=true` |
| `--set key=false` | boolean `false` | `--set-string key=false` |
| `--set key=null` | null | `--set-string key=null` |
| `--set key=hello` | string `hello` | No need, already a string |
| `--set key=[a,b]` | list `[a, b]` | `--set-string key=[a,b]` |
| `--set key={a,b}` | list `[a, b]` | `--set-string key={a,b}` |

---

## Upgrade Strategy Flags

| Flag | Effect | Use Case | Risk |
|---|---|---|---|
| `--atomic` | Rollback automatically on failure | Production deployments, CI/CD | Medium: rollback itself could fail |
| `--wait` | Block until all resources are ready | Production deployments, verification steps | Medium: can timeout on slow clusters |
| `--timeout DURATION` | Max time to wait (default 5m) | Large charts, slow image pulls | Low: just set a reasonable value |
| `--dry-run` | Simulate without applying | Pre-deployment validation | **None** (client-side only) |
| `--debug` | Verbose output including templates | Debugging rendering issues | **None** (client-side only) |
| `--force` | Delete and recreate resources that can't be patched | Stuck deployments, immutable fields changed | **High**: causes downtime |
| `--reuse-values` | Keep all previous `--set` values | Quick single-value changes | **Medium**: carries forward unknown state |
| `--reset-values` | Reset to chart defaults before applying new values | Major version upgrades, starting fresh | **Medium**: may remove required customizations |
| `--cleanup-on-fail` | Delete new resources created during a failed upgrade | Keeping the cluster clean | Low: useful with `--atomic` |
| `--install` | Install if release doesn't exist (with `upgrade`) | Idempotent CI/CD scripts | Low: recommended for all CI pipelines |
| `--no-hooks` | Skip running hooks | Debugging stuck hooks | **High**: hooks exist for a reason (db migrations, etc.) |
| `--history-max N` | Limit stored revisions to N | Preventing Secret bloat | Low: set to 5-10 for production |
| `--create-namespace` | Create the namespace if it doesn't exist | First deployment to a new environment | Low: convenient |
| `--dependency-update` | Run `helm dependency update` before install/upgrade | Charts with dependencies that changed | Low: ensures deps are current |

### Upgrade Strategy Decision Matrix

| Situation | Recommended Flags |
|---|---|
| Routine production upgrade | `--atomic --wait --timeout 10m` |
| Urgent hotfix (known working change) | `--wait --timeout 5m` |
| Major version upgrade | `--dry-run --debug` first; then `--reset-values --wait --timeout 15m` |
| CI/CD pipeline | `--install --wait --timeout 10m` |
| Quick single-value change | `--reuse-values --set key=value` |
| Debugging a failed upgrade | `--dry-run --debug` first; then fix and retry |
| Stuck deployment (immutable field) | `--force` (last resort, expect brief downtime) |
| First deployment to new namespace | `--create-namespace --wait --timeout 10m` |

---

## Release State Table

| Status | Meaning | Next Action |
|---|---|---|
| `deployed` | Release is installed and healthy | No action needed. Monitor as usual. |
| `failed` | Install or upgrade failed (timeout, error, hook failure) | Investigate `helm status RELEASE -n NS`. Run `helm rollback` to revert. |
| `superseded` | Previous revision; a newer revision has been deployed | No action needed. Old revisions exist for rollback. |
| `pending-install` | Installation is in progress | Wait. If stuck > 5 min, check for hung hooks with `kubectl get jobs -n NS`. |
| `pending-upgrade` | Upgrade is in progress | Wait. If stuck > 5 min, check `--wait` timeout. Rollback if necessary. |
| `pending-rollback` | Rollback is in progress | Wait. If stuck, the target revision may be invalid. Check `helm history`. |
| `uninstalling` | Release is being deleted (via `helm uninstall`) | Wait. If stuck, the `pre-delete` hook may be hung. |
| `uninstalled` | Release was successfully removed | No further action. Release name is available for reuse. |
| `unknown` | Helm can't determine the release state (corrupted Secret) | Inspect release Secrets: `kubectl get secrets -n NS -l owner=helm`. Manual cleanup may be needed. |

### Status Transition Flow

```
helm install → pending-install → deployed
                                  └── (failure) → failed

helm upgrade (from deployed) → pending-upgrade → deployed
                                                  └── (failure, with --atomic) → rollback → deployed
                                                  └── (failure, without --atomic) → failed

helm rollback → pending-rollback → deployed

helm uninstall → uninstalling → uninstalled
```

---

## Environment Variables

| Variable | Purpose | Example | Default |
|---|---|---|---|
| `HELM_CACHE_HOME` | Directory for cached repository indexes and charts | `/var/cache/helm` | `~/.cache/helm` |
| `HELM_CONFIG_HOME` | Directory for Helm configuration (repositories.yaml) | `/etc/helm` | `~/.config/helm` |
| `HELM_DATA_HOME` | Directory for Helm data (plugins, starters) | `/var/lib/helm` | `~/.local/share/helm` |
| `HELM_DRIVER` | Storage backend for release information | `secret`, `configmap`, `memory`, `sql` | `secret` |
| `HELM_DRIVER_SQL_CONNECTION_STRING` | SQL connection string (when HELM_DRIVER=sql) | `postgresql://user:pass@host:5432/helm` | (none) |
| `HELM_MAX_HISTORY` | Maximum release history (same as `--history-max`) | `10` | 256 (Helm default, effectively unlimited) |
| `HELM_NAMESPACE` | Default namespace for Helm operations | `my-namespace` | Uses `kubectl` current context namespace |
| `HELM_KUBECONTEXT` | Kubernetes context Helm should use | `prod-cluster` | Uses `kubectl` current context |
| `HELM_KUBEAPISERVER` | Override the Kubernetes API server URL | `https://k8s-prod.example.com:6443` | From kubeconfig |
| `HELM_KUBETOKEN` | Bearer token for Kubernetes API authentication | `eyJhbGciOi...` | From kubeconfig |
| `HELM_KUBEASGROUPS` | Groups for impersonation (JSON array) | `["system:masters"]` | (none) |
| `HELM_KUBEASUSER` | Username for impersonation | `admin` | (none) |
| `HELM_KUBECAFILE` | Path to CA certificate for Kubernetes API | `/etc/ssl/ca.pem` | From kubeconfig |
| `HELM_BURST_LIMIT` | Client-side throttle for requests per second | `100` | 100 |
| `HELM_REGISTRY_CONFIG` | Path to OCI registry config file | `/etc/docker/config.json` | `~/.config/helm/registry/config.json` |
| `HELM_REPOSITORY_CACHE` | Path to the repository cache directory | `/tmp/helm-cache` | `$HELM_CACHE_HOME/repository` |
| `HELM_REPOSITORY_CONFIG` | Path to the repositories.yaml file | `/etc/helm/repositories.yaml` | `$HELM_CONFIG_HOME/repositories.yaml` |
| `HELM_PLUGINS` | Path to the plugins directory | `/usr/local/helm/plugins` | `$HELM_DATA_HOME/plugins` |

### Environment Variable Patterns for CKA

```bash
# Workspace organization—keep Helm artifacts away from home directory
export HELM_CACHE_HOME="$HOME/.cache/helm"
export HELM_CONFIG_HOME="$HOME/.config/helm"
export HELM_DATA_HOME="$HOME/.local/share/helm"

# CI/CD—limit history to prevent Secret bloat
export HELM_MAX_HISTORY=10

# CI/CD—force a specific namespace
export HELM_NAMESPACE=production

# Multi-cluster—target a specific cluster context
export HELM_KUBECONTEXT=prod-cluster

# Troubleshooting—use memory driver to test without persisting state
export HELM_DRIVER=memory
# WARNING: releases are lost when the process exits!
```

---

## Hook Types Reference

Helm hooks are Kubernetes resources annotated with `helm.sh/hook`. They run at specific points in the release lifecycle.

### Hook Execution Points

| Hook | When It Runs | Common Use Case | Resource Types |
|---|---|---|---|
| `pre-install` | After templates are rendered but **before** any resources are created | Create prerequisite ConfigMaps, Secrets, or CRDs | Job, Pod, ConfigMap, Secret |
| `post-install` | After all resources are created and `--wait` conditions are met | Run database migrations, smoke tests, seed data | Job |
| `pre-delete` | Before any resources are deleted during `helm uninstall` | Backup data, deregister from service discovery, drain connections | Job, Pod |
| `post-delete` | After all resources are deleted | Cleanup external resources, notify monitoring systems | Job |
| `pre-upgrade` | After templates are rendered but before any resources are updated | Pre-upgrade database backup, pre-upgrade schema checks | Job |
| `post-upgrade` | After all resources are upgraded and `--wait` conditions are met | Post-migration validation, cache warming, integration tests | Job |
| `pre-rollback` | Before a rollback begins | Backup current state before reverting | Job |
| `post-rollback` | After rollback completes | Verify rollback was successful, notify | Job |
| `test` | When `helm test` is invoked | Integration tests, smoke tests, health check validation | Pod (with restartPolicy: Never) |
| `test-success` | When a test pod completes successfully (Helm 3.16+) | Post-test notifications, recording test results | Job |
| `test-failure` | When a test pod fails (Helm 3.16+) | Alerting on test failures, cleanup after failed tests | Job |

### Hook Annotation Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-db-migrate
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["./migrate.sh"]
  backoffLimit: 1
```

### Hook Deletion Policies

| Policy | Behavior | Use Case |
|---|---|---|
| `before-hook-creation` | Delete previous hook before creating a new one | Default; prevents duplicate hook resources |
| `hook-succeeded` | Delete the hook resource after it succeeds | Clean up completed migration/test pods |
| `hook-failed` | Delete the hook resource if it fails | Clean up failed hook resources (rare) |

### Hook Weight

Hooks with `helm.sh/hook-weight` determine execution order. Lower numbers run first. Default is 0.

```yaml
annotations:
  "helm.sh/hook": pre-install
  "helm.sh/hook-weight": "-10"  # Runs before hooks with weight 0 or higher
```

**Note:** Hooks of the same type and weight run in parallel. Use different weights to sequence hooks (e.g., create ConfigMap at weight -10, then run migration Job at weight 5, then run test at weight 10).

---

## ASCII Reference Cards

### Keyboard Shortcuts Reference Card

```
╔══════════════════════════════════════════════════════╗
║         EXAM TERMINAL KEYBOARD SHORTCUTS            ║
╠══════════════════════════════════════════════════════╣
║  NAVIGATION                                          ║
║  ───────────                                         ║
║  Ctrl+A  →  Go to beginning of line                 ║
║  Ctrl+E  →  Go to end of line                       ║
║  Ctrl+F  →  Forward one character                   ║
║  Ctrl+B  →  Backward one character                  ║
║  Alt+F   →  Forward one word                        ║
║  Alt+B   →  Backward one word                       ║
║                                                      ║
║  EDITING                                             ║
║  ───────                                             ║
║  Ctrl+U  →  Delete from cursor to line start        ║
║  Ctrl+K  →  Delete from cursor to line end          ║
║  Ctrl+W  →  Delete word before cursor               ║
║  Alt+D   →  Delete word after cursor                ║
║  Ctrl+Y  →  Yank (paste) deleted text               ║
║                                                      ║
║  HISTORY                                             ║
║  ───────                                             ║
║  Ctrl+R  →  Reverse search command history          ║
║  ↑       →  Previous command                        ║
║  ↓       →  Next command                            ║
║  !!      →  Repeat last command                     ║
║  !$      →  Last argument of previous command       ║
║  !helm   →  Last command starting with "helm"       ║
║                                                      ║
║  PROCESS                                             ║
║  ───────                                             ║
║  Ctrl+C  →  Interrupt (SIGINT)                      ║
║  Ctrl+D  →  End-of-file / exit                      ║
║  Ctrl+Z  →  Suspend (SIGTSTP), resume with fg       ║
║  Ctrl+L  →  Clear screen                            ║
╚══════════════════════════════════════════════════════╝
```

### Exam Workflow Quick Reference Card

```
╔══════════════════════════════════════════════════════╗
║       EXAM WORKFLOW: HELM TASK EXECUTION            ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  STEP 1: READ & NOTE                                ║
║  ────────────────────                                ║
║  ☐ Note the NAMESPACE (underline it!)               ║
║  ☐ Note the RELEASE NAME                             ║
║  ☐ Note the CHART NAME (repo/chart or local path)    ║
║  ☐ Note any required --set values                    ║
║                                                      ║
║  STEP 2: REPO CHECK                                  ║
║  ─────────────────                                   ║
║  ☐ helm repo list (is it configured?)               ║
║  ☐ helm repo add NAME URL (if missing)              ║
║  ☐ helm repo update (always!)                       ║
║                                                      ║
║  STEP 3: FIND CHART                                  ║
║  ─────────────────                                   ║
║  ☐ helm search repo KEYWORD                         ║
║  ☐ helm show values CHART (check required values)    ║
║                                                      ║
║  STEP 4: DEPLOY / UPGRADE                            ║
║  ─────────────────────────                           ║
║  ☐ kubectl create namespace NS (if needed)          ║
║  ☐ helm install RELEASE CHART -n NS [--set KEY=V]   ║
║  ☐ OR: helm upgrade --install RELEASE CHART -n NS   ║
║  ☐ ADD: --wait --timeout 5m (if task requires)      ║
║                                                      ║
║  STEP 5: VERIFY                                      ║
║  ─────────────                                       ║
║  ☐ helm list -n NS                                  ║
║  ☐ helm status RELEASE -n NS                        ║
║  ☐ helm get values RELEASE -n NS (right values?)     ║
║  ☐ kubectl get pods -n NS (all Running?)            ║
║                                                      ║
║  STEP 6: TEST (if required)                          ║
║  ───────────────────────                             ║
║  ☐ helm test RELEASE -n NS                          ║
║  ☐ curl/wget the service endpoint                   ║
║                                                      ║
║  ⚡ IF SOMETHING GOES WRONG:                         ║
║  1. helm history RELEASE -n NS (find good rev)      ║
║  2. helm rollback RELEASE REV -n NS (roll back)     ║
║  3. helm get manifest RELEASE -n NS (inspect YAML)  ║
║  4. helm get values RELEASE -n NS --all (check vals)║
║  5. Fix and helm upgrade again                      ║
╚══════════════════════════════════════════════════════╝
```

### Helm Command Skeleton Card

```
╔══════════════════════════════════════════════════════╗
║       THE 6 HELM COMMANDS YOU MUST NEVER LOOK UP     ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  helm install  <release> <chart> -n <ns> [--set ..] ║
║  helm upgrade  <release> <chart> -n <ns> [--set ..] ║
║  helm rollback <release> <rev>   -n <ns>             ║
║  helm list     -n <ns>                               ║
║  helm history  <release> -n <ns>                     ║
║  helm uninstall <release> -n <ns>                    ║
║                                                      ║
║  helm repo add    <name> <url>                       ║
║  helm repo update                                    ║
║  helm search repo <keyword>                          ║
║                                                      ║
║  helm get values   <release> -n <ns> [--all]        ║
║  helm get manifest <release> -n <ns>                 ║
║  helm status       <release> -n <ns>                 ║
║                                                      ║
║  ALWAYS:                                             ║
║  ── -n <namespace> on every command                 ║
║  ── helm repo update before searching/installing     ║
║  ── --dry-run --debug before applying               ║
║  ── helm history before helm rollback               ║
╚══════════════════════════════════════════════════════╝
```

### Pre-Flight Checklist Card

```
╔══════════════════════════════════════════════════════╗
║       MENTAL CHECKLIST (BEFORE EVERY HELM CMD)       ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  [ ]  -n <NAMESPACE> specified?                     ║
║  [ ]  Repo added and updated?                        ║
║  [ ]  Release name correct and spelled right?       ║
║  [ ]  Required --set values provided?               ║
║  [ ]  --dry-run first for this upgrade?             ║
║  [ ]  --atomic needed? (CI/CD = yes)                ║
║  [ ]  --wait needed? (need pods ready = yes)        ║
║  [ ]  Namespace exists? (or --create-namespace)     ║
║                                                      ║
║  GOLDEN RULE:                                        ║
║  Never run helm upgrade in prod without              ║
║    --dry-run --debug first!                          ║
╚══════════════════════════════════════════════════════╝
```

### Debugging Flow Card

```
╔══════════════════════════════════════════════════════╗
║          FAST DEBUGGING FLOW                         ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  ① helm list -n NAMESPACE                           ║
║     ├─ Release found? → Continue                    ║
║     └─ Not found? → Wrong -n flag or release name    ║
║                                                      ║
║  ② helm status RELEASE -n NAMESPACE                 ║
║     ├─ deployed → OK, check values                  ║
║     ├─ failed → Read error, check history           ║
║     └─ pending-* → Stuck hook or timeout             ║
║                                                      ║
║  ③ helm history RELEASE -n NAMESPACE                ║
║     └─ Find last good revision for rollback         ║
║                                                      ║
║  ④ helm get manifest RELEASE -n NAMESPACE           ║
║     └─ Inspect rendered YAML for errors             ║
║                                                      ║
║  ⑤ helm get values RELEASE -n NS --all              ║
║     └─ Did your values render correctly?            ║
║                                                      ║
║  ⑥ FIX & REDEPLOY                                   ║
║     └─ helm upgrade RELEASE CHART -n NS [fixes]     ║
║     └─ OR: helm rollback RELEASE REV -n NS         ║
╚══════════════════════════════════════════════════════╝
```

### Values Priority Ladder Card

```
╔══════════════════════════════════════════════════════╗
║          VALUES PRIORITY (LOWEST TO HIGHEST)         ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  ┌──────────────┐                                    ║
║  │ HIGHEST      │  --set key=value    (CLI flag)    ║
║  ├──────────────┤                                    ║
║  │              │  --set-string ...   (CLI flag)    ║
║  ├──────────────┤                                    ║
║  │              │  -f file2.yaml      (2nd values    ║
║  │              │                     file, wins)    ║
║  ├──────────────┤                                    ║
║  │              │  -f file1.yaml      (1st values    ║
║  │              │                     file)          ║
║  ├──────────────┤                                    ║
║  │              │  values.yaml        (chart's       ║
║  │              │                     defaults)      ║
║  ├──────────────┤                                    ║
║  │ LOWEST       │  Chart default      (hardcoded in  ║
║  │              │  values (Go)        template via   ║
║  │              │                     default func)  ║
║  └──────────────┘                                    ║
║                                                      ║
║  MEMORY AID:                                         ║
║  "CLI overrides files, files override defaults"      ║
╚══════════════════════════════════════════════════════╝
```

### Helm Directory Layout Card

```
╔══════════════════════════════════════════════════════╗
║          CHART STRUCTURE (HELM CREATE)               ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  mychart/                                            ║
║  ├── Chart.yaml             ← REQUIRED               ║
║  ├── values.yaml            ← REQUIRED               ║
║  ├── values.schema.json     ← Optional (validate)   ║
║  ├── charts/                ← Dependencies          ║
║  ├── templates/             ← REQUIRED               ║
║  │   ├── NOTES.txt          ← Release notes         ║
║  │   ├── _helpers.tpl       ← Template helpers      ║
║  │   ├── deployment.yaml                             ║
║  │   ├── service.yaml                                ║
║  │   ├── ingress.yaml                                ║
║  │   ├── hpa.yaml                                    ║
║  │   ├── serviceaccount.yaml                         ║
║  │   └── tests/             ← Helm test pods        ║
║  │       └── test-connection.yaml                    ║
║  ├── crds/                  ← Custom Resource Defs  ║
║  ├── .helmignore            ← Ignored when packaging ║
║  ├── Chart.lock             ← Dependency lock file  ║
║  ├── README.md                                       ║
║  └── CHANGELOG.md                                    ║
║                                                      ║
║  REQUIRED FIELDS in Chart.yaml:                      ║
║  apiVersion: v2                                       ║
║  name: mychart                                        ║
║  version: 0.1.0     (SemVer)                         ║
║                                                      ║
║  COMMON FIELDS in Chart.yaml:                         ║
║  appVersion: "1.16.0"                                 ║
║  description: A Helm chart for...                     ║
║  type: application | library                         ║
║  dependencies:                                        ║
║    - name: postgresql                                 ║
║      version: "15.5.17"                              ║
║      repository: https://charts.bitnami.com/bitnami  ║
║      condition: postgresql.enabled                    ║
╚══════════════════════════════════════════════════════╝
```
