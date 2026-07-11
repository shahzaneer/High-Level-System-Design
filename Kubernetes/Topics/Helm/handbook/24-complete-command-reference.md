# Chapter 24: Complete Command Reference (Alphabetical)

This chapter provides an exhaustive alphabetical reference of every Helm CLI command and subcommand. Each entry includes full syntax, every flag with its type/default/description, real-world examples with output, internal behavior notes, common mistakes, related commands, and CKA exam relevance.

---

## 24.1 Global Flags

These flags are available on every Helm command and are not listed individually in each entry below.

| Flag | Type | Default | Description |
|---|---|---|---|
| `--debug` | bool | `false` | Enable verbose debug output. Prints rendered templates during install/upgrade, full gRPC/logging traces. |
| `--kube-context` | string | `""` | Name of the kubeconfig context to use. Overrides the current context. |
| `--kubeconfig` | string | `""` | Path to the kubeconfig file. Defaults to `$KUBECONFIG` or `~/.kube/config`. |
| `--namespace` / `-n` | string | `"default"` | Namespace scope for this operation. Overrides the kubeconfig default namespace. |
| `--registry-config` | string | `"~/.config/helm/registry/config.json"` | Path to the registry credentials config file used for OCI registries. |
| `--repository-cache` | string | `"~/.cache/helm/repository"` | Path to the local cache of downloaded repository index files. |
| `--repository-config` | string | `"~/.config/helm/repositories.yaml"` | Path to the file containing Helm repository entries (URLs, names, credentials). |

**Note:** The global flag `--burst-limit` (int, default 100) was removed in Helm 3.14+. The Kubernetes client-side throttle limit is now configured via `--kube-api-burst` and `--kube-api-qps` on relevant commands.

---

## 24.2 Exit Codes Table

| Code | Meaning |
|---|---|
| `0` | Success — command completed without error. |
| `1` | General failure — the operation encountered an error (e.g., chart not found, invalid YAML, Kubernetes API error). |
| `2` | Misuse of shell builtins (bash-level, not Helm). Helm does not emit exit code 2. |
| `126` | Command found but not executable (e.g., permissions on the Helm binary). |
| `127` | Command not found — the Helm binary is not on `$PATH`. |
| `128` | Invalid exit argument — returned by Helm when the exit code is itself invalid. |
| `130` | Terminated by SIGINT (Ctrl+C). |
| `137` | Terminated by SIGKILL (OOM or external kill). |
| `143` | Terminated by SIGTERM (graceful shutdown). |

**Production Note:** In CI/CD pipelines, always check exit codes. A failed `helm install` returns exit code 1, but a partially completed install (resources created, hook failed) also returns exit code 1. Inspect `helm status` and `helm history` for the full picture.

---

## 24.3 Output Format Options

Several commands support `-o` / `--output` to control the formatting of tabular output.

| Flag Value | Description |
|---|---|
| `table` | Default. Prints results in a human-readable aligned table. |
| `json` | Prints results as a JSON array of objects. Useful for scripting with `jq`. |
| `yaml` | Prints results as a YAML list of objects. |

Commands supporting `-o`: `helm list`, `helm history`, `helm search hub`, `helm search repo`, `helm repo list`, `helm plugin list`, `helm get all`, `helm get hooks`, `helm get manifest`, `helm get metadata`, `helm get notes`, `helm get values`, `helm status`, `helm dependency list`, `helm env`.

---

## 24.4 Time Format Tokens

Used with `--time-format` on `helm list` and `helm history`.

| Token | Description | Example Output |
|---|---|---|
| `2006-01-02` | Go reference date format (full date) | `2026-07-11` |
| `2006-01-02T15:04:05Z07:00` | ISO 8601 with timezone | `2026-07-11T14:30:00+00:00` |
| `2006-01-02 15:04:05` | Space-separated date/time | `2026-07-11 14:30:00` |
| `Jan 2, 2006 15:04:05` | Human-readable US format | `Jul 11, 2026 14:30:00` |
| `2006/01/02` | Slash-separated date | `2026/07/11` |

**Note:** Go uses the reference time `Mon Jan 2 15:04:05 MST 2006` (which is `01/02 03:04:05PM '06 -0700`) to define format layouts. Any permutation of this reference date is valid. Common tokens: `2006` = year, `01` = month, `02` = day, `15` = hour (24h), `03` = hour (12h), `04` = minute, `05` = second, `PM` = AM/PM marker, `MST` = timezone name, `-0700` = timezone offset, `Z07:00` = ISO timezone.

---

## 24.5 Alphabetical Command Reference

---

### helm completion

**Syntax:**
```
helm completion <SHELL> [flags]
```

**Subcommands / Shells:**

| Shell | Syntax |
|---|---|
| bash | `helm completion bash` |
| zsh | `helm completion zsh` |
| fish | `helm completion fish` |
| powershell | `helm completion powershell` |

**Description:** Generates shell autocompletion scripts for the specified shell. The output is printed to stdout and should be sourced or saved to the appropriate shell configuration directory. After installation, pressing `<TAB>` after `helm` will suggest subcommands, flags, release names, and chart references.

**Flags:** None beyond global flags.

**Examples:**

1. Generate bash completion and source it:
```bash
$ helm completion bash > /etc/bash_completion.d/helm
$ source /etc/bash_completion.d/helm
# Pressing Tab after 'helm ' now shows subcommands
$ helm <TAB>
completion  dependency  env    get      help      history  install  lint  list
package     plugin      pull   push     registry  repo     rollback search
show        status      template  test   uninstall upgrade  verify   version
```

2. Generate zsh completion and install permanently:
```bash
$ helm completion zsh > "${fpath[1]}/_helm"
$ exec zsh
# Now type 'helm install <TAB>' to see release names and chart refs
```

**Output:** The generated script (Bourne shell script for bash, zsh script for zsh, fish function for fish, PowerShell script for powershell). No structured output format.

**Notes:**
- Completion scripts are auto-generated from the Cobra CLI framework. They are not hand-maintained.
- For bash, the script must be sourced, not executed as a binary.
- `helm completion bash` output is typically saved to `/etc/bash_completion.d/helm` on Linux or `/usr/local/etc/bash_completion.d/helm` on macOS (Homebrew).
- Completion is aware of the current kubeconfig context and can autocomplete release names from the cluster.

**Common Mistakes:**
- Running the script as a binary (`./helm_completion.sh`) instead of sourcing it (`source helm_completion.sh`).
- Saving zsh completion to a directory not in `$fpath`.
- Forgetting to restart the shell after installing completions.

**Related Commands:** `helm help`

**CKA Relevance:** Low — not tested on the exam. Useful for speed during hands-on practice.

---

### helm create

**Syntax:**
```
helm create NAME [flags]
```

**Description:** Creates a new chart directory with the given NAME. The directory is scaffolded with a standard layout including `Chart.yaml`, `values.yaml`, `charts/`, `templates/`, and helper files. This is the fastest way to bootstrap a new chart following best practices.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--dependency-update` / `-d` | bool | `false` | Run `helm dependency update` after chart creation. Populates `charts/` with any subcharts declared in the scaffolded `Chart.yaml`. The default scaffolded chart has no dependencies, so this is a no-op unless you modify `Chart.yaml` immediately. |
| `--starter` / `-p` | string | `""` | Path to a starter chart (a chart directory used as a template). The files from the starter are copied into the new chart, and any `{{ .Name }}` placeholders in file contents are replaced with the provided NAME. |
| `--force` | bool | `false` | Overwrite the target directory if it already exists. Without this flag, `helm create` refuses to overwrite an existing directory. |

**Examples:**

1. Create a new chart with the default scaffold:
```bash
$ helm create my-web-app
Creating my-web-app

$ tree my-web-app
my-web-app/
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

2. Create a chart using a starter template:
```bash
$ helm create my-microservice --starter ~/.helm/starters/microservice
Creating my-microservice
# The starter template files are copied and {{ .Name }} is replaced:
$ cat my-microservice/Chart.yaml
apiVersion: v2
name: my-microservice
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

**Output:** No stdout output except confirmation message. The chart directory is created on disk.

**Notes:**
- The default scaffold creates an Nginx deployment, a ClusterIP service, an optional Ingress, an optional HPA, and a ServiceAccount. These are templates meant to be customized.
- Starter charts must themselves be valid chart directories (contain `Chart.yaml`). The starter mechanism uses simple file copy with string replacement; it does not template with Go templates.
- The `--starter` flag looks for the starter path relative to the current working directory first, then in `$HELM_DATA_HOME/starters/`.
- Chart names must be valid DNS-1123 subdomain names: lowercase alphanumeric plus hyphens, starting and ending with an alphanumeric character.

**Common Mistakes:**
- Using uppercase letters in the chart name (invalid).
- Expecting `--dependency-update` to do something on a freshly scaffolded chart (no dependencies exist).
- Confusing `helm create` (creates a new chart skeleton) with `helm install` (deploys a chart).

**Related Commands:** `helm package`, `helm lint`, `helm template`, `helm install`

**CKA Relevance:** Medium — you may be asked to create a chart from scratch. Knowing the default scaffold saves time.

---

### helm dependency

**Syntax:**
```
helm dependency <subcommand> [CHART_PATH] [flags]
```

**Description:** Manages chart dependencies declared in `Chart.yaml` under the `dependencies` field. Dependencies are other charts that your chart requires, pulled from repositories or OCI registries. The subcommands handle the lifecycle of these dependencies.

**Positional Arguments:**
- `CHART_PATH`: Path to the chart directory. Defaults to the current directory (`.`).

---

#### helm dependency build

**Syntax:**
```
helm dependency build CHART_PATH [flags]
```

**Description:** Rebuilds the `charts/` directory from the `Chart.lock` file. Unlike `helm dependency update`, this does NOT fetch new versions from repositories — it uses the exact versions recorded in `Chart.lock`. Use this in CI/CD to ensure reproducible builds.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--keyring` | string | `"~/.gnupg/pubring.gpg"` | Path to the GPG keyring used for chart signature verification. |
| `--verify` | bool | `false` | Verify the integrity and provenance of downloaded charts using the GPG keyring before unpacking. |
| `--skip-refresh` | bool | `false` | Do not refresh the local repository cache. By default, `build` refreshes the cache before resolving dependencies. Setting this flag may use stale index data. |

**Examples:**

1. Build dependencies from the lock file in CI:
```bash
$ helm dependency build my-chart/
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Saving 2 charts
Downloading common from repo oci://registry-1.docker.io/bitnamicharts
Downloading postgresql from repo oci://registry-1.docker.io/bitnamicharts
Deleting outdated charts
```

2. Build with signature verification:
```bash
$ helm dependency build my-chart/ --verify --keyring /etc/helm/keyring.gpg
Hang tight while we grab the latest from your chart repositories...
Verifying common-2.5.0.tgz
Successfully verified common-2.5.0.tgz
Saving 2 charts
```

**Output:** Progress messages for each dependency download. On failure, exit code 1 with an error description.

**Notes:**
- `helm dependency build` requires `Chart.lock` to exist. If it does not exist, use `helm dependency update` first.
- The `--verify` flag requires that the chart was signed with `helm package --sign` and the signer's public key is in the keyring.
- Internally, `build` reads `Chart.lock`, downloads each dependency's `.tgz` into `charts/`, and deletes any charts in `charts/` not listed in the lock file.

**Common Mistakes:**
- Running `build` without a `Chart.lock` file — it will fail with "no lock file found."
- Expecting `build` to pull newer versions — it always uses exact versions from the lock file.

**Related Commands:** `helm dependency update`, `helm dependency list`, `helm install --dependency-update`

**CKA Relevance:** Low — understanding the difference between `build` and `update` is helpful but unlikely to be explicitly tested.

---

#### helm dependency list

**Syntax:**
```
helm dependency list CHART_PATH [flags]
```

**Description:** Lists all dependencies declared in `Chart.yaml` and shows their current status: whether they are present in `charts/`, whether their version matches the declared version, and which repository they come from.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--max-col-width` | uint | `60` | Maximum column width for the table output. Columns wider than this are truncated with an ellipsis. |
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |

**Examples:**

1. List dependencies in table format:
```bash
$ helm dependency list my-chart/
NAME            VERSION REPOSITORY                                      STATUS
common          2.5.0   oci://registry-1.docker.io/bitnamicharts        ok
postgresql      12.1.0  oci://registry-1.docker.io/bitnamicharts        ok
redis           18.0.0  https://charts.bitnami.com/bitnami              missing
```

2. List dependencies in JSON format for scripting:
```bash
$ helm dependency list my-chart/ -o json
[
  {
    "name": "common",
    "version": "2.5.0",
    "repository": "oci://registry-1.docker.io/bitnamicharts",
    "status": "ok"
  },
  {
    "name": "postgresql",
    "version": "12.1.0",
    "repository": "oci://registry-1.docker.io/bitnamicharts",
    "status": "ok"
  },
  {
    "name": "redis",
    "version": "18.0.0",
    "repository": "https://charts.bitnami.com/bitnami",
    "status": "missing"
  }
]
```

**Output:** A table (or JSON/YAML) with columns: NAME, VERSION, REPOSITORY, STATUS. STATUS values: `ok` (downloaded and matches version), `missing` (not in `charts/`), `unpacked` (present but no `.tgz`), `wrong version` (present but version differs).

**Notes:**
- This command reads only `Chart.yaml` and the `charts/` directory. It does not contact any repository.
- The STATUS column does not indicate whether the dependency is out of date on the remote — only whether it matches the local declaration.
- Use `helm dependency update` to resolve `missing` or `wrong version` entries.

**Common Mistakes:**
- Confusing `helm dependency list` with `helm list` (the latter lists releases, not chart dependencies).
- Expecting `dependency list` to show available upstream versions — it only shows declared vs. installed local state.

**Related Commands:** `helm dependency update`, `helm dependency build`

**CKA Relevance:** Low — useful for debugging dependency issues but not a primary exam topic.

---

#### helm dependency update

**Syntax:**
```
helm dependency update CHART_PATH [flags]
```

**Description:** Downloads all declared dependencies into the `charts/` directory and creates or updates `Chart.lock`. This command fetches repository index files, resolves version constraints (e.g., `^1.2.3`), and pulls the latest matching chart `.tgz` archives. Use this to populate dependencies for development and to regenerate the lock file.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--keyring` | string | `"~/.gnupg/pubring.gpg"` | Path to the GPG keyring for chart verification. |
| `--verify` | bool | `false` | Verify downloaded chart signatures against the keyring. |
| `--skip-refresh` | bool | `false` | Skip refreshing the local repository cache. If the cache is stale, version resolution may fail or select outdated versions. |

**Examples:**

1. Update dependencies for a chart:
```bash
$ helm dependency update my-chart/
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. Happy Helming!
Saving 3 charts
Downloading postgresql from repo https://charts.bitnami.com/bitnami
Downloading redis from repo https://charts.bitnami.com/bitnami
Downloading common from repo oci://registry-1.docker.io/bitnamicharts
Deleting outdated charts
```

2. Update without refreshing local indexes (offline-friendly):
```bash
$ helm dependency update --skip-refresh
Saving 3 charts
Downloading postgresql from repo https://charts.bitnami.com/bitnami
Deleting outdated charts
```

**Output:** Progress messages showing each repository index refresh and chart download. The generated `Chart.lock` file is not printed.

**Notes:**
- This command updates `Chart.lock` with the exact versions resolved. The lock file should be committed to version control for reproducible builds.
- Version constraints in `Chart.yaml` follow SemVer range syntax: `>1.2.3`, `>=1.2.3`, `<2.0.0`, `^1.2.3`, `~1.2.3`, `1.2.x`, `*`, or an exact version `1.2.3`.
- When using OCI-based dependencies, the chart is pulled using the OCI distribution protocol.
- Dependencies with a `repository` of `file://../path/to/chart` are resolved locally and copied into `charts/`.

**Common Mistakes:**
- Forgetting to commit `Chart.lock` — this breaks reproducibility in CI.
- Using `*` as the version constraint — this pulls the absolute latest, which may introduce breaking changes.
- Running `update` without network access and without `--skip-refresh` — it will hang on the index refresh.

**Related Commands:** `helm dependency build`, `helm dependency list`, `helm install --dependency-update`

**CKA Relevance:** Medium — the CKA exam expects you to know how to manage chart dependencies.

---

### helm env

**Syntax:**
```
helm env [flags]
```

**Description:** Prints all Helm-related environment variables and their current values. This is useful for debugging: it shows which directories Helm uses for configuration, cache, plugins, and data. The output includes both user-configurable variables (like `HELM_REPOSITORY_CONFIG`) and informational variables (like `HELM_BIN`).

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |

**Examples:**

1. Show Helm environment in table format:
```bash
$ helm env
HELM_BIN="helm"
HELM_CACHE_HOME="/Users/username/Library/Caches/helm"
HELM_CONFIG_HOME="/Users/username/Library/Preferences/helm"
HELM_DATA_HOME="/Users/username/Library/helm"
HELM_DEBUG="false"
HELM_KUBEAPISERVER=""
HELM_KUBECONTEXT=""
HELM_MAX_HISTORY="10"
HELM_NAMESPACE="default"
HELM_PLUGINS="/Users/username/Library/helm/plugins"
HELM_REGISTRY_CONFIG="/Users/username/Library/Preferences/helm/registry/config.json"
HELM_REPOSITORY_CACHE="/Users/username/Library/Caches/helm/repository"
HELM_REPOSITORY_CONFIG="/Users/username/Library/Preferences/helm/repositories.yaml"
```

2. Get a single value using JSON output and jq:
```bash
$ helm env -o json | jq -r '.[] | select(.name=="HELM_REPOSITORY_CONFIG") | .value'
/Users/username/Library/Preferences/helm/repositories.yaml
```

**Output:** A list of key-value pairs showing Helm environment variable names and their effective values.

**Notes:**
- Environment variables prefixed with `HELM_` can be set to override the defaults shown by this command.
- The output reflects the **effective** values, considering both environment variables and defaults.
- `HELM_CACHE_HOME`, `HELM_CONFIG_HOME`, and `HELM_DATA_HOME` follow the XDG Base Directory Specification on Linux and macOS conventions.

**Common Mistakes:**
- Confusing `helm env` with shell `env` — the former only shows Helm variables, not all environment variables.
- Expecting `helm env` to show Kubernetes-related variables like `KUBECONFIG` — only variables with the `HELM_` prefix are shown.

**Related Commands:** `helm version`, `helm plugin list`

**CKA Relevance:** Medium — understanding Helm's environment configuration helps debug namespace issues and repository resolution problems.

---

### helm get

**Syntax:**
```
helm get <subcommand> RELEASE_NAME [flags]
```

**Description:** Retrieves information about a deployed release. The subcommands extract different facets of the release: its Kubernetes manifests, user-supplied values, chart metadata, hooks, release notes, or all of the above combined.

---

#### helm get all

**Syntax:**
```
helm get all RELEASE_NAME [flags]
```

**Description:** Retrieves all available information about a release: hooks, manifest, metadata, notes, and values. This is a convenience command that aggregates the output of `helm get hooks`, `helm get manifest`, `helm get metadata`, `helm get notes`, and `helm get values` into a single output.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. (Note: `table` mode prints YAML blocks for each section.) |
| `--revision` | int | `0` | The revision number to retrieve. `0` means the latest revision. |
| `--template` | string | `""` | A Go template string to format the output. The template has access to the entire release object. |

**Examples:**

1. Get all information for a release:
```bash
$ helm get all my-release
METADATA:
NAME: my-release
LAST DEPLOYED: Sat Jul 11 14:30:00 2026
NAMESPACE: default
STATUS: deployed
REVISION: 3
TEST SUITE:     None
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=nginx,app.kubernetes.io/instance=my-release" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:80
USER-SUPPLIED VALUES:
replicaCount: 3
image:
  tag: "1.20.0"
COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
...
HOOKS:
---
# Source: nginx/templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "my-release-nginx-test-connection"
  annotations:
    "helm.sh/hook": test
...
MANIFEST:
---
# Source: nginx/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-release-nginx
...
```

2. Filter the manifest section using `--template`:
```bash
$ helm get all my-release --template '{{ .Release.Manifest }}' | kubectl get -f - -o name
serviceaccount/my-release-nginx
service/my-release-nginx
deployment.apps/my-release-nginx
```

**Output:** Combined YAML blocks for each section (metadata, notes, user-supplied values, computed values, hooks, manifest). When using `-o json`, each section is a key in a JSON object.

**Notes:**
- "USER-SUPPLIED VALUES" are the values provided at install/upgrade time (via `--set`, `--values`, etc.). "COMPUTED VALUES" are the fully merged values after applying defaults from `values.yaml`.
- The `--template` flag gives access to the entire `Release` object, including `.Release`, `.Chart`, `.Values`, `.Hooks`, and `.Manifest`.

**Common Mistakes:**
- Using `helm get all` for automation — prefer `helm get manifest` or `helm get values` with `-o json` for machine-readable output.
- Confusing user-supplied values with computed values in scripting.

**Related Commands:** `helm get manifest`, `helm get values`, `helm get hooks`, `helm get metadata`, `helm get notes`

**CKA Relevance:** Medium — useful for verifying release state during troubleshooting tasks.

---

#### helm get hooks

**Syntax:**
```
helm get hooks RELEASE_NAME [flags]
```

**Description:** Retrieves the hook resources for a release. Hooks are Kubernetes resources annotated with `helm.sh/hook` that run at specific lifecycle events (pre-install, post-install, pre-upgrade, post-upgrade, pre-delete, post-delete, pre-rollback, post-rollback, test).

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--revision` | int | `0` | The revision number to retrieve. `0` means the latest revision. |

**Examples:**

1. Get hooks for a release:
```bash
$ helm get hooks my-release
---
# Source: my-chart/templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "my-release-my-chart-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['my-release-my-chart:80']
  restartPolicy: Never
```

2. Get hooks as JSON to check hook annotations programmatically:
```bash
$ helm get hooks my-release -o json | jq '.'
```

**Output:** Rendered YAML manifest for all hook resources, or a JSON array of hook resource objects when using `-o json`.

**Notes:**
- Hook resources are NOT part of the main manifest. They are stored separately in the release record.
- Even if a hook has already executed, its definition is still retrievable via this command.
- Hook weights and deletion policies are visible in the annotations of each hook resource.

**Common Mistakes:**
- Expecting this command to show only active/running hooks — it shows all hook definitions regardless of execution state.
- Looking for hooks in `helm get manifest` — hooks are filtered out and only appear in `helm get hooks`.

**Related Commands:** `helm get all`, `helm get manifest`, `helm test`

**CKA Relevance:** Low — hooks are an advanced Helm feature, testing focus is minimal.

---

#### helm get manifest

**Syntax:**
```
helm get manifest RELEASE_NAME [flags]
```

**Description:** Retrieves the generated Kubernetes manifest (the full set of rendered templates) for a release. This is the exact YAML that Helm submitted to the Kubernetes API during install or upgrade. It excludes hook resources.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--revision` | int | `0` | The revision number to retrieve. `0` means the latest revision. |

**Examples:**

1. Get the manifest and pipe to kubectl for inspection:
```bash
$ helm get manifest my-release | kubectl get -f - -o wide
NAME                                   READY   STATUS    RESTARTS   AGE
my-release-nginx-7d4f8b9c6-abc12       1/1     Running   0          5m
my-release-nginx-7d4f8b9c6-def34       1/1     Running   0          5m
my-release-nginx-7d4f8b9c6-ghi56       1/1     Running   0          5m
```

2. Get manifest for a specific revision:
```bash
$ helm get manifest my-release --revision 2 | grep 'image:'
  image: "nginx:1.19.0"
```

**Output:** Multi-document YAML (separated by `---`) containing all Kubernetes resources for the release. When using `-o json`, returns a JSON array of resource objects.

**Notes:**
- This is the most commonly used `helm get` subcommand for debugging: it shows exactly what was deployed.
- The output includes the `# Source:` comment lines indicating which template file generated each resource.
- Hooks are not included. Use `helm get hooks` to see those.
- The manifest is stored as-is in the release Secret/ConfigMap. It is not re-fetched from the Kubernetes API.

**Common Mistakes:**
- Piping `helm get manifest` directly into `kubectl apply` — this bypasses Helm's release tracking and can cause state drift.
- Expecting live cluster state — `helm get manifest` shows what Helm **intended** to deploy, not what currently exists. Use `kubectl get` for live state.

**Related Commands:** `helm get all`, `helm get hooks`, `helm template`, `helm status`

**CKA Relevance:** High — frequently used in CKA exam scenarios to verify what a Helm release deployed.

---

#### helm get metadata

**Syntax:**
```
helm get metadata RELEASE_NAME [flags]
```

**Description:** Retrieves the release metadata: name, namespace, chart, status, deployed timestamp, revision number, and application version. Does not show manifests, values, or hooks.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--revision` | int | `0` | The revision number to retrieve. `0` means the latest revision. |

**Examples:**

1. Get metadata for a release:
```bash
$ helm get metadata my-release
NAME: my-release
LAST DEPLOYED: Sat Jul 11 14:30:00 2026
NAMESPACE: default
STATUS: deployed
REVISION: 3
TEST SUITE:     None
```

2. Get metadata in JSON for scripting:
```bash
$ helm get metadata my-release -o json | jq '{name: .name, revision: .version}'
{
  "name": "my-release",
  "revision": 3
}
```

**Output:** Key-value pairs showing the release's identifying information. The `TEST SUITE` field shows the status of the last `helm test` run (`None` if never tested).

**Notes:**
- The output format for `-o json` uses `.name`, `.version` (revision number), `.namespace`, `.status`, `.chart`, and `.deployedAt` fields.
- `LAST DEPLOYED` reflects the timestamp of the most recent install or upgrade, not necessarily the current cluster state.

**Common Mistakes:**
- Using `helm get metadata` to check if pods are running — use `helm status` or `kubectl get pods` for that.

**Related Commands:** `helm get all`, `helm status`, `helm history`

**CKA Relevance:** Medium — quick way to verify release identity during exam tasks.

---

#### helm get notes

**Syntax:**
```
helm get notes RELEASE_NAME [flags]
```

**Description:** Retrieves the rendered `NOTES.txt` for a release. The `NOTES.txt` template is typically used to display post-installation instructions: how to access the application, next steps, or credentials.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--revision` | int | `0` | The revision number to retrieve. `0` means the latest revision. |

**Examples:**

1. Get notes for a release:
```bash
$ helm get notes my-release
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=nginx,app.kubernetes.io/instance=my-release" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:80
```

2. Pipe notes into a script:
```bash
$ helm get notes my-release -o json | jq -r '.notes' | bash
```

**Output:** The plain text content of the rendered `NOTES.txt` file.

**Notes:**
- `NOTES.txt` is a Go template file located at `templates/NOTES.txt`. It has access to the same built-in objects as any other template.
- If the chart has no `NOTES.txt`, the output is empty.
- Notes are rendered once at install/upgrade time and stored in the release record.

**Common Mistakes:**
- Expecting notes to contain live connection information — notes are static text rendered at deployment time.

**Related Commands:** `helm get all`, `helm get manifest`, `helm status`

**CKA Relevance:** Low — situational, but useful for verifying post-install instructions.

---

#### helm get values

**Syntax:**
```
helm get values RELEASE_NAME [flags]
```

**Description:** Retrieves the values that were supplied to the release at install or upgrade time. This includes values provided via `--set`, `--values`, `--set-file`, `--set-string`, and `--set-json`. It does NOT include default values from `values.yaml` — use `--all` to include those.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--all` / `-a` | bool | `false` | Include all computed values (user-supplied + chart defaults) instead of only user-supplied values. |
| `--output` / `-o` | string | `"yaml"` | Output format: `table`, `json`, or `yaml`. Note: default is `yaml`, not `table`. |
| `--revision` | int | `0` | The revision number to retrieve. `0` means the latest revision. |

**Examples:**

1. Get only user-supplied values:
```bash
$ helm get values my-release
USER-SUPPLIED VALUES:
replicaCount: 5
image:
  tag: "1.20.0"
  pullPolicy: Always
```

2. Get all computed values including defaults:
```bash
$ helm get values my-release --all
COMPUTED VALUES:
affinity: {}
autoscaling:
  enabled: false
  maxReplicas: 100
  minReplicas: 1
  targetCPUUtilizationPercentage: 80
fullnameOverride: ""
image:
  pullPolicy: Always
  repository: nginx
  tag: "1.20.0"
imagePullSecrets: []
ingress:
  annotations: {}
  enabled: false
  ...
replicaCount: 5
...
```

**Output:** YAML-formatted values. By default, only "USER-SUPPLIED VALUES". With `--all`, "COMPUTED VALUES" (the fully merged values with defaults).

**Notes:**
- The default output format for `helm get values` is YAML (not table), unlike other `helm get` subcommands.
- "USER-SUPPLIED VALUES" captures everything passed via `--set`, `--values`, `--set-file`, `--set-string`, and `--set-json`.
- Values are stored in the release Secret/ConfigMap. Sensitive values are stored as-is — Helm does not encrypt release secrets by default.
- The `--revision` flag can be used to inspect values from a prior revision, which is useful for understanding what changed during a failed upgrade.

**Common Mistakes:**
- Forgetting the `--all` flag and being confused about why chart defaults are not shown.
- Assuming `helm get values` shows the current live cluster configuration — it shows what was submitted, not what currently exists.
- Storing plaintext secrets in values and then retrieving them with this command — consider using `helm-secrets` plugin.

**Related Commands:** `helm get all`, `helm upgrade --reset-values`, `helm install --set`

**CKA Relevance:** High — very likely to appear in exam scenarios. You need to know how to inspect and verify release values.

---

### helm help

**Syntax:**
```
helm help [COMMAND] [flags]
```

**Description:** Displays help information for any Helm command or subcommand. Running `helm help` without arguments shows the top-level usage and lists all available commands.

**Flags:** None beyond global flags.

**Examples:**

1. Show top-level help:
```bash
$ helm help
The Kubernetes package manager

Common actions for Helm:

- helm search:    search for charts
- helm pull:      download a chart to your local directory to view
- helm install:   upload the chart to Kubernetes
- helm list:      list releases of charts

Environment variables:

| Name                               | Description                                     |
|------------------------------------|-------------------------------------------------|
| $HELM_CACHE_HOME                   | set an alternative location for cached files.   |
| $HELM_CONFIG_HOME                  | set an alternative location for Helm config.    |
| $HELM_DATA_HOME                    | set an alternative location for Helm data.      |
...

Usage:
  helm [command]

Available Commands:
  completion  Generate autocompletion scripts for the specified shell
  create      create a new chart with the given name
  dependency  manage a chart's dependencies
  ...
```

2. Show help for a specific command:
```bash
$ helm help install
This command installs a chart archive.

Usage:
  helm install [NAME] [CHART] [flags]

Flags:
      --create-namespace                create the release namespace if not present
      --dependency-update               update dependencies if they are missing before installing
      --description string              add a custom description
  ...
```

**Output:** Help text including usage, description, environment variables (top-level only), subcommands, and flags.

**Notes:**
- `helm help` is equivalent to `helm --help`.
- The help output is generated directly from the Cobra CLI framework declarations in the Helm source code. It is always up to date with the installed version.
- Common environment variables are listed in the top-level help output.

**Common Mistakes:** None — this is a reference command with no side effects.

**Related Commands:** `helm version`, `helm env`

**CKA Relevance:** High — the CKA exam terminal has no internet access. `helm help` is your primary reference for command syntax.

---

### helm history

**Syntax:**
```
helm history RELEASE_NAME [flags]
```

**Description:** Displays the revision history for a release, showing each revision number, the date it was deployed, its description, its status, the chart version used, and the app version. This is essential for understanding the deployment timeline and identifying when and what changed.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--max` | int | `256` | Maximum number of revisions to display. `0` means no limit. |
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--time-format` | string | `"2006-01-02 15:04:05.999999999 -0700 MST"` | Go time format string for the `UPDATED` column. See Section 24.4 for tokens. |

**Examples:**

1. Show revision history in table format:
```bash
$ helm history my-release
REVISION  UPDATED                   STATUS        CHART           APP VERSION  DESCRIPTION
1         Sat Jul 11 10:00:00 2026  superseded    nginx-15.0.0    1.25.0       Install complete
2         Sat Jul 11 12:00:00 2026  superseded    nginx-15.1.0    1.26.0       Upgrade complete
3         Sat Jul 11 14:30:00 2026  deployed      nginx-15.2.0    1.27.0       Upgrade complete
```

2. Get history as JSON for automated auditing:
```bash
$ helm history my-release -o json | jq '.[] | {rev: .revision, status: .status, chart: .chart}'
{
  "rev": 1,
  "status": "superseded",
  "chart": "nginx-15.0.0"
}
{
  "rev": 2,
  "status": "superseded",
  "chart": "nginx-15.1.0"
}
{
  "rev": 3,
  "status": "deployed",
  "chart": "nginx-15.2.0"
}
```

**Output:** A table (or JSON/YAML) with columns: REVISION, UPDATED, STATUS, CHART, APP VERSION, DESCRIPTION.

**Notes:**
- The maximum number of stored revisions is controlled by `--history-max` set at `helm install` / `helm upgrade` time. The default is `10` (configurable via `HELM_MAX_HISTORY`). Older revisions beyond this limit are automatically purged.
- Revisions with a `failed` status still consume a revision number. A release with 3 failed upgrades and 1 successful install has 4 revisions.
- The `DESCRIPTION` column shows the message provided via `--description` at install/upgrade time.
- Revisions are stored as Kubernetes Secrets (default) or ConfigMaps in the release namespace. The naming convention is `sh.helm.release.v1.<release-name>.v<revision>`.

**Common Mistakes:**
- Confusing `helm history` with `helm list` — history shows all revisions for one release; list shows all releases (latest revision only).
- Exceeding `--history-max` and losing the ability to rollback to older revisions.
- Not providing a `--description` during critical upgrades, making it hard to identify what each revision contains.

**Related Commands:** `helm rollback`, `helm list`, `helm get metadata`

**CKA Relevance:** High — rollback scenarios require understanding revision history. You may be asked to identify which revision to roll back to.

---

### helm install

**Syntax:**
```
helm install [NAME] [CHART] [flags]
```

**Description:** Installs a chart as a new Helm release in the target Kubernetes cluster and namespace. This is the primary deployment command. The NAME argument is optional — if omitted, `--generate-name` or `--name-template` must be provided. The CHART can be a chart reference (e.g., `bitnami/nginx`), a path to an unpacked chart directory, a path to a `.tgz` archive, or a fully qualified URL (HTTP or OCI).

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--atomic` | bool | `false` | If true, automatically roll back the release on failure. The release is deleted and purged. Equivalent to `--wait && (on failure) helm rollback`. |
| `--ca-file` | string | `""` | Path to a CA certificate bundle for verifying the chart repository's TLS certificate. |
| `--cert-file` | string | `""` | Path to a client certificate for TLS authentication to the chart repository. |
| `--cleanup-on-fail` | bool | `false` | Delete newly created resources if the install fails (but keep the release record for debugging). |
| `--create-namespace` | bool | `false` | Create the release namespace if it does not already exist. |
| `--dependency-update` | bool | `false` | Run `helm dependency update` before installing. Pulls subcharts defined in `Chart.yaml`. |
| `--description` | string | `""` | Human-readable description stored in release metadata. Visible in `helm history`. |
| `--devel` | bool | `false` | Use development (pre-release) versions of charts. By default, only stable versions are considered. |
| `--disable-openapi-validation` | bool | `false` | Skip OpenAPI schema validation of rendered templates against the target Kubernetes API. |
| `--dry-run` | bool | `false` | Simulate the installation. Templates are rendered and printed but NOT submitted to the cluster. |
| `--dry-run-option` | string | `"none"` | Fine-grained dry-run mode. Values: `none` (render locally only), `client` (same as `none`), `server` (submit to server-side validation but do not persist). |
| `--enable-dns` | bool | `false` | Enable DNS lookups when rendering templates (rare, requires network access during template execution). |
| `--force` | bool | `false` | Force resource updates through a delete-and-recreate strategy. Use with caution. |
| `--generate-name` | bool | `false` | Generate a release name by appending a random suffix to the chart name (e.g., `nginx-ingress-1691427345`). Mutually exclusive with a positional NAME argument and `--name-template`. |
| `--history-max` | int | `10` | Maximum number of revisions to retain. Older revisions are automatically purged. Set to `0` for unlimited (not recommended in production). |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS certificate verification for chart downloads. **Security risk** — use only in trusted environments. |
| `--key-file` | string | `""` | Path to a client private key file for TLS authentication. |
| `--keyring` | string | `"~/.gnupg/pubring.gpg"` | Path to the GPG keyring for chart provenance verification. |
| `--labels` | stringToString | `[]` | Labels to add to the release metadata (not Kubernetes resource labels). Format: `key=value`. Can be specified multiple times. |
| `--name-template` | string | `""` | Go template string for generating the release name. Example: `"{{ .Release.Name }}-{{ .Release.Namespace }}"`. Mutually exclusive with a positional NAME and `--generate-name`. |
| `--no-hooks` | bool | `false` | Disable hook execution during install. Hook resources are still rendered and submitted, but their hook annotations are ignored. |
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--pass-credentials` | bool | `false` | Forward HTTP basic authentication credentials to all domains. |
| `--password` | string | `""` | Password for chart repository authentication. |
| `--plain-http` | bool | `false` | Use plain HTTP connections (not HTTPS) for chart repositories. |
| `--post-renderer` | string | `""` | Path to an executable post-renderer script. The rendered manifests are piped through this script before submission. |
| `--post-renderer-args` | stringSlice | `[]` | Arguments to pass to the post-renderer. Can be specified multiple times. |
| `--render-subchart-notes` | bool | `false` | Render NOTES.txt from subcharts (dependencies) in addition to the parent chart. |
| `--replace` | bool | `false` | Reuse the given name even if a release with that name already exists. The existing release is deleted and replaced. **This is destructive.** |
| `--repo` | string | `""` | Chart repository URL where to locate the chart. If set, CHART should be just the chart name. |
| `--set` | stringArray | `[]` | Set values on the command line. Can specify multiple times or use comma-separated key=value pairs. Example: `--set image.tag=1.20.0,replicaCount=3`. |
| `--set-file` | stringArray | `[]` | Set values from files. Format: `key=/path/to/file`. The file content becomes the value string. |
| `--set-json` | stringArray | `[]` | Set values from JSON strings. Example: `--set-json 'resources={"limits":{"cpu":"500m"}}'`. |
| `--set-literal` | stringArray | `[]` | Set values as literal strings, preventing the template engine from interpreting `${}` expressions. |
| `--set-string` | stringArray | `[]` | Set values as strings (forces string type). Example: `--set-string replicaCount="3"`. |
| `--skip-crds` | bool | `false` | Skip installing CRDs defined in the chart's `crds/` directory. CRDs are only installed during `helm install`, never on `helm upgrade`. |
| `--timeout` | duration | `5m0s` | Maximum time to wait for Kubernetes operations. Format: Go duration string (`5m`, `1h`, `300s`). |
| `--username` | string | `""` | Username for chart repository authentication. |
| `--values` / `-f` | stringArray | `[]` | Specify YAML values files. Can be specified multiple times (merged left to right). Example: `-f values.yaml -f values-prod.yaml`. |
| `--verify` | bool | `false` | Verify the chart's provenance signature before installing. Requires a GPG keyring. |
| `--version` | string | `""` | Specify a version constraint for the chart version. Uses SemVer ranges. Example: `--version ">=1.0.0 <2.0.0"`. |
| `--wait` | bool | `false` | Wait for all resources to reach a ready state before marking the release as deployed. |
| `--wait-for-jobs` | bool | `false` | Also wait for Jobs to complete successfully when `--wait` is set. Jobs must exit with code 0. |

**Examples:**

1. Install a chart from a repository with custom values:
```bash
$ helm install my-nginx bitnami/nginx \
    --set replicaCount=3 \
    --set service.type=ClusterIP \
    --namespace web \
    --create-namespace \
    --wait

NAME: my-nginx
LAST DEPLOYED: Sat Jul 11 14:30:00 2026
NAMESPACE: web
STATUS: deployed
REVISION: 1
TEST SUITE:     None
NOTES:
CHART NAME: nginx
CHART VERSION: 15.0.0
APP VERSION: 1.25.0
...
```

2. Dry-run install to validate templates:
```bash
$ helm install my-nginx bitnami/nginx \
    --dry-run \
    --set image.tag=latest \
    --debug 2>&1 | head -20

NAME: my-nginx
LAST DEPLOYED: Sat Jul 11 14:30:00 2026
NAMESPACE: default
STATUS: pending-install
REVISION: 1
USER-SUPPLIED VALUES:
image:
  tag: latest
COMPUTED VALUES:
...
---
# Source: nginx/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
...
```

**Output:** Release summary including NAME, LAST DEPLOYED, NAMESPACE, STATUS, REVISION, and rendered NOTES.txt. On failure, an error message with exit code 1.

**Notes:**
- `helm install` creates a release Secret named `sh.helm.release.v1.<release-name>.v1` (revision 1) in the target namespace.
- The `--atomic` flag is equivalent to `--wait` plus automatic rollback on failure. Without `--atomic`, a failed install leaves the release in a `failed` state.
- `--dry-run` renders templates locally and applies minimal validation. `--dry-run-option=server` sends the manifest to the Kubernetes API server for full validation (requires a running cluster).
- CRDs in `crds/` are installed only during `helm install`, never on `helm upgrade`. Use `--skip-crds` to suppress this behavior.
- Post-renderers (via `--post-renderer`) receive the full multi-document YAML manifest on stdin and must output the modified manifest on stdout.

**Common Mistakes:**
- Using `--replace` casually — it deletes the existing release and its resources before installing. Prefer `helm upgrade` for updating.
- Forgetting `--create-namespace` and getting a "namespace not found" error.
- Using `--wait` without `--timeout` on charts with long initialization times.
- Using `--force` unnecessarily — it deletes and recreates resources, which can cause downtime.

**Related Commands:** `helm upgrade`, `helm uninstall`, `helm template`, `helm lint`, `helm rollback`

**CKA Relevance:** High — core Helm command. You will almost certainly need to install a chart on the exam.

---

### helm lint

**Syntax:**
```
helm lint PATH [flags]
```

**Description:** Validates a chart by checking its structure, metadata, and rendered templates against best practices and Kubernetes API schemas. Detects issues like missing required fields, invalid YAML, template rendering errors, bad chart metadata, and deprecated API versions. This is a static analysis tool — it does not require a running Kubernetes cluster.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--debug` | bool | `false` | Enable verbose debug output (rendered template content on errors). |
| `--namespace` / `-n` | string | `"default"` | Namespace to use when rendering templates (affects namespace-scoped API lookups). |
| `--set` | stringArray | `[]` | Set values for template rendering (same syntax as `helm install --set`). |
| `--set-file` | stringArray | `[]` | Set values from files. |
| `--set-json` | stringArray | `[]` | Set values from JSON. |
| `--set-literal` | stringArray | `[]` | Set literal string values. |
| `--set-string` | stringArray | `[]` | Set string values. |
| `--strict` | bool | `false` | Treat warnings as errors. If any lint rule produces a warning, return exit code 1. |
| `--values` / `-f` | stringArray | `[]` | Specify values files for template rendering. |
| `--with-subcharts` | bool | `false` | Lint subcharts (dependencies) in `charts/` as well as the parent chart. |

**Examples:**

1. Lint a chart with strict mode:
```bash
$ helm lint my-chart/ --strict
==> Linting my-chart/
[INFO] Chart.yaml: icon is recommended
[WARNING] templates/deployment.yaml: containerImage does not use a tagged version (image: "nginx:latest")
Error: 1 warning(s) found
```

2. Lint a chart with custom values:
```bash
$ helm lint my-chart/ -f values-prod.yaml --set replicaCount=5
==> Linting my-chart/
[INFO] Chart.yaml: icon is recommended
1 chart(s) linted, 0 chart(s) failed
```

**Output:** A report for each chart file, showing `[INFO]`, `[WARNING]`, or `[ERROR]` messages. The final summary line shows how many charts were linted and how many failed.

**Notes:**
- `helm lint` does not require cluster access. It validates templates against the Kubernetes API schema embedded in the Helm binary.
- The lint rules include: chart name validity (`Chart.yaml`), version format, deprecated API versions, missing required fields, invalid YAML syntax, template engine errors, icon URL presence (info only).
- Use `--with-subcharts` to validate dependency charts. Without this flag, subcharts are skipped.
- Linting does not catch all possible errors — it cannot detect logical errors in template conditionals.
- The icon warning (`[INFO] Chart.yaml: icon is recommended`) is just a recommendation and does not affect lint pass/fail.

**Common Mistakes:**
- Forgetting to pass required values via `--set` or `-f`, causing template rendering failures during lint.
- Treating lint success as a guarantee that the chart will deploy successfully — always test with `helm install --dry-run`.
- Not using `--with-subcharts` and missing errors in dependency charts.

**Related Commands:** `helm template`, `helm install --dry-run`, `helm package`

**CKA Relevance:** Medium — chart validation is part of the CKA workflow.

---

### helm list

**Syntax:**
```
helm list [flags]
```

**Description:** Lists all Helm releases in the current namespace (or all namespaces with `--all-namespaces`). Shows the release name, namespace, revision, last updated timestamp, status, chart, and app version. This is the primary command for discovering what is deployed in a cluster via Helm.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--all` / `-a` | bool | `false` | Show all releases regardless of status. By default, only `deployed` releases are shown. |
| `--all-namespaces` / `-A` | bool | `false` | List releases across ALL namespaces. |
| `--date-format` | string | `""` | Format string for the date in the `UPDATED` column. Same as `--time-format`. |
| `--deployed` | bool | `false` | Show only `deployed` releases (default filter). Can be combined with other state filters. |
| `--failed` | bool | `false` | Show only `failed` releases. |
| `--filter` | string | `""` | Regular expression to filter releases by name. Example: `--filter 'nginx\|redis'`. |
| `--max` | int | `256` | Maximum number of releases to return. |
| `--no-headers` | bool | `false` | Suppress the column headers in table output. Useful for scripting. |
| `--offset` | int | `0` | Number of releases to skip (pagination offset). |
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--pending` | bool | `false` | Show only `pending-*` releases (pending-install, pending-upgrade, pending-rollback). |
| `--reverse` / `-r` | bool | `false` | Reverse the sort order. |
| `--selector` / `-l` | string | `""` | Label selector to filter releases. Uses Kubernetes label selector syntax. Example: `--selector 'owner=platform-team,env=prod'`. |
| `--short` / `-q` | bool | `false` | Output only release names (one per line). Ignores `--output`. |
| `--superseded` | bool | `false` | Show only `superseded` releases. |
| `--time-format` | string | `""` | Go time format string for the `UPDATED` column. Overrides `--date-format`. |
| `--uninstalled` | bool | `false` | Show only `uninstalled` releases. (Requires `--keep-history` during uninstall.) |
| `--uninstalling` | bool | `false` | Show only releases currently being uninstalled. |

**Examples:**

1. List all deployed releases in the current namespace:
```bash
$ helm list
NAME        NAMESPACE   REVISION  UPDATED                   STATUS    CHART           APP VERSION
my-nginx    default     3         2026-07-11 14:30:00...    deployed  nginx-15.2.0    1.27.0
my-redis    default     1         2026-07-10 09:00:00...    deployed  redis-18.0.0    7.2.0
```

2. List all releases across all namespaces in JSON:
```bash
$ helm list -A -o json | jq '.[] | {name: .name, ns: .namespace, status: .status}'
{
  "name": "my-nginx",
  "ns": "default",
  "status": "deployed"
}
{
  "name": "monitoring",
  "ns": "monitoring",
  "status": "deployed"
}
```

3. List only failed releases for cleanup:
```bash
$ helm list --all --failed
NAME            NAMESPACE   REVISION  UPDATED                   STATUS    CHART
broken-release  default     1         2026-07-11 13:00:00...    failed    my-chart-1.0.0
```

**Output:** A table (or JSON/YAML) with columns: NAME, NAMESPACE, REVISION, UPDATED, STATUS, CHART, APP VERSION. With `--short`, only release names.

**Notes:**
- By default, `helm list` shows only `deployed` releases. Use `--all` to see releases in other states.
- `--uninstalled` only shows releases that were uninstalled with `--keep-history`.
- Releases are sorted by name alphabetically by default. Use `--reverse` for reverse alphabetical order.
- The `--selector` flag filters based on labels set with `--labels` during `helm install` / `helm upgrade`. These are NOT the same as Kubernetes resource labels.
- In Helm 3, releases are scoped to namespaces. `helm list` uses the namespace from `--namespace`, kubeconfig context, or `HELM_NAMESPACE`.

**Common Mistakes:**
- Running `helm list` without `--all` and thinking releases are missing — they may be in `failed` or `pending` state.
- Confusing `--filter` (regex on release name) with `--selector` (label query on release labels).
- Expecting `helm list` to show releases from all namespaces without `-A`.

**Related Commands:** `helm status`, `helm history`, `helm uninstall`

**CKA Relevance:** High — `helm list` is the most basic discovery command and essential for exam scenarios.

---

### helm package

**Syntax:**
```
helm package [CHART_PATH] [...] [flags]
```

**Description:** Packages a chart directory into a versioned chart archive (`.tgz` file). The archive is created in the current working directory. The chart version is taken from `Chart.yaml`. This command is used to prepare charts for distribution: uploading to repositories, pushing to OCI registries, or sharing with other teams.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--app-version` | string | `""` | Override the `appVersion` field in the packaged chart's `Chart.yaml`. Does not modify the source `Chart.yaml`. |
| `--dependency-update` / `-u` | bool | `false` | Run `helm dependency update` before packaging. Ensures `charts/` is populated and `Chart.lock` is current. |
| `--destination` / `-d` | string | `"."` | Directory where the `.tgz` archive will be written. |
| `--key` | string | `""` | Name of the GPG key to use for signing the chart. Produces a `.tgz.prov` provenance file alongside the archive. |
| `--keyring` | string | `"~/.gnupg/pubring.gpg"` | Path to the GPG keyring for signing. |
| `--passphrase-file` | string | `""` | Path to a file containing the GPG key passphrase. Avoids interactive passphrase prompts. |
| `--sign` | bool | `false` | Sign the chart archive with GPG. Requires `--key` to also be set. |
| `--version` | string | `""` | Override the `version` field in the packaged chart's `Chart.yaml`. Does not modify the source `Chart.yaml`. |

**Examples:**

1. Package a chart:
```bash
$ helm package my-chart/
Successfully packaged chart and saved it to: /home/user/projects/my-chart-1.2.3.tgz
```

2. Package with dependency update and custom version:
```bash
$ helm package my-chart/ --dependency-update --version 2.0.0 --destination ./dist/
Hang tight while we grab the latest from your chart repositories...
Update Complete. Happy Helming!
Saving 2 charts
Successfully packaged chart and saved it to: ./dist/my-chart-2.0.0.tgz
```

**Output:** A single confirmation message with the path to the created `.tgz` file. The `.tgz` is a gzipped tar archive containing the chart directory.

**Notes:**
- The archive filename follows the pattern `<chart-name>-<version>.tgz`, where the name and version are taken from `Chart.yaml`.
- `--app-version` and `--version` only affect the metadata inside the archive. The source `Chart.yaml` is NOT modified.
- When `--sign` is used, a `.tgz.prov` provenance file is created using GPG clearsign. This file contains the SHA-256 digest of the archive and can be verified with `helm verify`.
- Packaging always excludes the `charts/` directory from the archive (subcharts are expected to be downloaded separately via `helm dependency update` after pulling).

**Common Mistakes:**
- Forgetting to bump the version in `Chart.yaml` before packaging — you may overwrite a previously published archive.
- Using `--dependency-update` in automated pipelines without a reliable network connection.
- Packaging a chart with uncommitted local changes — the packaged chart may not match the VCS state.

**Related Commands:** `helm push`, `helm pull`, `helm dependency update`, `helm verify`, `helm lint`

**CKA Relevance:** Medium — chart packaging is part of the Helm workflow. The exam may ask you to prepare a chart for distribution.

---

### helm plugin

**Syntax:**
```
helm plugin <subcommand> [flags]
```

**Description:** Manages Helm plugins — external executables that extend Helm's functionality. Plugins are fetched from remote URLs, installed into `$HELM_DATA_HOME/plugins/`, and invoked via `helm <plugin-name>`. Common plugins include `helm-secrets`, `helm-diff`, `helm-ssm`, and `helm-s3`.

---

#### helm plugin install

**Syntax:**
```
helm plugin install [URL|PATH] [flags]
```

**Description:** Installs a Helm plugin from a URL (Git repository or HTTP archive) or a local path. The plugin must follow the Helm plugin specification: it must contain a `plugin.yaml` descriptor with a `name`, `version`, and `command`.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--version` | string | `""` | Specify a version constraint for the plugin. Works with Git tags (e.g., `--version ">=1.0.0"`). |

**Examples:**

1. Install the helm-diff plugin:
```bash
$ helm plugin install https://github.com/databus23/helm-diff
Downloading and installing helm-diff v3.9.0 ...
Installed plugin: diff
```

2. Install a specific version of a plugin:
```bash
$ helm plugin install https://github.com/jkroepke/helm-secrets --version v4.5.0
Downloading and installing helm-secrets v4.5.0 ...
Installed plugin: secrets
```

**Output:** Confirmation message showing the plugin name and installed version.

**Notes:**
- Plugins are installed into `$HELM_DATA_HOME/plugins/<name>/`. Each plugin gets its own subdirectory.
- The `plugin.yaml` descriptor must define: `name` (string), `version` (semver), `usage` (string), `description` (string), and `command` (string — the executable path relative to the plugin directory).
- For Git-based plugins, the repository is cloned. For HTTP archives, the archive is downloaded and extracted.
- Plugin executables can be any binary (Go, Python, shell script, etc.). Helm invokes them with all arguments after the plugin name.
- Plugins are not sandboxed — they run with the same privileges as the invoking user. Audit plugin source code before installing.

**Common Mistakes:**
- Installing a plugin that conflicts with a built-in Helm command name.
- Installing a plugin without reading its source code — plugins have full access to your kubeconfig and file system.
- Forgetting that plugins persist across Helm version upgrades.

**Related Commands:** `helm plugin list`, `helm plugin uninstall`, `helm plugin update`

**CKA Relevance:** Low — plugins are not included in the CKA exam.

---

#### helm plugin list

**Syntax:**
```
helm plugin list [flags]
```

**Description:** Lists all installed Helm plugins with their name, version, and description.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |

**Examples:**

1. List installed plugins:
```bash
$ helm plugin list
NAME    VERSION DESCRIPTION
diff    3.9.0   Preview helm upgrade changes as a diff
secrets 4.5.0   This plugin provides secrets values encryption
```

2. List plugins in JSON for scripting:
```bash
$ helm plugin list -o json | jq '.[].name'
"diff"
"secrets"
```

**Output:** A table (or JSON/YAML) with columns: NAME, VERSION, DESCRIPTION.

**Notes:**
- The list is generated by scanning `$HELM_DATA_HOME/plugins/` for directories containing a valid `plugin.yaml`.
- Plugins with missing or corrupt `plugin.yaml` files are silently skipped.

**Common Mistakes:** None — this is a read-only command.

**Related Commands:** `helm plugin install`, `helm plugin uninstall`, `helm plugin update`

**CKA Relevance:** Low.

---

#### helm plugin uninstall

**Syntax:**
```
helm plugin uninstall <PLUGIN_NAME> [flags]
```

**Description:** Uninstalls (removes) a Helm plugin by name. The plugin directory is deleted from `$HELM_DATA_HOME/plugins/`.

**Flags:** None beyond global flags.

**Examples:**

1. Uninstall a plugin:
```bash
$ helm plugin uninstall diff
Uninstalled plugin: diff
```

**Output:** Confirmation message.

**Notes:**
- Plugin removal is irreversible (directory is deleted). Ensure you have the plugin source URL if you need to reinstall.
- The plugin name must match exactly as shown in `helm plugin list`.

**Common Mistakes:**
- Trying to uninstall a plugin by its download URL rather than its name.
- Uninstalling a plugin that is used in CI/CD pipelines — breaks automated workflows.

**Related Commands:** `helm plugin install`, `helm plugin list`, `helm plugin update`

**CKA Relevance:** Low.

---

#### helm plugin update

**Syntax:**
```
helm plugin update <PLUGIN_NAME> [flags]

helm plugin update  # Updates ALL installed plugins
```

**Description:** Updates one or all installed Helm plugins to their latest version. For Git-based plugins, this runs `git pull`. For HTTP archive plugins, it re-downloads the archive.

**Flags:** None beyond global flags.

**Examples:**

1. Update a specific plugin:
```bash
$ helm plugin update diff
Updated plugin: diff
```

2. Update all installed plugins:
```bash
$ helm plugin update
Updating plugin: diff
Updated plugin: diff
Updating plugin: secrets
Updated plugin: secrets
```

**Output:** Confirmation for each updated plugin.

**Notes:**
- If a plugin name is provided, only that plugin is updated. If no name is provided, ALL plugins are updated.
- For Git-based plugins, `update` runs `git pull` in the plugin's directory. It does not switch branches or tags.
- The update mechanism depends on how the plugin was originally installed.

**Common Mistakes:**
- Running `helm plugin update` in a CI/CD environment without pinning plugin versions — may introduce unexpected changes.

**Related Commands:** `helm plugin install`, `helm plugin list`, `helm plugin uninstall`

**CKA Relevance:** Low.

---

### helm pull

**Syntax:**
```
helm pull [CHART_URL | REPO/CHART] [flags]
```

**Description:** Downloads a chart from a repository or OCI registry to the local file system as a `.tgz` archive. Unlike `helm install`, this command does NOT deploy the chart — it only fetches the package. Use this to inspect charts before deployment, mirror charts to private registries, or cache charts for offline use.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--ca-file` | string | `""` | CA certificate bundle for TLS verification of the repository. |
| `--cert-file` | string | `""` | Client certificate for TLS authentication. |
| `--destination` / `-d` | string | `"."` | Directory to write the chart archive. |
| `--devel` | bool | `false` | Include development (pre-release) versions. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS certificate verification. |
| `--key-file` | string | `""` | Client private key for TLS authentication. |
| `--keyring` | string | `"~/.gnupg/pubring.gpg"` | GPG keyring for provenance verification. |
| `--pass-credentials` | bool | `false` | Forward HTTP authentication credentials to all domains. |
| `--password` | string | `""` | Chart repository password. |
| `--plain-http` | bool | `false` | Use plain HTTP instead of HTTPS. |
| `--prov` | bool | `false` | Also download the provenance file (`.tgz.prov`) alongside the chart. |
| `--repo` | string | `""` | Chart repository URL. If set, CHART should be just the chart name. |
| `--untar` | bool | `false` | Untar (extract) the chart archive after downloading. The chart directory replaces the `.tgz`. |
| `--untardir` | string | `"."` | Directory to extract the chart into (when `--untar` is set). |
| `--username` | string | `""` | Chart repository username. |
| `--verify` | bool | `false` | Verify the chart's provenance file before saving. |
| `--version` | string | `""` | Chart version to download. If not specified, the latest stable version is pulled. |

**Examples:**

1. Pull a chart from an OCI registry:
```bash
$ helm pull oci://registry-1.docker.io/bitnamicharts/nginx --version 15.0.0
Pulled: registry-1.docker.io/bitnamicharts/nginx:15.0.0
Digest: sha256:abc123def456...
```

2. Pull and untar a chart for inspection:
```bash
$ helm pull bitnami/nginx --untar --destination ./charts-inspect/
$ tree ./charts-inspect/nginx/
./charts-inspect/nginx/
├── Chart.yaml
├── README.md
├── charts/
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ...
└── values.yaml
```

**Output:** A confirmation message with the chart reference and digest (for OCI). The chart `.tgz` archive is saved to the destination directory.

**Notes:**
- When pulling from an OCI registry (`oci://`), Helm uses the OCI distribution protocol, not the classic Helm chart repository protocol.
- For OCI pulls, the `--version` flag is required to identify the OCI tag.
- When pulling from classic repositories, if `--version` is omitted, the latest stable (non-prerelease) version is resolved from the repository index.
- `--untar` extracts the `.tgz` and deletes the archive. The chart content is directly available as a directory.

**Common Mistakes:**
- Forgetting `oci://` prefix when pulling from an OCI registry — Helm treats the reference as a classic repo path.
- Forgetting `--version` for OCI pulls — unlike classic repos, OCI references require a tag.
- Using `--untar` in a shared location and clobbering an existing chart directory.

**Related Commands:** `helm push`, `helm install`, `helm verify`, `helm show`

**CKA Relevance:** Medium — pulling charts is a prerequisite for inspecting and modifying them on the exam.

---

### helm push

**Syntax:**
```
helm push <CHART_TGZ> <REPO_URL> [flags]
```

**Description:** Uploads a packaged chart (`.tgz`) to a Helm chart repository or OCI registry. Starting from Helm 3.8+, native OCI support allows pushing via `helm push <chart>.tgz oci://<registry>/<namespace>`. For classic chart repositories, the `helm-push` plugin is required.

**Flags (OCI native push):**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--access-entry-type` | string | `""` | OCI access entry type. |
| `--ca-file` | string | `""` | CA certificate bundle. |
| `--cert-file` | string | `""` | Client certificate. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS verification. |
| `--key-file` | string | `""` | Client private key. |
| `--password` | string | `""` | Registry password. |
| `--plain-http` | bool | `false` | Use plain HTTP. |
| `--username` | string | `""` | Registry username. |

**Examples:**

1. Push a chart to an OCI registry:
```bash
$ helm push my-chart-1.2.3.tgz oci://registry-1.docker.io/myuser/
Pushed: registry-1.docker.io/myuser/my-chart:1.2.3
Digest: sha256:abc123def456...
```

2. Push with authentication:
```bash
$ helm registry login registry-1.docker.io
Username: myuser
Password:
Login Succeeded

$ helm push my-chart-1.2.3.tgz oci://registry-1.docker.io/myuser/
Pushed: registry-1.docker.io/myuser/my-chart:1.2.3
Digest: sha256:abc123...
```

**Output:** Confirmation with the pushed chart reference and content digest.

**Notes:**
- Native OCI push was introduced in Helm 3.8. For earlier versions, use the `helm-push` plugin.
- For classic chart repositories (ChartMuseum, Harbor, etc.), use the `helm cm-push` or `helm push` plugin.
- OCI push uses the same authentication as `helm registry login`. Credentials are stored in `$HELM_REGISTRY_CONFIG`.
- The chart `.tgz` file must be pre-packaged with `helm package` before pushing.

**Common Mistakes:**
- Forgetting to run `helm registry login` before pushing to a private registry.
- Trying to push to a classic chart repository without the `helm-push` plugin installed.
- Pushing a chart with a version that already exists in the registry (OCI registries typically reject tag overwrites).

**Related Commands:** `helm package`, `helm pull`, `helm registry login`, `helm registry logout`

**CKA Relevance:** Low — chart distribution is not a primary CKA topic.

---

### helm registry

**Syntax:**
```
helm registry <subcommand> [flags]
```

**Description:** Manages authentication with OCI-compliant container registries for chart storage. Subcommands handle login and logout. After logging in, credentials are stored in `$HELM_REGISTRY_CONFIG` and used automatically by `helm pull`, `helm push`, and `helm dependency update` when interacting with OCI registries.

---

#### helm registry login

**Syntax:**
```
helm registry login [HOST] [flags]
```

**Description:** Authenticates to an OCI registry. Prompts for username and password interactively unless provided via flags or stdin. Credentials are stored in the registry configuration file (`$HELM_REGISTRY_CONFIG`).

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--ca-file` | string | `""` | CA certificate bundle for TLS. |
| `--cert-file` | string | `""` | Client certificate. |
| `--insecure` | bool | `false` | Allow connections to the registry over plain HTTP. |
| `--key-file` | string | `""` | Client private key. |
| `--password` | string | `""` | Registry password. Use with `--password-stdin` for secure input. |
| `--password-stdin` | bool | `false` | Read password from stdin. **Recommended for CI/CD** to avoid exposing passwords in process lists. |
| `--username` / `-u` | string | `""` | Registry username. |

**Examples:**

1. Login interactively:
```bash
$ helm registry login registry-1.docker.io
Username: myuser
Password:
Login Succeeded
```

2. Login in CI/CD using stdin:
```bash
$ echo "$REGISTRY_PASSWORD" | helm registry login registry-1.docker.io \
    --username "$REGISTRY_USERNAME" --password-stdin
Login Succeeded
```

**Output:** `Login Succeeded` on success. On failure, an error message with exit code 1.

**Notes:**
- Credentials are stored in `$HELM_REGISTRY_CONFIG` (default: `~/.config/helm/registry/config.json`). This file follows the Docker config.json format.
- Helm registry credentials are separate from Docker registry credentials. Logging into Docker (`docker login`) does not automatically log Helm in.
- The `--password-stdin` flag is the recommended method for CI/CD pipelines — it prevents the password from appearing in process lists (`ps aux`).
- OCI registries supported include Docker Hub, AWS ECR, Azure ACR, Google Artifact Registry, Harbor, and any OCI-compliant registry.

**Common Mistakes:**
- Using `docker login` instead of `helm registry login` and expecting it to work.
- Exposing registry passwords in shell scripts without using `--password-stdin`.
- Forgetting to log out of a shared CI/CD runner, leaving credentials accessible to subsequent jobs.

**Related Commands:** `helm registry logout`, `helm push`, `helm pull`, `helm dependency update`

**CKA Relevance:** Medium — OCI registry authentication is part of the modern Helm workflow and may appear in exam contexts.

---

#### helm registry logout

**Syntax:**
```
helm registry logout [HOST] [flags]
```

**Description:** Removes stored credentials for an OCI registry host from the registry configuration file.

**Flags:** None beyond global flags.

**Examples:**

1. Logout from a registry:
```bash
$ helm registry logout registry-1.docker.io
Logout Succeeded
```

**Output:** `Logout Succeeded`.

**Notes:**
- This command modifies `$HELM_REGISTRY_CONFIG`. It does not interact with the registry — it only removes local credentials.
- If no HOST is specified, the command may fail or behave unpredictably. Always specify the host.

**Common Mistakes:**
- Expecting `helm registry logout` to revoke tokens on the registry server — it only removes the local credential entry.

**Related Commands:** `helm registry login`

**CKA Relevance:** Low.

---

### helm repo

**Syntax:**
```
helm repo <subcommand> [flags]
```

**Description:** Manages Helm chart repositories. Repositories are HTTP servers serving an `index.yaml` file that lists available charts and their download URLs. Subcommands handle adding, listing, removing, updating, and indexing repositories.

---

#### helm repo add

**Syntax:**
```
helm repo add [NAME] [URL] [flags]
```

**Description:** Adds a chart repository to Helm's local repository list. After adding, you can search for charts (`helm search repo <name>/<chart>`), pull charts, and install from the repository.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--allow-deprecated-repos` | bool | `false` | Allow adding repositories that have been marked as deprecated. |
| `--ca-file` | string | `""` | CA certificate bundle for TLS. |
| `--cert-file` | string | `""` | Client certificate for TLS. |
| `--force-update` | bool | `false` | Overwrite an existing repository entry with the same name. Without this flag, adding a repo that already exists returns an error. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS certificate verification. |
| `--key-file` | string | `""` | Client private key. |
| `--no-update` | bool | `false` | Skip the automatic index download that normally occurs after adding a repo. The repo is added but its index is not fetched. |
| `--pass-credentials` | bool | `false` | Forward HTTP authentication credentials to all domains. |
| `--password` | string | `""` | Repository authentication password. |
| `--plain-http` | bool | `false` | Use plain HTTP (not HTTPS) to connect to the repository. Required for repos served over insecure HTTP. |
| `--username` | string | `""` | Repository authentication username. |

**Examples:**

1. Add the Bitnami chart repository:
```bash
$ helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
```

2. Add a private authenticated repository:
```bash
$ helm repo add my-private-repo https://charts.mycompany.com/ \
    --username ci-user \
    --password "$REPO_PASSWORD"
"my-private-repo" has been added to your repositories
```

**Output:** `"<name>" has been added to your repositories` on success. The repository index is downloaded and cached unless `--no-update` is set.

**Notes:**
- Repository entries are stored in `$HELM_REPOSITORY_CONFIG` (default: `~/.config/helm/repositories.yaml`).
- After adding a repo, Helm automatically downloads the `index.yaml` and caches it in `$HELM_REPOSITORY_CACHE`.
- Repository names must be unique. Use `--force-update` to replace an existing entry.
- For OCI registries, you do NOT use `helm repo add`. Use `helm registry login` and reference charts directly via `oci://` URLs.
- Username and password are stored in plaintext in `repositories.yaml` if provided. **Prefer using short-lived tokens or OCI registries for production.**

**Common Mistakes:**
- Forgetting `--force-update` when re-adding a repo with new credentials.
- Confusing repo names with chart names — repos are sources of charts, not the charts themselves.
- Adding HTTP repos without `--plain-http` — Helm defaults to HTTPS.

**Related Commands:** `helm repo list`, `helm repo remove`, `helm repo update`, `helm search repo`

**CKA Relevance:** High — adding repositories is a prerequisite for installing most charts on the exam.

---

#### helm repo index

**Syntax:**
```
helm repo index [DIR] [flags]
```

**Description:** Generates an `index.yaml` file for a directory containing packaged chart `.tgz` archives. The index is the metadata file that Helm clients download to discover available charts and their versions. Use this when running your own Helm chart repository (e.g., served via NGINX, S3, or GitHub Pages).

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--merge` | string | `""` | Path to an existing `index.yaml` to merge new entries into. Useful for publishing new chart versions without rebuilding the entire index. |
| `--url` | string | `""` | Base URL prefix for chart download URLs in the generated index. Example: `--url https://charts.example.com/`. |

**Examples:**

1. Generate an index for a local chart directory:
```bash
$ helm repo index ./charts/ --url https://charts.example.com/
$ head -30 ./charts/index.yaml
apiVersion: v1
entries:
  my-app:
  - apiVersion: v2
    appVersion: 1.2.0
    created: "2026-07-11T14:30:00Z"
    description: A Helm chart for my application
    digest: 5d9a2e6f...
    name: my-app
    type: application
    urls:
    - https://charts.example.com/my-app-1.2.3.tgz
    version: 1.2.3
generated: "2026-07-11T14:30:00Z"
```

2. Merge a new chart into an existing index:
```bash
$ helm repo index ./new-charts/ --merge ./existing-index.yaml --url https://charts.example.com/
```

**Output:** An `index.yaml` file is written to the specified directory. No stdout output except on error.

**Notes:**
- The `--url` flag is required for the generated download URLs to be correct. Without it, the URLs will be relative (just the filename).
- The `--merge` flag is essential for incremental updates — without it, a new `index.yaml` overwrites the old one and loses all previous entries.
- The index generator reads all `.tgz` files in the directory, extracts `Chart.yaml` from each, and builds the index entries.
- The `digest` field in the index is a SHA-256 hash of the `.tgz` file, used for integrity verification.

**Common Mistakes:**
- Forgetting `--url` — the generated index has broken download links.
- Generating an index without `--merge` on an existing chart repo — all previous chart entries are lost.
- Including non-chart `.tgz` files in the directory — they cause index generation to fail.

**Related Commands:** `helm package`, `helm repo add`, `helm repo update`

**CKA Relevance:** Low — running your own repository is not a CKA topic.

---

#### helm repo list

**Syntax:**
```
helm repo list [flags]
```

**Description:** Lists all chart repositories currently configured in Helm's repository file.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |

**Examples:**

1. List all repositories:
```bash
$ helm repo list
NAME            URL
bitnami         https://charts.bitnami.com/bitnami
ingress-nginx   https://kubernetes.github.io/ingress-nginx
jetstack        https://charts.jetstack.io
```

2. List in JSON for automation:
```bash
$ helm repo list -o json | jq '.[].name'
"bitnami"
"ingress-nginx"
"jetstack"
```

**Output:** A table (or JSON/YAML) with columns: NAME, URL.

**Notes:**
- This reads from `$HELM_REPOSITORY_CONFIG`. It does not contact any remote servers.
- Repository credentials (username/password) are NOT displayed in the output.
- The list includes all repos added via `helm repo add`, including those that are currently unreachable.

**Common Mistakes:** None — this is a read-only command.

**Related Commands:** `helm repo add`, `helm repo remove`, `helm repo update`

**CKA Relevance:** Medium — knowing what repositories are configured is important for exam workflow.

---

#### helm repo remove

**Syntax:**
```
helm repo remove [REPO_NAME] [...] [flags]
```

**Description:** Removes one or more chart repositories from Helm's configuration file. Multiple repository names can be specified in a single command.

**Flags:** None beyond global flags.

**Examples:**

1. Remove a single repository:
```bash
$ helm repo remove jetstack
"jetstack" has been removed from your repositories
```

2. Remove multiple repositories at once:
```bash
$ helm repo remove jetstack ingress-nginx
"jetstack" has been removed from your repositories
"ingress-nginx" has been removed from your repositories
```

**Output:** `"<name>" has been removed from your repositories` for each removed repo.

**Notes:**
- This command does NOT delete the cached index files. Manually clear `$HELM_REPOSITORY_CACHE` to remove stale cache entries.
- Removing a repo does NOT uninstall any charts that were installed from it. Releases are independent of repository configuration.

**Common Mistakes:**
- Removing a repo and then being unable to run `helm dependency update` on charts that depend on it.
- Forgetting to remove cached indexes — they remain on disk indefinitely.

**Related Commands:** `helm repo add`, `helm repo list`, `helm repo update`

**CKA Relevance:** Medium.

---

#### helm repo update

**Syntax:**
```
helm repo update [REPO_NAME] [...] [flags]
```

**Description:** Updates the local cache of chart repository indexes. Fetches the latest `index.yaml` from each configured repository (or only the specified ones) and stores it in the repository cache directory. Run this after adding repos or before searching/installing to ensure you have the latest chart listings.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--fail-on-repo-update-fail` | bool | `false` | Exit with error if any repository fails to update. By default, a single repo failure prints a warning but does not cause the command to fail. |

**Examples:**

1. Update all repositories:
```bash
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
...Successfully got an update from the "ingress-nginx" chart repository
...Unable to get an update from the "jetstack" chart repository (https://charts.jetstack.io):
        Get "https://charts.jetstack.io/index.yaml": dial tcp: i/o timeout
Update Complete. Happy Helming!
```

2. Update a specific repository:
```bash
$ helm repo update bitnami
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. Happy Helming!
```

**Output:** Progress messages for each repository. Successes show "Successfully got an update." Failures show the error reason but do NOT exit with error unless `--fail-on-repo-update-fail` is set.

**Notes:**
- Cached indexes are stored in `$HELM_REPOSITORY_CACHE` (default: `~/.cache/helm/repository/`). Each repository gets a file named `<repo-name>-index.yaml`.
- Helm does NOT automatically update indexes before `helm install` or `helm search repo`. You must run `helm repo update` manually.
- If a repository is unreachable, Helm prints a warning and continues updating other repos. Use `--fail-on-repo-update-fail` in CI/CD.
- Repository index files can be large (the Bitnami index is 10+ MB). Updates may be slow on low-bandwidth connections.

**Common Mistakes:**
- Forgetting to run `helm repo update` after `helm repo add` — the search/install may use stale data or fail.
- Running `helm repo update` in a tight loop in CI/CD — cache the index once and reuse it via `--skip-refresh` on individual commands.
- Not using `--fail-on-repo-update-fail` in CI/CD — a repo outage may go unnoticed.

**Related Commands:** `helm repo add`, `helm search repo`, `helm dependency update --skip-refresh`

**CKA Relevance:** High — repository updates are part of the standard Helm workflow. You will likely run this before installing charts on the exam.

---

### helm rollback

**Syntax:**
```
helm rollback RELEASE_NAME [REVISION] [flags]
```

**Description:** Rolls back a release to a previous revision. This reverts the Kubernetes resources to the state they were in at the specified revision. If no REVISION is specified, it rolls back to the previous revision (current - 1). Rollback creates a new revision (not truly going back in time — it creates a new revision that is a copy of the old one).

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--cleanup-on-fail` | bool | `false` | Delete newly created resources if the rollback fails. |
| `--dry-run` | bool | `false` | Simulate the rollback without making changes. |
| `--force` | bool | `false` | Force resource updates through delete-and-recreate. |
| `--history-max` | int | `10` | Maximum number of revisions to retain after rollback. Overrides the value set at install time. |
| `--no-hooks` | bool | `false` | Disable hook execution during rollback. |
| `--recreate-pods` | bool | `false` | Perform a rolling restart of pods (deprecated — use `--restart-pods` in newer Helm versions). |
| `--timeout` | duration | `5m0s` | Maximum time to wait for the rollback to complete. |
| `--wait` | bool | `false` | Wait for all resources to reach a ready state before marking the rollback complete. |
| `--wait-for-jobs` | bool | `false` | Also wait for Jobs to complete successfully when `--wait` is set. |

**Examples:**

1. Rollback to the previous revision:
```bash
$ helm rollback my-release
Rollback was a success! Happy Helming!
```

2. Rollback to a specific revision with wait:
```bash
$ helm history my-release
REVISION  STATUS        ...
1         superseded    ...
2         superseded    ...
3         deployed       ...

$ helm rollback my-release 1 --wait
Rollback was a success! Happy Helming!

$ helm history my-release
REVISION  STATUS        ...
1         superseded    ...
2         superseded    ...
3         superseded    ...
4         deployed       ...  # New revision, content copied from revision 1
```

**Output:** `Rollback was a success! Happy Helming!` on success. On failure, an error message.

**Notes:**
- Rollback creates a **new revision** whose content is identical to the target revision. It does not delete intermediate revisions.
- For example, rolling back revision 5 to revision 2 creates revision 6, which is a copy of revision 2. Revision 5 is now `superseded`.
- The `--history-max` flag during rollback can be used to trim revision history. This is the only time the history max can be changed without reinstalling.
- Rollback respects existing resource states — if a resource was deleted manually after the target revision was deployed, the rollback will NOT recreate it automatically (Helm uses patches, not full resource replacement).
- Hook resources annotated with `helm.sh/hook: pre-rollback` and `post-rollback` are executed during rollback unless `--no-hooks` is set.

**Common Mistakes:**
- Rolling back to a revision that used a different CRD schema — the CRDs may be incompatible with the current cluster state.
- Assuming rollback restores the exact cluster state — it only reapplies the manifests from the target revision.
- Rolling back without checking `helm history` first — you may roll back to a failed revision.

**Related Commands:** `helm history`, `helm upgrade`, `helm install`, `helm get`

**CKA Relevance:** High — rollback is a key exam topic. The CKA specifically tests the ability to roll back a failed Helm release.

---

### helm search

**Syntax:**
```
helm search <subcommand> [QUERY] [flags]
```

**Description:** Searches for Helm charts. Two subcommands: `hub` searches the Artifact Hub (https://artifacthub.io) for all publicly available charts, and `repo` searches the locally configured repositories.

---

#### helm search hub

**Syntax:**
```
helm search hub [QUERY] [flags]
```

**Description:** Searches the Artifact Hub for Helm charts. This is a global search across all chart repositories indexed by the Artifact Hub. Requires internet access. The search query matches against chart name, description, and keywords.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--endpoint` | string | `"https://hub.helm.sh"` | URL of the Artifact Hub API endpoint. |
| `--list-repo-url` | bool | `false` | Show the repository URL for each chart in the output. |
| `--max-col-width` | uint | `50` | Maximum column width for the table output. |
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |

**Examples:**

1. Search for Prometheus charts:
```bash
$ helm search hub prometheus
URL                                                     CHART VERSION  APP VERSION  DESCRIPTION
https://artifacthub.io/packages/helm/prometheus-co...   25.0.0         2.50.0       Prometheus is a monitoring system...
https://artifacthub.io/packages/helm/prometheus-ada...   4.7.0          0.45.0       Prometheus adapter for Kubernetes
https://artifacthub.io/packages/helm/kube-prometheus...  9.0.0          0.72.0       kube-prometheus-stack collects...
```

2. Search with JSON output for programmatic filtering:
```bash
$ helm search hub nginx -o json | jq '.[] | {name: .name, repo: .repository_name}'
```

**Output:** A table (or JSON/YAML) with columns: URL, CHART VERSION, APP VERSION, DESCRIPTION. With `--list-repo-url`, an additional REPO URL column is shown.

**Notes:**
- `helm search hub` contacts `https://hub.helm.sh` and is subject to rate limits and availability of the Artifact Hub service.
- This command has no concept of "local repositories." It searches the entire public Helm ecosystem.
- Results from `helm search hub` cannot be directly installed — you must first `helm repo add` the relevant repository.

**Common Mistakes:**
- Trying to install a chart directly from `helm search hub` results without adding the corresponding repository.
- Using `helm search hub` in air-gapped environments — it requires internet access.

**Related Commands:** `helm search repo`, `helm repo add`, `helm install`

**CKA Relevance:** Low — the exam environment may not have internet access.

---

#### helm search repo

**Syntax:**
```
helm search repo [QUERY] [flags]
```

**Description:** Searches the locally configured chart repositories (added via `helm repo add`) for charts matching the query. Uses the locally cached index files. Run `helm repo update` first to ensure the cache is current.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--devel` | bool | `false` | Include development (pre-release) chart versions in results. |
| `--max-col-width` | uint | `50` | Maximum column width. |
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--regexp` | bool | `false` | Treat the QUERY as a regular expression instead of a substring match. |
| `--versions` | bool | `false` | Show all available versions for each chart (not just the latest stable). |
| `--version` | string | `""` | Filter results by a SemVer constraint. Example: `--version ">=1.0.0 <2.0.0"`. |

**Examples:**

1. Search all local repos for nginx charts:
```bash
$ helm search repo nginx
NAME                            CHART VERSION  APP VERSION  DESCRIPTION
bitnami/nginx                   15.2.0         1.27.0       NGINX Open Source is a web server...
bitnami/nginx-ingress-controller 9.9.0         1.10.0       NGINX Ingress Controller...
ingress-nginx/ingress-nginx     4.10.0         1.10.0       Ingress controller for Kubernetes...
```

2. Search with regex and show all versions:
```bash
$ helm search repo 'postgres.*' --regexp --versions | head -20
```

**Output:** A table (or JSON/YAML) with columns: NAME, CHART VERSION, APP VERSION, DESCRIPTION. With `--versions`, multiple rows per chart.

**Notes:**
- The search is performed against cached index files. Stale caches will show outdated versions.
- Chart names are prefixed with the repository name (e.g., `bitnami/nginx`).
- The `--regexp` flag uses Go's `regexp` package syntax (RE2), not PCRE. Lookaheads and backreferences are not supported.
- Without `--versions`, only the latest stable version of each chart is shown.

**Common Mistakes:**
- Searching without running `helm repo update` first — results may be outdated.
- Using the wrong repo prefix — the prefix is the name given to `helm repo add`, not necessarily the remote repository name.

**Related Commands:** `helm search hub`, `helm repo update`, `helm install`, `helm show`

**CKA Relevance:** High — finding charts is essential on the exam. You will likely search for a specific chart to install.

---

### helm show

**Syntax:**
```
helm show <subcommand> CHART [flags]
```

**Description:** Displays information about a chart from a repository or local path. Subcommands extract specific pieces: `all` (everything), `chart` (Chart.yaml), `crds` (CRD definitions), `readme` (README.md), and `values` (values.yaml).

---

#### helm show all

**Syntax:**
```
helm show all CHART [flags]
```

**Description:** Displays all available information about a chart: Chart.yaml, values.yaml, README.md, and CRDs. This is the combined output of `helm show chart`, `helm show values`, `helm show readme`, and `helm show crds`.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--ca-file` | string | `""` | CA certificate bundle. |
| `--cert-file` | string | `""` | Client certificate. |
| `--devel` | bool | `false` | Include development versions. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS verification. |
| `--key-file` | string | `""` | Client private key. |
| `--keyring` | string | `"~/.gnupg/pubring.gpg"` | GPG keyring for verification. |
| `--pass-credentials` | bool | `false` | Forward authentication credentials. |
| `--password` | string | `""` | Repository password. |
| `--plain-http` | bool | `false` | Use plain HTTP. |
| `--repo` | string | `""` | Repository URL. |
| `--username` | string | `""` | Repository username. |
| `--verify` | bool | `false` | Verify chart provenance. |
| `--version` | string | `""` | Chart version. |

**Examples:**

1. Show all information for a chart:
```bash
$ helm show all bitnami/nginx | head -50
annotations:
  category: Infrastructure
apiVersion: v2
appVersion: 1.27.0
description: NGINX Open Source is a web server...
name: nginx
version: 15.2.0
...
# Values:
image:
  registry: docker.io
  repository: bitnami/nginx
  tag: 1.27.0-debian-12-r0
  ...
```

2. Pipe show all to a file for offline review:
```bash
$ helm show all bitnami/nginx > nginx-chart-info.txt
```

**Output:** Concatenated YAML and Markdown content: first `Chart.yaml` content, then `values.yaml`, then `README.md`, then any CRDs.

**Notes:**
- This is a read-only operation — it pulls chart information from the repository but does not download the chart archive.
- For OCI charts, this does a manifest fetch (not a full chart pull).
- The output can be very long for complex charts.

**Common Mistakes:**
- Using `helm show all` for scripting — prefer the specific subcommands for machine-readable output.
- Expecting `helm show all` to work offline — it requires access to the chart repository.

**Related Commands:** `helm show chart`, `helm show values`, `helm show readme`, `helm show crds`, `helm pull`

**CKA Relevance:** Medium — inspecting charts before installation is a best practice the exam expects.

---

#### helm show chart

**Syntax:**
```
helm show chart CHART [flags]
```

**Description:** Displays the `Chart.yaml` metadata for a chart. Shows the chart name, version, appVersion, description, keywords, maintainers, dependencies, annotations, and other metadata.

**Flags:** Same as `helm show all`.

**Examples:**

1. View chart metadata:
```bash
$ helm show chart bitnami/nginx
annotations:
  category: Infrastructure
apiVersion: v2
appVersion: 1.27.0
description: NGINX Open Source is a web server that can be also used as a reverse proxy...
home: https://github.com/bitnami/charts/tree/main/bitnami/nginx
icon: https://bitnami.com/assets/stacks/nginx/img/nginx-stack-220x234.png
keywords:
- nginx
- http
- web
name: nginx
sources:
- https://github.com/bitnami/charts/tree/main/bitnami/nginx
type: application
version: 15.2.0
```

2. Extract the version for automation:
```bash
$ helm show chart bitnami/nginx --version 15.0.0 | grep '^version:'
version: 15.0.0
```

**Output:** The YAML content of `Chart.yaml`.

**Notes:**
- The `--version` flag specifies which chart version's metadata to retrieve. Without it, the latest stable version is used.
- For OCI charts, the version is specified as part of the OCI reference.

**Common Mistakes:** None — this is a straightforward read-only command.

**Related Commands:** `helm show all`, `helm show values`, `helm show readme`

**CKA Relevance:** Medium.

---

#### helm show crds

**Syntax:**
```
helm show crds CHART [flags]
```

**Description:** Displays the Custom Resource Definition (CRD) YAML files included in a chart's `crds/` directory. CRDs are installed at `helm install` time (before templates) and are never upgraded or deleted by Helm.

**Flags:** Same as `helm show all`.

**Examples:**

1. View CRDs in a chart:
```bash
$ helm show crds jetstack/cert-manager | head -30
---
# Source: cert-manager/templates/crd-certificaterequests.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: certificaterequests.cert-manager.io
  labels:
    app: cert-manager
    app.kubernetes.io/name: cert-manager
...
```

**Output:** Multi-document YAML containing the CRD definitions included in the chart.

**Notes:**
- CRDs are the only resources Helm treats specially. They are installed before any templates and are never modified during upgrades.
- Charts with no `crds/` directory produce no output.
- CRDs cannot be templated — they must be plain YAML files in the `crds/` directory.

**Common Mistakes:**
- Expecting `helm show crds` to show all Kubernetes resources — it only shows CRDs.

**Related Commands:** `helm show all`, `helm install --skip-crds`

**CKA Relevance:** Low — CRDs are an advanced topic, not heavily tested in CKA.

---

#### helm show readme

**Syntax:**
```
helm show readme CHART [flags]
```

**Description:** Displays the `README.md` file from a chart. The README typically contains usage instructions, parameter documentation, prerequisites, and examples.

**Flags:** Same as `helm show all`.

**Examples:**

1. View a chart's README:
```bash
$ helm show readme bitnami/nginx | head -40
# NGINX

[NGINX](https://nginx.org) is a web server that can be also used as a reverse proxy...

## TL;DR

```bash
helm install my-release oci://registry-1.docker.io/bitnamicharts/nginx
```

## Introduction

This chart bootstraps an [NGINX](...) deployment on a [Kubernetes](https://kubernetes.io) cluster...
```

**Output:** The Markdown content of `README.md`.

**Notes:**
- `helm show readme` does not template the README — it returns the raw Markdown as stored in the chart.
- Some charts have very long READMEs. Consider piping through `less` or `head`.

**Common Mistakes:** None — this is a straightforward read-only command.

**Related Commands:** `helm show all`, `helm show values`

**CKA Relevance:** Low — useful for understanding chart options but not essential for the exam.

---

#### helm show values

**Syntax:**
```
helm show values CHART [flags]
```

**Description:** Displays the `values.yaml` file from a chart. This shows all configurable parameters with their default values and documentation comments. Use this before `helm install` to understand what parameters you can customize.

**Flags:** Same as `helm show all`.

**Examples:**

1. View chart values:
```bash
$ helm show values bitnami/nginx | head -50
# Copyright Broadcom, Inc. All Rights Reserved.
# SPDX-License-Identifier: APACHE-2.0

## @section Global parameters
## @param global.imageRegistry Global Docker image registry
## @param global.imagePullSecrets Global Docker registry secret names as an array
## @param global.storageClass Global StorageClass for Persistent Volume(s)
##
global:
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: ""
  ...

## @section Common parameters
## @param kubeVersion Override Kubernetes version
##
kubeVersion: ""
## @param nameOverride String to partially override nginx.fullname template
nameOverride: ""
...
```

2. Save values for customization:
```bash
$ helm show values bitnami/nginx > my-custom-values.yaml
$ # Edit my-custom-values.yaml
$ helm install my-nginx bitnami/nginx -f my-custom-values.yaml
```

**Output:** The YAML content of `values.yaml`, including YAML comments documenting parameters.

**Notes:**
- The output is the chart's default `values.yaml`, NOT the values that would be applied after templating.
- Comments in `values.yaml` are preserved in the output, which is valuable for understanding parameter documentation.
- For OCI charts, use `--version` to specify which version's values to display.

**Common Mistakes:**
- Assuming these are the only parameters — some charts generate additional default values via `_helpers.tpl` or template logic.
- Not checking for nested parameters — `helm show values` returns the raw `values.yaml`, which may be long.

**Related Commands:** `helm show all`, `helm install -f`, `helm template`, `helm get values`

**CKA Relevance:** High — inspecting chart values is essential for customizing installations on the exam.

---

### helm status

**Syntax:**
```
helm status RELEASE_NAME [flags]
```

**Description:** Displays the current status of a Helm release, including its deployment state, the last deployment time, the namespace, the Kubernetes resources it manages, and any helpful notes. This is the primary command for checking the health of a deployed release.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--revision` | int | `0` | The revision to display. `0` means the latest. |
| `--show-desc` | bool | `false` | Show the release description (set via `--description` during install/upgrade). |
| `--show-resources` | bool | `false` | Show the Kubernetes resources created by the release with their status (e.g., Pods Running, Services active). |
| `--time-format` | string | `""` | Time format string for the LAST DEPLOYED field. |

**Examples:**

1. Check release status with resources:
```bash
$ helm status my-release --show-resources
NAME: my-release
LAST DEPLOYED: Sat Jul 11 14:30:00 2026
NAMESPACE: default
STATUS: deployed
REVISION: 3
TEST SUITE:     None
NOTES:
...

RESOURCES:
==> v1/ServiceAccount
NAME                       SECRETS   AGE
my-release-nginx           0         2h

==> v1/Service
NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
my-release-nginx           ClusterIP   10.96.1.100    <none>        80/TCP    2h

==> apps/v1/Deployment
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
my-release-nginx           3/3     3            3           2h

==> v1/Pod
NAME                                        READY   STATUS    RESTARTS   AGE
my-release-nginx-7d4f8b9c6-abc12            1/1     Running   0          2h
my-release-nginx-7d4f8b9c6-def34            1/1     Running   0          2h
my-release-nginx-7d4f8b9c6-ghi56            1/1     Running   0          2h
```

2. Get status in JSON for monitoring:
```bash
$ helm status my-release -o json | jq '{name: .name, status: .info.status, revision: .version}'
{
  "name": "my-release",
  "status": "deployed",
  "revision": 3
}
```

**Output:** A block showing NAME, LAST DEPLOYED, NAMESPACE, STATUS, REVISION, TEST SUITE, NOTES, and (with `--show-resources`) a list of all Kubernetes resources grouped by API version/kind.

**Notes:**
- `STATUS` values: `deployed`, `failed`, `pending-install`, `pending-upgrade`, `pending-rollback`, `uninstalling`.
- `--show-resources` queries the live Kubernetes API for resource status. This requires cluster access and may be slow for releases with many resources.
- The `TEST SUITE` field shows the result of the last `helm test` run. It is `None` if no tests have been run.
- Resource status under `--show-resources` is fetched live from the API server, unlike `helm get manifest` which shows stored manifests.

**Common Mistakes:**
- Confusing `STATUS: deployed` with "all pods are healthy" — a release can be `deployed` even if pods are crash-looping. Check `--show-resources` for actual pod health.
- Not using `--show-resources` when debugging — the default output shows only metadata, not actual Kubernetes resource health.

**Related Commands:** `helm list`, `helm history`, `helm get all`, `helm test`

**CKA Relevance:** High — checking release status is a fundamental operation. The exam will test your ability to diagnose deployment issues.

---

### helm template

**Syntax:**
```
helm template [NAME] [CHART] [flags]
```

**Description:** Renders chart templates locally and prints the generated Kubernetes YAML manifests to stdout. This command does NOT contact the Kubernetes API server and does NOT create a release. It is used for local validation, debugging template logic, generating manifests for GitOps workflows, and inspecting what a chart would deploy.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--api-versions` | stringArray | `[]` | Kubernetes API versions to use for `Capabilities.APIVersions` during template rendering. |
| `--ca-file` | string | `""` | CA certificate for repository access. |
| `--cert-file` | string | `""` | Client certificate. |
| `--debug` | bool | `false` | Enable verbose template output (shows computed values alongside rendered templates). |
| `--dependency-update` | bool | `false` | Run `helm dependency update` before rendering templates. |
| `--devel` | bool | `false` | Include development versions. |
| `--disable-openapi-validation` | bool | `false` | Skip OpenAPI validation of rendered templates. |
| `--enable-dns` | bool | `false` | Enable DNS lookups in templates (rare). |
| `--include-crds` | bool | `false` | Include CRDs from the `crds/` directory in the rendered output. By default, CRDs are excluded. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS verification. |
| `--is-upgrade` | bool | `false` | Render templates as if for an upgrade (sets `.Release.IsUpgrade` to `true`). Affects conditional logic in templates. |
| `--kube-version` | string | `""` | Kubernetes version to use for `Capabilities.KubeVersion`. Example: `--kube-version "1.29"`. |
| `--key-file` | string | `""` | Client private key. |
| `--keyring` | string | `"~/.gnupg/pubring.gpg"` | GPG keyring. |
| `--name-template` | string | `""` | Go template for the release name. |
| `--no-hooks` | bool | `false` | Exclude hook resources from the rendered output. |
| `--output-dir` | string | `""` | Write rendered manifests to a directory (one file per template) instead of stdout. |
| `--pass-credentials` | bool | `false` | Forward credentials. |
| `--password` | string | `""` | Repository password. |
| `--plain-http` | bool | `false` | Use plain HTTP. |
| `--post-renderer` | string | `""` | Post-renderer executable. |
| `--post-renderer-args` | stringSlice | `[]` | Post-renderer arguments. |
| `--release-name` | bool | `false` | Use the NAME argument as the release name. |
| `--render-subchart-notes` | bool | `false` | Include subchart NOTES.txt. |
| `--repo` | string | `""` | Repository URL. |
| `--set` | stringArray | `[]` | Set values. |
| `--set-file` | stringArray | `[]` | Set values from files. |
| `--set-json` | stringArray | `[]` | Set values from JSON. |
| `--set-literal` | stringArray | `[]` | Set literal values. |
| `--set-string` | stringArray | `[]` | Set string values. |
| `--show-only` | stringArray | `[]` | Show only specific template files. Example: `--show-only templates/deployment.yaml`. Can be specified multiple times. |
| `--skip-tests` | bool | `false` | Exclude test pod templates from the output. |
| `--username` | string | `""` | Repository username. |
| `--validate` | bool | `false` | Validate the rendered manifests against the Kubernetes API schema. Requires `--kube-version` or a running cluster. |
| `--values` / `-f` | stringArray | `[]` | Values files. |
| `--verify` | bool | `false` | Verify chart provenance. |
| `--version` | string | `""` | Chart version. |

**Examples:**

1. Render templates for local inspection:
```bash
$ helm template my-release bitnami/nginx --set replicaCount=2 | head -30
---
# Source: nginx/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-release-nginx
  labels:
    app.kubernetes.io/name: nginx
    helm.sh/chart: nginx-15.2.0
    app.kubernetes.io/instance: my-release
    app.kubernetes.io/managed-by: Helm
---
# Source: nginx/templates/service.yaml
apiVersion: v1
kind: Service
...
```

2. Render to files for GitOps (Argo CD / Flux):
```bash
$ helm template my-app ./my-chart/ \
    -f values-prod.yaml \
    --output-dir ./manifests/
Wrote ./manifests/my-chart/templates/deployment.yaml
Wrote ./manifests/my-chart/templates/service.yaml
Wrote ./manifests/my-chart/templates/ingress.yaml

$ tree manifests/
manifests/
└── my-chart
    └── templates
        ├── deployment.yaml
        ├── ingress.yaml
        └── service.yaml
```

3. Render only a specific template file and validate:
```bash
$ helm template my-app ./my-chart/ --show-only templates/deployment.yaml --validate
---
# Source: my-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
...
```

**Output:** Multi-document YAML with `# Source:` comments showing which template generated each resource. With `--output-dir`, files are written to disk and no stdout output is produced.

**Notes:**
- `helm template` is completely local. It does not require cluster access unless `--validate` is set.
- The `.Release` object is partially populated: `.Release.Name`, `.Release.Namespace`, `.Release.Service`, `.Release.IsUpgrade`, and `.Release.IsInstall` are set. `.Release.Revision` is always `1`.
- Unlike `helm install --dry-run`, `helm template` does NOT perform server-side validation or require API server connectivity.
- `--show-only` uses filename glob patterns matching template paths relative to the chart root.
- `--kube-version` and `--api-versions` simulate different Kubernetes clusters, affecting built-in objects like `Capabilities.KubeVersion`.

**Common Mistakes:**
- Using `helm template` output for production deployment (e.g., `helm template ... | kubectl apply -f -`) — this bypasses Helm's release tracking and lifecycle management.
- Forgetting to set `--is-upgrade` when testing upgrade-specific template logic.
- Assuming `.Release.Revision` is accurate — it is always `1` in template mode.
- Missing chart dependencies — run `helm dependency update` first or use `--dependency-update`.

**Related Commands:** `helm install --dry-run`, `helm lint`, `helm get manifest`

**CKA Relevance:** High — `helm template` is a key debugging and validation tool. The exam may ask you to render and inspect templates.

---

### helm test

**Syntax:**
```
helm test RELEASE_NAME [flags]
```

**Description:** Runs the test suite for a release. Tests are Kubernetes resources (typically Pods or Jobs) annotated with `helm.sh/hook: test`. Helm creates these resources and waits for them to complete (status: Succeeded) or fail (status: Failed). Test results are reported for each test resource.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--filter` | stringArray | `[]` | Run only tests matching this name. Can be specified multiple times. Example: `--filter test-connection`. |
| `--logs` | bool | `false` | Dump the logs from test pods after the test completes. Essential for debugging test failures. |
| `--timeout` | duration | `5m0s` | Maximum time to wait for tests to complete. Tests that exceed this timeout are marked as failed. |
| `--no-hooks` | bool | `false` | Disable running test hooks. (Rare — tests are hooks themselves.) |

**Examples:**

1. Run all tests for a release:
```bash
$ helm test my-release
NAME: my-release
LAST DEPLOYED: Sat Jul 11 14:30:00 2026
NAMESPACE: default
STATUS: deployed
REVISION: 3
TEST SUITE:     my-release-nginx-test-connection
Last Started:   Sat Jul 11 14:35:00 2026
Last Completed: Sat Jul 11 14:35:05 2026
Phase:          Succeeded

TEST SUITE:     my-release-nginx-test-health
Last Started:   Sat Jul 11 14:35:05 2026
Last Completed: Sat Jul 11 14:35:10 2026
Phase:          Succeeded
```

2. Run tests with logs for debugging:
```bash
$ helm test my-release --logs
NAME: my-release
...
Phase:          Failed
Pod logs:
Connecting to my-release-nginx:80...
Connection refused
```

3. Run only a specific test:
```bash
$ helm test my-release --filter test-connection
NAME: my-release
...
TEST SUITE:     my-release-nginx-test-connection
Phase:          Succeeded
```

**Output:** For each test resource, the test name, timestamps, and phase (Succeeded/Failed). With `--logs`, stdout/stderr from test pods.

**Notes:**
- Test resources are defined in `templates/tests/` in a chart. The default scaffolded chart includes a `test-connection.yaml`.
- Test pods are NOT automatically deleted after execution. Use `helm.sh/hook-delete-policy: hook-succeeded` annotation to auto-delete them.
- Tests run synchronously — Helm waits for each test to complete before starting the next.
- A failed test does NOT roll back the release. It only reports failure.
- Tests are re-entrant — running `helm test` multiple times creates new test pods each time.

**Common Mistakes:**
- Forgetting to include `--logs` when a test fails — without logs, debugging is difficult.
- Not setting `--timeout` appropriately — tests with long-running connections may need longer timeouts.
- Assuming tests clean themselves up — test pods persist by default and consume cluster resources.

**Related Commands:** `helm install`, `helm upgrade`, `helm status`, `helm get hooks`

**CKA Relevance:** Medium — test hooks are part of the Helm lifecycle. The exam may include chart testing scenarios.

---

### helm uninstall

**Syntax:**
```
helm uninstall RELEASE_NAME [...] [flags]
```

**Description:** Uninstalls a Helm release, deleting all Kubernetes resources that were created by the release, removing the release record from Helm's storage, and freeing the release name for reuse. Multiple releases can be uninstalled in a single command.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--cascade` | string | `"background"` | Kubernetes cascade deletion policy. Values: `background` (default — delete owner and then dependents in the background), `foreground` (delete dependents first, then owner), `orphan` (delete the Helm release record but NOT the Kubernetes resources). |
| `--deletion-propagation` | string | `"background"` | Same as `--cascade`. Overrides `--cascade` if both are set. |
| `--description` | string | `""` | Description added to the release metadata for this uninstall. |
| `--dry-run` | bool | `false` | Simulate the uninstall. Prints what would be deleted without making changes. |
| `--keep-history` | bool | `false` | Keep the release history (revision records) after uninstalling. The release appears in `helm list --uninstalled`. Useful for auditing. |
| `--no-hooks` | bool | `false` | Disable execution of pre-delete and post-delete hooks. |
| `--timeout` | duration | `5m0s` | Maximum time to wait for resource deletion. |
| `--wait` | bool | `false` | Wait for all resources to be fully deleted before returning. Without this, Helm returns immediately after initiating deletion. |

**Examples:**

1. Uninstall a release:
```bash
$ helm uninstall my-release
release "my-release" uninstalled
```

2. Uninstall with history preservation for auditing:
```bash
$ helm uninstall my-release --keep-history
release "my-release" uninstalled

$ helm list --uninstalled
NAME        NAMESPACE   REVISION  UPDATED                   STATUS        CHART
my-release  default     3         2026-07-11 15:00:00...    uninstalled   nginx-15.2.0
```

3. Dry-run to see what would be deleted:
```bash
$ helm uninstall my-release --dry-run
release "my-release" uninstalled (dry-run)
# No resources are actually deleted
```

4. Orphan resources (delete release record but keep resources):
```bash
$ helm uninstall my-release --cascade orphan
release "my-release" uninstalled

$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
my-release-nginx-abc       1/1     Running   0          5m
# Pods still exist because they were orphaned from the release
```

**Output:** `release "<name>" uninstalled` for each release.

**Notes:**
- Default behavior: Helm deletes the release record AND all Kubernetes resources created by the release.
- `--cascade orphan` deletes ONLY the Helm release record. The Kubernetes resources remain and become orphaned.
- `--cascade foreground` deletes dependents first (e.g., Pods before ReplicaSets). This is slower but ensures clean deletion order.
- `--cascade background` deletes the owner resource first and lets Kubernetes garbage-collect dependents in the background. This is the default.
- `--keep-history` stores the release record in `uninstalled` state. This keeps the release name reserved until the history is manually purged.

**Common Mistakes:**
- Uninstalling without `--keep-history` and then needing to audit what was deployed.
- Using `--cascade orphan` accidentally — this leaves orphaned resources in the cluster that must be cleaned up manually.
- Forgetting to add `--namespace` when the release is in a non-default namespace.

**Related Commands:** `helm install`, `helm list`, `helm history`, `helm status`

**CKA Relevance:** High — uninstalling releases is a basic Helm operation. The exam expects you to know how to clean up releases.

---

### helm upgrade

**Syntax:**
```
helm upgrade [RELEASE_NAME] [CHART] [flags]
```

**Description:** Upgrades an existing Helm release to a new version of a chart or with new configuration values. This is the primary command for deploying updates. The release name must already exist (use `helm install` for new releases). The chart version and values can be changed.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--atomic` | bool | `false` | If true, roll back on failure. Equivalent to `--wait && (on failure) helm rollback`. |
| `--ca-file` | string | `""` | CA certificate bundle. |
| `--cert-file` | string | `""` | Client certificate. |
| `--cleanup-on-fail` | bool | `false` | Delete newly created resources if the upgrade fails. |
| `--create-namespace` | bool | `false` | Create the namespace if it does not exist (only applies when `--install` is also used). |
| `--dependency-update` | bool | `false` | Run `helm dependency update` before upgrading. |
| `--description` | string | `""` | Description for this revision. Visible in `helm history`. |
| `--devel` | bool | `false` | Include development versions. |
| `--disable-openapi-validation` | bool | `false` | Skip OpenAPI validation. |
| `--dry-run` | bool | `false` | Simulate the upgrade. Templates are rendered but not submitted to the cluster. |
| `--dry-run-option` | string | `"none"` | Fine-grained dry-run mode: `none`, `client`, `server`. |
| `--enable-dns` | bool | `false` | Enable DNS lookups in templates. |
| `--force` | bool | `false` | Force resource updates through a delete-and-recreate strategy. **Use with caution** — causes downtime. |
| `--history-max` | int | `10` | Maximum number of revisions to retain. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS verification. |
| `--install` / `-i` | bool | `false` | If a release with this name does not exist, run `helm install` instead. Useful in CI/CD where you don't know if the release exists yet. |
| `--key-file` | string | `""` | Client private key. |
| `--keyring` | string | `"~/.gnupg/pubring.gpg"` | GPG keyring. |
| `--labels` | stringToString | `[]` | Labels to add to the release metadata. |
| `--no-hooks` | bool | `false` | Disable hook execution during upgrade. |
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--pass-credentials` | bool | `false` | Forward authentication credentials. |
| `--password` | string | `""` | Repository password. |
| `--plain-http` | bool | `false` | Use plain HTTP. |
| `--post-renderer` | string | `""` | Post-renderer executable. |
| `--post-renderer-args` | stringSlice | `[]` | Post-renderer arguments. |
| `--render-subchart-notes` | bool | `false` | Render subchart NOTES.txt. |
| `--repo` | string | `""` | Repository URL. |
| `--reset-then-reuse-values` | bool | `false` | Reset values to chart defaults, then re-apply the last release's user-supplied values, then merge any new overrides. **Recommended for most upgrade workflows.** |
| `--reset-values` | bool | `false` | Reset all values to chart defaults. Any previous user-supplied values are discarded. |
| `--reuse-values` | bool | `false` | Reuse the last release's values exactly as-is. Any new `--set` or `-f` overrides are merged on top. **Caution:** chart default changes in the new version are ignored. |
| `--set` | stringArray | `[]` | Set values on the command line. |
| `--set-file` | stringArray | `[]` | Set values from files. |
| `--set-json` | stringArray | `[]` | Set values from JSON. |
| `--set-literal` | stringArray | `[]` | Set literal values. |
| `--set-string` | stringArray | `[]` | Set string values. |
| `--skip-crds` | bool | `false` | Skip CRD verification. (CRDs are never upgraded by Helm, so this is largely informational.) |
| `--timeout` | duration | `5m0s` | Maximum time to wait for upgrade to complete. |
| `--username` | string | `""` | Repository username. |
| `--values` / `-f` | stringArray | `[]` | Values files. |
| `--verify` | bool | `false` | Verify chart provenance. |
| `--version` | string | `""` | Chart version to upgrade to. |
| `--wait` | bool | `false` | Wait for all resources to reach a ready state. |
| `--wait-for-jobs` | bool | `false` | Also wait for Jobs to complete successfully. |

**Examples:**

1. Upgrade a release with new values:
```bash
$ helm upgrade my-release bitnami/nginx \
    --set replicaCount=5 \
    --set image.tag=1.27.0 \
    --wait

Release "my-release" has been upgraded. Happy Helming!
NAME: my-release
LAST DEPLOYED: Sat Jul 11 15:00:00 2026
NAMESPACE: default
STATUS: deployed
REVISION: 4
...
```

2. Upgrade or install (idempotent, great for CI/CD):
```bash
$ helm upgrade --install my-release bitnami/nginx \
    --namespace prod \
    --create-namespace \
    --values values-prod.yaml \
    --atomic

Release "my-release" does not exist. Installing it now.
NAME: my-release
...
# Or, if it already exists:
Release "my-release" has been upgraded. Happy Helming!
```

3. Dry-run upgrade to preview changes:
```bash
$ helm upgrade my-release bitnami/nginx --dry-run --set replicaCount=10 | head -20
```

**Output:** Release summary on success. On failure with `--atomic`, the release is automatically rolled back.

**Notes:**
- `--reuse-values` keeps the previous release's values and merges any new `--set`/`-f` overrides. **However**, if the new chart version adds new default values, they are NOT picked up because the old values "cover" them.
- `--reset-then-reuse-values` is safer: it first resets to the new chart's defaults, then re-applies the user's previous values, then merges new overrides.
- `--reset-values` discards ALL previous user values and uses only the chart defaults plus any new `--set`/`-f` overrides.
- `--install` makes the command idempotent — it installs if the release doesn't exist, upgrades if it does. This is the preferred pattern for CI/CD pipelines.
- Upgrades create a new revision. The previous revision remains as `superseded` unless `--history-max` causes older revisions to be purged.

**Common Mistakes:**
- Using `--reuse-values` and then being confused when new chart defaults are not applied — use `--reset-then-reuse-values` instead.
- Forgetting `--install` in CI/CD and having pipelines fail because the release doesn't exist yet.
- Using `--force` casually — it causes resource downtime.
- Not bumping the chart version when upgrading to a new version of the same chart.

**Related Commands:** `helm install`, `helm rollback`, `helm history`, `helm template`, `helm get values`

**CKA Relevance:** High — upgrading releases is a core Helm operation. The CKA exam will test `helm upgrade` and value management during upgrades.

---

### helm verify

**Syntax:**
```
helm verify PATH [flags]
```

**Description:** Verifies the provenance (GPG signature) of a packaged chart archive (`.tgz`). The chart must have been signed with `helm package --sign` and a `.tgz.prov` provenance file must be present alongside the chart. Verification checks that the SHA-256 digest in the provenance file matches the chart archive and that the GPG signature is valid.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--keyring` | string | `"~/.gnupg/pubring.gpg"` | Path to the GPG keyring containing the public keys of chart signers. |

**Examples:**

1. Verify a signed chart:
```bash
$ helm verify my-chart-1.2.3.tgz
Signed by: John Doe <john@example.com>
Using Key with fingerprint: 5DE3E0509C47DE3A...
Chart Hash Verified: sha256:abc123def456...
```

2. Verify with a custom keyring:
```bash
$ helm verify my-chart-1.2.3.tgz --keyring /etc/helm/trusted-keys.gpg
Signed by: CI Pipeline <ci@company.com>
Using Key with fingerprint: F7AE...
Chart Hash Verified: sha256:def789...
```

**Output:** The signer identity, key fingerprint, and hash verification status. On failure, an error message explaining why (e.g., "no provenance file found", "signature verification failed", "hash mismatch").

**Notes:**
- The provenance file (`.tgz.prov`) must be in the same directory as the chart archive and have the same base filename.
- Verification checks two things: (1) the SHA-256 hash in the provenance matches the chart archive, (2) the GPG signature on the provenance file is valid and from a trusted key.
- Use `helm install --verify` or `helm upgrade --verify` to verify at install/upgrade time without running `helm verify` separately.

**Common Mistakes:**
- Expecting verification without a `.tgz.prov` file — the chart must be signed during packaging.
- Not having the signer's public key in the keyring — verification fails with "no public key" errors.

**Related Commands:** `helm package --sign`, `helm install --verify`, `helm dependency build --verify`

**CKA Relevance:** Low — chart signing is a security topic, not heavily tested in CKA.

---

### helm version

**Syntax:**
```
helm version [flags]
```

**Description:** Prints the version of the Helm client binary. With `--short`, prints only the client version string. This command does NOT require cluster access. It simply reports the build version, Git commit, Go version, and platform of the Helm binary.

**Flags:**

| Flag | Type | Default | Description |
|---|---|---|---|
| `--client` / `-c` | bool | `false` | (Legacy, Helm 2 compatibility. Ignored in Helm 3 — client version is always shown.) |
| `--output` / `-o` | string | `"table"` | Output format: `table`, `json`, or `yaml`. |
| `--short` | bool | `false` | Print only the version string (e.g., `v3.15.0`). |
| `--template` | string | `""` | Go template for formatting the output. Example: `--template '{{ .Version }}'`. |

**Examples:**

1. Show full version information:
```bash
$ helm version
version.BuildInfo{Version:"v3.15.0", GitCommit:"c4e74854886b2ef332ebd", GitTreeState:"clean", GoVersion:"go1.22.0"}
```

2. Show only the version string:
```bash
$ helm version --short
v3.15.0
```

3. Show version in JSON for scripting:
```bash
$ helm version -o json
{
  "Version": "v3.15.0",
  "GitCommit": "c4e74854886b2ef332ebd",
  "GitTreeState": "clean",
  "GoVersion": "go1.22.0"
}
```

4. Use a template for custom formatting:
```bash
$ helm version --template 'Helm {{ .Version }} (Go {{ .GoVersion }})'
Helm v3.15.0 (Go go1.22.0)
```

**Output:** Version information including the Helm version, Git commit SHA, Git tree state (clean/dirty), and Go version.

**Notes:**
- In Helm 2, `helm version` showed both client and server (Tiller) versions. In Helm 3, there is no server component — only the client version is shown.
- The `--client` flag is retained for backward compatibility but has no effect.
- The `--template` flag uses Go's `text/template` package. Available fields: `.Version`, `.GitCommit`, `.GitTreeState`, `.GoVersion`.

**Common Mistakes:**
- Expecting `helm version` to show the Kubernetes server version — it only shows the Helm client version. Use `kubectl version` for Kubernetes versions.
- Expecting `helm version` to require cluster access — it's a purely local operation.

**Related Commands:** `helm env`, `kubectl version`

**CKA Relevance:** Medium — checking the Helm version is useful to ensure compatibility, but not typically a tested task.

---

## 24.6 Quick Reference: Most Used Flag Shortcuts

| Short Flag | Long Flag | Used On |
|---|---|---|
| `-n` | `--namespace` | All commands (global) |
| `-f` | `--values` | `helm install`, `helm upgrade`, `helm template`, `helm lint` |
| `-o` | `--output` | `helm list`, `helm history`, `helm search repo`, etc. |
| `-A` | `--all-namespaces` | `helm list` |
| `-a` | `--all` | `helm list`, `helm get values` |
| `-d` | `--destination` | `helm pull`, `helm package` |
| `-i` | `--install` | `helm upgrade` |
| `-l` | `--selector` | `helm list` |
| `-p` | `--starter` | `helm create` |
| `-q` | `--short` | `helm list` |
| `-r` | `--reverse` | `helm list` |
| `-u` | `--dependency-update` | `helm package` |
| `-c` | `--client` | `helm version` (legacy, no-op in Helm 3) |

---

## 24.7 State Filter Cross-Reference

When troubleshooting releases, these `helm list` flags filter by state:

| State | Flag | 
|---|---|
| `deployed` | `--deployed` (default) |
| `failed` | `--failed` |
| `pending-install` | `--pending` |
| `pending-upgrade` | `--pending` |
| `pending-rollback` | `--pending` |
| `uninstalling` | `--uninstalling` |
| `uninstalled` | `--uninstalled` |
| `superseded` | `--superseded` |
| ALL states | `--all` / `-a` |

---

*End of Chapter 24 — Complete Command Reference.*
