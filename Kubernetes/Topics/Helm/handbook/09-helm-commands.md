# Chapter 9: Helm Commands — Complete Reference

This chapter provides an exhaustive reference for every Helm CLI command, including all
subcommands, flags, examples, production scenarios, exam-oriented tips, common mistakes, and
troubleshooting guidance.

---

## Table of Contents

1. [helm version](#1-helm-version)
2. [helm env](#2-helm-env)
3. [helm plugin](#3-helm-plugin)
4. [helm registry](#4-helm-registry)
5. [helm create](#5-helm-create)
6. [helm package](#6-helm-package)
7. [helm lint](#7-helm-lint)
8. [helm install](#8-helm-install)
9. [helm upgrade](#9-helm-upgrade)
10. [helm rollback](#10-helm-rollback)
11. [helm uninstall](#11-helm-uninstall)
12. [helm list](#12-helm-list)
13. [helm history](#13-helm-history)
14. [helm status](#14-helm-status)
15. [helm test](#15-helm-test)
16. [helm get](#16-helm-get)
17. [helm pull](#17-helm-pull)
18. [helm push](#18-helm-push)
19. [helm verify](#19-helm-verify)
20. [helm show](#20-helm-show)
21. [helm search](#21-helm-search)
22. [helm template](#22-helm-template)
23. [helm dependency](#23-helm-dependency)
24. [helm repo](#24-helm-repo)
25. [helm registry login / logout](#25-helm-registry-login--logout)
26. [helm completion](#26-helm-completion)
27. [helm help](#27-helm-help)
28. [Summary Tables](#28-summary-tables)

---

## 1. helm version

### Purpose

Displays the version of the Helm client being used. Optionally prints a Go template string.
It helps verify that the correct Helm binary is installed and matches the server (Tiller in
Helm v2, or the Kubernetes API version in Helm v3).

### Syntax

```
helm version [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--short`, `-s` | bool | `false` | Print only the version number (compact output). The output becomes just the semantic version string, e.g. `v3.15.0`, without any extra metadata. |
| `--template` | string | `""` | A Go template string used to format the output. Accepts the same template variables as `helm version` normally prints: `.Version`, `.GitCommit`, `.GitTreeState`, `.GoVersion`. |

### Real-World Examples

```bash
# Basic version check
helm version

# Output (example):
# version.BuildInfo{Version:"v3.15.0", GitCommit:"c9e33a0a7a9b9e5b5e5b5e5b5e5b5e5b5e5b5e5", GitTreeState:"clean", GoVersion:"go1.22.3"}
```

```bash
# Short, script-friendly output
helm version --short

# Output:
# v3.15.0
```

```bash
# Custom template output (useful for CI pipelines)
helm version --template 'Helm {{ .Version }} (commit {{ .GitCommit }})'

# Output:
# Helm v3.15.0 (commit c9e33a0a7a9b9e5b5e5b5e5b5e5b5e5b5e5b5e5)
```

```bash
# Check version in CI without extra noise
HELM_VERSION=$(helm version --short --template '{{ .Version }}')
echo "Deploying with Helm $HELM_VERSION"
```

### Production Scenario

In a CI/CD pipeline, you may need to assert that a minimum Helm version is available before
proceeding:

```bash
MIN_VERSION="3.12.0"
CURRENT_VERSION=$(helm version --short --template '{{ .Version }}' | sed 's/^v//')
if [[ "$(printf '%s\n' "$MIN_VERSION" "$CURRENT_VERSION" | sort -V | head -n1)" != "$MIN_VERSION" ]]; then
  echo "ERROR: Helm >= $MIN_VERSION required, found $CURRENT_VERSION"
  exit 1
fi
```

### Exam Scenario

CKA/CKAD exams often ask you to verify the installed Helm version. Run `helm version --short`.
The exam environment typically has Helm v3+ pre-installed.

### Common Mistakes

- Running `helm version` expecting to see the Kubernetes server version. Helm v3 does NOT
  connect to Tiller; it only reports the client version.
- Using `--template` with an invalid Go template syntax, causing a silent empty output.

### Troubleshooting

| Problem | Solution |
|---|---|
| `helm: command not found` | Helm is not installed or not in `$PATH`. Run `which helm` or reinstall. |
| Template prints nothing | Verify Go template syntax. Use `helm version --template '{{ .Version }}'` as a minimal test. |

### Related Commands

- `helm env` — shows all environment variables Helm uses
- `kubectl version` — shows the Kubernetes client and server versions

---

## 2. helm env

### Purpose

Lists all environment variables that Helm recognizes and their current values. Useful for
debugging configuration issues, verifying Helm's runtime context, and understanding which
variables influence Helm's behaviour.

### Syntax

```
helm env [flags]
```

This command accepts no subcommands and has no flags.

### Environment Variables

The full list of environment variables printed by `helm env`:

| Variable | Typical Value | Purpose |
|---|---|---|
| `HELM_BIN` | `/usr/local/bin/helm` | Path to the Helm binary |
| `HELM_BURST_LIMIT` | `100` | Kubernetes client burst limit (QPS burst) |
| `HELM_CACHE_HOME` | `~/.cache/helm` | Directory for cached data (repo indexes, charts) |
| `HELM_CONFIG_HOME` | `~/.config/helm` | Directory for configuration files (repositories.yaml, registry.json) |
| `HELM_DATA_HOME` | `~/.local/share/helm` | Directory for Helm data (plugins, starters) |
| `HELM_DEBUG` | `false` | Enable verbose debug logging |
| `HELM_KUBEAPISERVER` | (empty) | Override the Kubernetes API server address |
| `HELM_KUBEASGROUPS` | (empty) | Comma-separated list of groups for impersonation |
| `HELM_KUBEASUSER` | (empty) | Username for impersonation |
| `HELM_KUBECAFILE` | (empty) | Path to CA certificate for Kubernetes API |
| `HELM_KUBECONTEXT` | (empty) | Kubernetes context name override |
| `HELM_KUBEINSECURE_SKIP_TLS_VERIFY` | `false` | Skip TLS verification |
| `HELM_KUBETOKEN` | (empty) | Bearer token for authentication |
| `HELM_MAX_HISTORY` | `10` | Default maximum number of revisions to keep |
| `HELM_NAMESPACE` | `default` | Default Kubernetes namespace |
| `HELM_PLUGINS` | `~/.local/share/helm/plugins` | Plugins directory |
| `HELM_REGISTRY_CONFIG` | `~/.config/helm/registry/config.json` | OCI registry auth config path |
| `HELM_REPOSITORY_CACHE` | `~/.cache/helm/repository` | Repository cache directory |
| `HELM_REPOSITORY_CONFIG` | `~/.config/helm/repositories.yaml` | Repository list config file |

### Real-World Examples

```bash
# See all Helm environment variables
helm env

# Sample output (abbreviated):
# HELM_BIN="/usr/local/bin/helm"
# HELM_CACHE_HOME="/home/user/.cache/helm"
# HELM_CONFIG_HOME="/home/user/.config/helm"
# HELM_DATA_HOME="/home/user/.local/share/helm"
# HELM_DEBUG="false"
# HELM_NAMESPACE="default"
# ...
```

```bash
# Check a specific variable in a script
helm env | grep HELM_NAMESPACE
```

```bash
# Pipe to grep to verify cache location
helm env | grep HELM_CACHE_HOME
```

### Production Scenario

In a locked-down production environment, you may need to override the cache and config
directories to ephemeral storage to comply with read-only root filesystem policies. Use
`helm env` to verify those overrides are being picked up:

```bash
export HELM_CACHE_HOME=/tmp/helm-cache
export HELM_CONFIG_HOME=/tmp/helm-config
helm env | grep -E 'HELM_CACHE_HOME|HELM_CONFIG_HOME'
```

### Exam Scenario

CKAD/CKA exams: `helm env` is handy to check that `HELM_NAMESPACE` is set correctly before
installing charts into the expected namespace. The exam environment may pre-set `HELM_NAMESPACE`.

### Common Mistakes

- Confusing `HELM_HOME` (deprecated in Helm v3) with `HELM_CONFIG_HOME`.
- Setting `HELM_NAMESPACE` and then using `--namespace` as well — the flag takes precedence.

### Troubleshooting

| Problem | Solution |
|---|---|
| Plugins not loading | Check `HELM_PLUGINS` points to the correct directory |
| Repo list empty | Verify `HELM_REPOSITORY_CONFIG` and `HELM_REPOSITORY_CACHE` |

### Related Commands

- `helm version` — version info
- `helm repo list` — shows configured repositories

---

## 3. helm plugin

### Purpose

Manages Helm plugins — extensions that add new subcommands to Helm. Plugins are executables
that follow Helm's plugin protocol, enabling custom functionality like secrets management,
diffing, or custom chart operations.

### Subcommands

| Subcommand | Purpose |
|---|---|
| `install` | Install a plugin from a URL (local path, remote tarball, or Git repo) |
| `list` | List all installed plugins |
| `uninstall` | Remove an installed plugin |
| `update` | Update all installed plugins to their latest versions |

### 3.1 helm plugin install

#### Syntax

```
helm plugin install [url] [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--version` | string | `""` | Specify a version constraint. If the plugin source is a Git repo, this can be a tag, branch, or commit hash. |

#### Real-World Examples

```bash
# Install the helm-diff plugin (shows a diff before upgrade)
helm plugin install https://github.com/databus23/helm-diff

# Install a specific version
helm plugin install https://github.com/databus23/helm-diff --version v3.8.1

# Install from a local directory
helm plugin install ./my-custom-plugin/

# Install from a tarball URL
helm plugin install https://example.com/plugins/my-plugin.tar.gz
```

#### Output Example

```
Downloading and installing helm-diff v3.8.1 ...
Installed plugin: diff
```

### 3.2 helm plugin list

#### Syntax

```
helm plugin list [flags]
```

No flags.

#### Output Example

```
NAME    VERSION DESCRIPTION
diff    3.8.1   Preview helm upgrade changes as a diff
secrets 4.5.1   This plugin provides secrets values encryption
```

### 3.3 helm plugin uninstall

#### Syntax

```
helm plugin uninstall <plugin-name> [flags]
```

No flags.

#### Real-World Example

```bash
helm plugin uninstall diff
```

#### Output Example

```
Uninstalled plugin: diff
```

### 3.4 helm plugin update

#### Syntax

```
helm plugin update <plugin-name> [flags]
```

If no plugin name is given, ALL plugins are updated.

No flags.

#### Real-World Examples

```bash
# Update a specific plugin
helm plugin update diff

# Update all installed plugins
helm plugin update
```

#### Output Example

```
Updated plugin: diff
```

### Production Scenario

In CI pipelines, ensure required plugins are installed before running deployments:

```bash
#!/bin/bash
REQUIRED_PLUGINS=("diff" "secrets")
for plugin in "${REQUIRED_PLUGINS[@]}"; do
  if ! helm plugin list | grep -q "$plugin"; then
    echo "Plugin $plugin not found, installing..."
    helm plugin install "https://github.com/some-org/helm-${plugin}"
  fi
done
```

### Exam Scenario

The CKA/CKAD exams typically do not require plugin knowledge, but the Helm Certified Associate
exam may ask about the purpose of plugins and how to list installed plugins.

### Common Mistakes

- Installing plugins from untrusted URLs. Always verify the source.
- Forgetting that `helm plugin update` without a name updates ALL plugins, which may break
  compatibility.
- Using `--version` with a non-Git-based plugin source — the flag is generally only supported
  for Git-based installs.

### Troubleshooting

| Problem | Solution |
|---|---|
| Plugin install fails with "permission denied" | Check permissions on `HELM_PLUGINS` directory |
| Plugin does not appear in `helm plugin list` | The plugin `plugin.yaml` may be malformed; check the plugin's install directory |
| Update fails | Some plugins do not support updates; reinstall by uninstalling then installing again |

### Related Commands

- `helm env` — shows `HELM_PLUGINS` directory location
- `helm help` — installed plugins extend the help output automatically

---

## 4. helm registry

> **Note**: `helm registry` commands were introduced for OCI-based registries. In Helm v3.8+,
> OCI support is native and the `helm registry` subcommand is available. In earlier v3
> versions, these commands may not exist or may behave differently.

### Purpose

Manages authentication with OCI-compliant container registries for storing and retrieving
Helm charts as OCI artifacts.

### Subcommands

| Subcommand | Purpose |
|---|---|
| `login` | Log in to an OCI registry |
| `logout` | Log out from an OCI registry |

### 4.1 helm registry login

#### Syntax

```
helm registry login [host] [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--insecure` | bool | `false` | Allow connections to TLS registries without valid certificates |
| `--password`, `-p` | string | `""` | Registry password or identity token (prompts if not provided) |
| `--password-stdin` | bool | `false` | Read password from stdin |
| `--username`, `-u` | string | `""` | Registry username |

#### Real-World Examples

```bash
# Interactive login (prompts for password)
helm registry login registry.example.com

# Non-interactive login with flags
helm registry login registry.example.com -u myuser -p mypassword

# Login via pipeline (password from stdin)
echo "$REGISTRY_PASSWORD" | helm registry login registry.example.com -u myuser --password-stdin

# Docker Hub
helm registry login registry-1.docker.io -u mydockerhubuser

# GitHub Container Registry
echo "$GITHUB_TOKEN" | helm registry login ghcr.io -u myuser --password-stdin

# Insecure registry (development only)
helm registry login myregistry.local:5000 -u admin --insecure
```

### 4.2 helm registry logout

#### Syntax

```
helm registry logout [host] [flags]
```

No flags.

#### Real-World Example

```bash
helm registry logout registry.example.com
```

#### Output Example

```
Logout succeeded
```

### Production Scenario

In a CI/CD pipeline, authenticate to a private OCI registry to push/pull charts:

```bash
echo "$OCI_REGISTRY_TOKEN" | helm registry login "$OCI_REGISTRY" \
  -u "$OCI_REGISTRY_USER" --password-stdin

# Push a packaged chart
helm push mychart-1.0.0.tgz oci://$OCI_REGISTRY/charts/

# Always logout when done (security best practice)
helm registry logout "$OCI_REGISTRY"
```

### Exam Scenario

The Helm Certified Associate exam tests OCI registry usage. Know the difference between
`helm repo` (classic HTTP chart repositories) and `helm registry` (OCI registries).

### Common Mistakes

- Using `helm repo` commands (add, push) for OCI registries — OCI uses `helm registry login`
  and `helm push/pull` with `oci://` protocol.
- Forgetting to logout from registries in CI, potentially leaving credentials in shared
  runners.
- Using `--password` in command-line arguments (visible in process lists and shell history).
  Prefer `--password-stdin`.

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: could not find registry` | The OCI registry hostname may be unreachable or incorrect |
| `unauthorized: authentication required` | Credentials are invalid or missing; re-login |
| `tls: failed to verify certificate` | The registry uses a self-signed cert; use `--insecure` (non-production only) or configure CA trust |

### Related Commands

- `helm push` — push a packaged chart to an OCI registry
- `helm pull` — pull a chart from an OCI registry
- `helm repo` — manage classic HTTP chart repositories

---

## 5. helm create

### Purpose

Scaffolds a new Helm chart directory with the standard layout and boilerplate files. This is
the starting point for creating custom charts.

### Syntax

```
helm create NAME [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--starter`, `-p` | string | `""` | Name or path of a starter template to base the chart on. Starters are stored in `$HELM_DATA_HOME/starters`. |

### Files Generated

When you run `helm create mychart`, the following structure is created:

```
mychart/
├── Chart.yaml          # Chart metadata: name, version, description, apiVersion, etc.
├── values.yaml         # Default configuration values for the chart
├── charts/             # Directory for chart dependencies (subcharts)
├── templates/          # Kubernetes resource templates (Go templates)
│   ├── NOTES.txt       # Post-install usage notes displayed after `helm install`
│   ├── _helpers.tpl    # Reusable template helpers (partials, named templates)
│   ├── deployment.yaml # Default Deployment resource template
│   ├── hpa.yaml        # HorizontalPodAutoscaler template
│   ├── ingress.yaml    # Ingress resource template
│   ├── service.yaml    # Service resource template
│   ├── serviceaccount.yaml # ServiceAccount template
│   └── tests/          # Helm test pod definitions
│       └── test-connection.yaml
└── .helmignore         # Files to exclude from the packaged chart (like .gitignore)
```

#### File Descriptions

| File | Purpose |
|---|---|
| `Chart.yaml` | **Mandatory**. Contains `apiVersion` (v1 or v2), `name`, `version`, `appVersion`, `description`, `type` (application or library), `keywords`, `maintainers`, `dependencies`, and more. |
| `values.yaml` | **Default values**. All template references to `.Values.*` resolve against this file, overridden by user-supplied `--values` or `--set` flags. |
| `charts/` | Houses chart dependencies (subcharts). Dependencies declared in `Chart.yaml` are downloaded here by `helm dependency update`. |
| `templates/` | Contains all Kubernetes manifest templates. Any `.yaml`, `.tpl`, or `.yml` file here is processed by the Helm template engine. |
| `templates/NOTES.txt` | Rendered and displayed after `helm install` or `helm upgrade`. Typically includes usage instructions, next steps, and connection details. |
| `templates/_helpers.tpl` | Named templates (partials) used across other templates. Common for defining labels, selectors, and full names. |
| `templates/deployment.yaml` | Default Deployment with liveness/readiness probes, resource requests, and a `nginx` container. |
| `templates/hpa.yaml` | HPA bound to the Deployment; disabled by default (controlled by `values.yaml`). |
| `templates/ingress.yaml` | Ingress resource; disabled by default unless `ingress.enabled` is true. |
| `templates/service.yaml` | ClusterIP Service (configurable type via `values.yaml`). |
| `templates/serviceaccount.yaml` | ServiceAccount with optional annotations and automount token control. |
| `templates/tests/test-connection.yaml` | A Pod for `helm test` that validates the chart's Service is reachable. |
| `.helmignore` | Glob patterns for files excluded during `helm package`. Similar to `.gitignore`. |

### Real-World Examples

```bash
# Basic chart creation
helm create web-app

# Create a chart with a starter template
helm create my-microservice -p microservice-starter

# In an organization with standardized starters:
helm create api-service -p company-api-starter
```

### Production Scenario

Organizations maintain custom starter charts that enforce internal standards (security
contexts, label conventions, resource limits, monitoring annotations). Developers run
`helm create myservice -p company-starter` to scaffold a compliant chart:

```bash
# Install the company starter first
helm plugin install ./company-starter-plugin

# Create a new service chart
helm create customer-api -p company-starter

# Customize values.yaml for the specific service
vim customer-api/values.yaml
```

### Exam Scenario

CKA/CKAD: you may be asked to create a chart from scratch. `helm create` gives you a working
template immediately. You then modify `values.yaml` and `templates/` as required.

### Common Mistakes

- Running `helm create` in a non-empty directory. It must target a new or non-existent
  directory.
- Deleting `_helpers.tpl` — other templates reference its named templates (e.g.,
  `mychart.fullname`, `mychart.labels`, `mychart.selectorLabels`). If you remove it,
  template rendering fails.
- Forgetting to update `Chart.yaml` with correct metadata before packaging.

### Troubleshooting

| Problem | Solution |
|---|---|
| Starter template not found | Verify the starter exists in `$HELM_DATA_HOME/starters/` |
| `Error: directory mychart already exists` | Choose a different name or remove the existing directory |
| Template rendering errors after modifying files | Check that `_helpers.tpl` is intact and `values.yaml` references are correct |

### Related Commands

- `helm lint` — validate the chart's structure and templates
- `helm package` — package the chart for distribution
- `helm template` — render templates locally without installing

---

## 6. helm package

### Purpose

Packages a chart directory into a versioned `.tgz` (gzipped tar) archive for distribution
via chart repositories or direct transfer. The resulting archive is the standard unit of
Helm chart distribution.

### Syntax

```
helm package [CHART_PATH] [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--app-version` | string | `""` | Override the `appVersion` field in `Chart.yaml` for this package. The original `Chart.yaml` is NOT modified. |
| `--version` | string | `""` | Override the `version` field in `Chart.yaml` for this package. The original `Chart.yaml` is NOT modified. |
| `--destination`, `-d` | string | `"."` | Directory where the packaged `.tgz` file is written. |
| `--dependency-update`, `-u` | bool | `false` | Run `helm dependency update` before packaging. Downloads any missing or outdated dependencies declared in `Chart.yaml`. |
| `--sign` | bool | `false` | Sign the chart using a PGP private key. Requires `--key` or `--keyring`. Produces a `.prov` provenance file alongside the `.tgz`. |
| `--key` | string | `""` | Name of the PGP private key to use for signing. Must exist in the keyring. |
| `--keyring` | string | `"~/.gnupg/secring.gpg"` | Path to the PGP keyring file containing the signing key. |
| `--passphrase-file` | string | `""` | Path to a file containing the passphrase for the PGP private key. Useful for non-interactive signing in CI/CD. |

### Real-World Examples

```bash
# Basic packaging
helm package ./mychart

# Output:
# Successfully packaged chart and saved it to: /path/to/mychart-0.1.0.tgz
```

```bash
# Override version and destination
helm package ./mychart --version 1.2.3 --destination ./releases/

# Output:
# Successfully packaged chart and saved it to: ./releases/mychart-1.2.3.tgz
```

```bash
# Package with dependency resolution
helm package ./mychart --dependency-update

# Override appVersion (CI use case: match the container image tag)
helm package ./mychart --app-version "v2.4.1" --version "2.4.1"
```

```bash
# Signed chart packaging
helm package ./mychart --sign --key "my-signing-key" --keyring ~/.gnupg/secring.gpg

# Output:
# Successfully packaged chart and saved it to: ./mychart-0.1.0.tgz
# Signed and saved provenance file to: ./mychart-0.1.0.tgz.prov
```

```bash
# CI-friendly signed packaging with passphrase from file
helm package ./mychart \
  --sign \
  --key "ci-signing-key" \
  --keyring /secrets/secring.gpg \
  --passphrase-file /secrets/passphrase.txt
```

### Production Scenario

A typical CI pipeline for chart publication:

```bash
#!/bin/bash
CHART_DIR="./mychart"
VERSION=$(git describe --tags --abbrev=0)
APP_VERSION="${VERSION}"

# Lint first
helm lint "$CHART_DIR" --strict

# Package with dependency update
helm package "$CHART_DIR" \
  --version "$VERSION" \
  --app-version "$APP_VERSION" \
  --dependency-update \
  --destination ./dist/

# Sign the package
helm package "$CHART_DIR" \
  --version "$VERSION" \
  --sign --key "release-key" \
  --passphrase-file /secrets/passphrase.txt \
  --destination ./dist/

# Upload to chart repository (e.g., ChartMuseum, Harbor, or OCI)
curl --data-binary "@dist/mychart-${VERSION}.tgz" \
  https://charts.example.com/api/charts
```

### Exam Scenario

The Helm Certified Associate exam may ask:
- How to override chart version at package time (answer: `--version`)
- How to produce a signed chart (answer: `--sign` and `--key`)
- The difference between `version` and `appVersion` (chart version vs. application version)

### Common Mistakes

- Forgetting `--dependency-update` when dependencies have changed, leading to a package with
  stale subcharts.
- Overriding `--version` but not updating `Chart.yaml`, causing drift between the source and
  the package.
- Attempting to sign without a PGP key configured — `gpg` must be installed and the key
  available.

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: found in Chart.yaml, but missing in charts/` | Run `helm dependency update` first, or use `--dependency-update` |
| `could not find key` | Verify key name with `gpg --list-secret-keys` and ensure `--keyring` path is correct |
| `passphrase file does not exist` | Verify the file path; ensure CI secrets are mounted correctly |
| Chart version already exists in destination | Use `--version` to bump, or clean the destination directory |

### Related Commands

- `helm lint` — validate the chart before packaging
- `helm dependency update` — download chart dependencies
- `helm push` — push the package to an OCI registry
- `helm verify` — verify a signed chart's provenance
- `helm pull` — download a packaged chart

---

## 7. helm lint

### Purpose

Validates a chart's structure, `Chart.yaml` metadata, and template rendering. It catches
syntax errors, missing required fields, undefined template variables, and violations of best
practices. Linting should always be run before packaging or installing a chart.

### Syntax

```
helm lint PATH [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--strict` | bool | `false` | Treat warnings as errors. If any lint rule produces a warning, `helm lint` exits with a non-zero code. Use this in CI to enforce quality. |
| `--quiet`, `-q` | bool | `false` | Print only error and warning messages. Suppress the "=> Linting chart" and "=> Chart OK" banner lines. |
| `--with-subcharts` | bool | `false` | Also lint subcharts (dependencies in `charts/`). By default, only the parent chart is linted. |
| `--set` | []string | `[]` | Set values on the command line for template rendering validation. Multiple `--set` flags can be used. Helps catch template errors that only manifest with specific values. |
| `--values`, `-f` | []string | `[]` | Specify YAML values files for template rendering validation. Like `--set`, helps test templates with realistic values. |
| `--namespace`, `-n` | string | `"default"` | Namespace scope for linting. Some templates use `.Release.Namespace`, and this flag sets it for validation. |

### Real-World Examples

```bash
# Basic lint
helm lint ./mychart

# Output (passing):
# ==> Linting ./mychart
# 1 chart(s) linted, 0 chart(s) failed
```

```bash
# Lint with warnings treated as errors (CI mode)
helm lint ./mychart --strict

# Quiet mode (for scripting)
helm lint ./mychart --quiet

# Lint with specific values to catch conditional template issues
helm lint ./mychart --values ./values-prod.yaml

# Lint with --set to test feature flags
helm lint ./mychart --set ingress.enabled=true --set autoscaling.enabled=true

# Lint including subcharts
helm lint ./mychart --with-subcharts

# Combine multiple options
helm lint ./mychart \
  --strict \
  --namespace production \
  --values ./values-prod.yaml \
  --set image.tag=v2.0.0
```

#### Output with Warnings (Example)

```
==> Linting ./mychart
[INFO] Chart.yaml: icon is recommended
[WARNING] templates/deployment.yaml: containers[0].resources.limits: 
  resource limits are recommended
[ERROR] templates/service.yaml: could not parse template: ...
1 chart(s) linted, 1 chart(s) failed
```

### Production Scenario

In CI/CD, linting is a gate before packaging or deployment:

```bash
#!/bin/bash
# Lint with all value files used in different environments
for env in dev staging prod; do
  echo "Linting for environment: $env"
  helm lint ./mychart \
    --strict \
    --with-subcharts \
    --values "./values-${env}.yaml" \
    --namespace "$env" || exit 1
done
```

### Exam Scenario

CKA/CKAD: If asked to validate a chart before installation, use `helm lint --strict`.
The exam may present a chart with intentional errors; `helm lint` helps identify them.

### Common Mistakes

- Running `helm lint` without `--strict` in CI — warnings are silently ignored.
- Linting only the parent chart when subcharts have been modified; use `--with-subcharts`.
- Forgetting that `helm lint` does NOT resolve remote dependencies (use
  `helm dependency update` first).
- Not providing environment-specific `--values` or `--set`, missing template errors that
  only appear with certain value combinations.

### Troubleshooting

| Problem | Solution |
|---|---|
| Lint passes but `helm install` fails | Some errors only manifest at runtime (e.g., Kubernetes API validation). Test with `helm template --dry-run` as well. |
| `could not find template` | A template references a named template that is missing; check `_helpers.tpl` |
| `nil pointer evaluating interface` | A template references a value key that does not exist and has no default; add a default in `values.yaml` or use `default` function in the template |

### Related Commands

- `helm template` — render templates locally for debugging
- `helm install --dry-run` — server-side dry run against the Kubernetes API
- `helm package` — package after linting

---

## 8. helm install

### Purpose

Installs a chart into a Kubernetes cluster, creating a new release. This is the primary
command for deploying applications with Helm. On first run, it creates a release record
(Secret by default) and instantiates all Kubernetes resources defined by the chart.

### Syntax

```
helm install [NAME] [CHART] [flags]
```

Arguments:
- `NAME`: The release name. If omitted, `--generate-name` must be used.
- `CHART`: Chart reference — can be a path (`./mychart`), a repo reference
  (`bitnami/nginx`), an OCI reference (`oci://registry/ns/chart`), or a URL (`.tgz` file).

### Flags (Complete)

#### General

| Flag | Type | Default | Description |
|---|---|---|---|
| `--atomic` | bool | `false` | If set, the release is automatically rolled back on failure. Implies `--wait`. The release is deleted and the previous version restored. |
| `--create-namespace` | bool | `false` | Create the target namespace if it does not exist. Without this, Helm returns an error if the namespace is missing. |
| `--dependency-update`, `-u` | bool | `false` | Run `helm dependency update` before installing. Downloads declared dependencies. Required when installing from an unpacked chart directory with unmet dependencies. |
| `--description` | string | `""` | Custom descriptive text added to the release record. Visible in `helm list` and `helm history`. Useful for audit trails. |
| `--devel` | bool | `false` | Include development versions (pre-release, alpha, beta, rc) when resolving chart versions from a repository. By default, only stable versions are considered. |
| `--dry-run` | bool | `false` | Simulate the install. Renders templates; submits to Kubernetes API server for validation; does NOT create resources. Outputs the rendered manifests. Used for pre-flight checks. |
| `--enable-dns` | bool | `false` | Enable DNS lookups during template rendering. Required when templates use `lookup` with DNS-related functions. Rarely used. |
| `--force` | bool | `false` | Force resource updates via delete + recreate when install conflicts with existing resources. DRAMATIC: can cause data loss. Generally NOT recommended for install; more relevant for `helm upgrade`. |
| `--generate-name` | bool | `false` | Automatically generate a release name. The chart name is prefixed followed by a random suffix (e.g., `nginx-1687539210`). Mutually exclusive with providing an explicit name. |
| `--history-max` | int | `10` | Maximum number of revisions saved per release. When exceeded, older revisions are garbage-collected. Limits Secret/ConfigMap usage in the cluster. |
| `--name-template` | string | `""` | Go template for generating the release name when `--generate-name` is used. Variables: `.Release.Name`, `.Release.Namespace`, `.Chart`, `.Files`, `.Capabilities`. |
| `--no-hooks` | bool | `false` | Skip running chart hooks (pre-install, post-install, etc.). Useful for debugging or when hooks are known to cause issues. |
| `--output`, `-o` | string | `"table"` | Output format for the install summary. Values: `table`, `json`, `yaml`. |
| `--post-renderer` | string | `""` | Path to an executable that post-processes rendered manifests. The executable receives raw YAML on stdin and must output processed YAML on stdout. Used with Kustomize, for example. |
| `--render-subchart-notes` | bool | `false` | If set, also render and display NOTES.txt from subcharts after install. By default, only the parent chart's NOTES.txt is shown. |
| `--replace` | bool | `false` | Reuse the given release name even if it was previously deleted. Without this, reusing a deleted name returns an error. |
| `--skip-crds` | bool | `false` | Skip installing CRDs defined in the chart's `crds/` directory. WARNING: If CRDs are skipped and the chart creates CR instances, install will fail. |
| `--timeout` | duration | `5m0s` | Maximum time to wait for Kubernetes operations to complete (e.g., `--wait` for resources to become ready). Format: Go duration (`5m`, `10m30s`, `1h`). |
| `--wait` | bool | `false` | Wait for all resources to be in a ready state before marking the release as successful. Checks Deployments, StatefulSets, DaemonSets, etc. |
| `--wait-for-jobs` | bool | `false` | When used with `--wait`, also wait for Jobs to complete successfully. Without this, Jobs are ignored during the wait phase. |

#### Value Overrides

| Flag | Type | Default | Description |
|---|---|---|---|
| `--set` | []string | `[]` | Set values via command line. Format: `key1=val1,key2=val2`. Supports dot notation: `image.tag=v2`. Can be specified multiple times. |
| `--set-file` | []string | `[]` | Set values from file contents. Format: `key=path`. The file content becomes the value. Useful for large multi-line values like certificates. |
| `--set-json` | []string | `[]` | Set JSON values. Format: `key='{"nested":"value"}'`. The value is parsed as JSON before injection, enabling lists and maps. |
| `--set-string` | []string | `[]` | Set values as strings, preventing Helm from interpreting numeric values. E.g., `--set-string replicas="3"` ensures `"3"` is a string, not `3` (int). |
| `--values`, `-f` | []string | `[]` | YAML values file(s). Can be specified multiple times. Later files override earlier ones. Files are merged in order. |

#### Connection & Security

| Flag | Type | Default | Description |
|---|---|---|---|
| `--ca-file` | string | `""` | Path to CA certificate for TLS verification of the chart repository server or Kubernetes API server. |
| `--cert-file` | string | `""` | Path to client certificate for TLS authentication to the chart repository server. |
| `--key-file` | string | `""` | Path to client certificate private key for TLS authentication to the chart repository server. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS certificate verification for chart downloads. NOT for production. |
| `--keyring` | string | `"~/.gnupg/secring.gpg"` | Path to PGP keyring for verifying signed charts. Used with `--verify`. |
| `--pass-credentials` | bool | `false` | Pass HTTP Basic Auth credentials to all domains contacted during the install (chart repo, Kubernetes API, etc.). Use cautiously. |
| `--password` | string | `""` | Chart repository password for HTTP Basic Auth. Must be used with `--username`. |
| `--username` | string | `""` | Chart repository username for HTTP Basic Auth. Must be used with `--password`. |
| `--verify` | bool | `false` | Verify the chart's provenance file (`.prov`) before installing. Requires the signer's public key in the configured keyring. |
| `--version` | string | `""` | Specific chart version to install. If omitted, the latest stable version is used. Format: semver constraint (e.g., `>=1.2.0`, `~1.2.3`). |
| `--repo` | string | `""` | Chart repository URL. When used with a chart name (not path/URL), this specifies the repository without requiring `helm repo add`. |

### Real-World Examples

```bash
# Basic install
helm install my-release bitnami/nginx

# Output:
# NAME: my-release
# LAST DEPLOYED: Sat Jul 11 12:00:00 2026
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# ...
```

```bash
# Install with custom values file
helm install my-app ./mychart --values ./values-prod.yaml

# Install with multiple values files (merged in order)
helm install my-app ./mychart \
  -f values-base.yaml \
  -f values-prod.yaml \
  -f values-us-east.yaml

# Install with inline value overrides
helm install my-app bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=LoadBalancer \
  --set-string image.tag="v1.2.3"

# Install with JSON values
helm install my-app bitnami/nginx \
  --set-json 'resources.limits={"cpu":"500m","memory":"512Mi"}'

# Install from file content (e.g., TLS certificate)
helm install my-app ./mychart \
  --set-file tls.crt=./certs/server.crt \
  --set-file tls.key=./certs/server.key

# Generate name (auto-named release)
helm install --generate-name bitnami/nginx

# Custom name template
helm install --generate-name --name-template "{{ .Release.Name }}-{{ .Release.Namespace }}" bitnami/nginx

# Atomic install (rollback on failure)
helm install my-app ./mychart --atomic --timeout 10m

# Dry run (validate without installing)
helm install my-app ./mychart --dry-run --debug | less

# Install into a specific namespace, creating it if needed
helm install my-app bitnami/nginx \
  --namespace my-namespace \
  --create-namespace

# Install verified (signed chart)
helm install my-app bitnami/nginx --verify --keyring ~/.gnupg/secring.gpg

# Install a specific version
helm install my-app bitnami/nginx --version 15.2.0

# Install with devel versions available
helm install my-app bitnami/nginx --devel --version ">=16.0.0-alpha"

# Install from OCI registry
helm install my-app oci://registry.example.com/charts/nginx --version 1.0.0

# Install from a tarball URL
helm install my-app https://charts.example.com/nginx-15.2.0.tgz

# Wait for resources and skip hooks
helm install my-app ./mychart --wait --no-hooks --timeout 15m

# Use post-renderer (e.g., Kustomize)
helm install my-app ./mychart --post-renderer ./kustomize-wrapper.sh

# Replace a previously deleted release
helm install my-app ./mychart --replace

# Install with max history of 5 revisions
helm install my-app ./mychart --history-max 5

# Full production install
helm install my-prod-app ./mychart \
  --namespace production \
  --create-namespace \
  --values ./values-prod.yaml \
  --set image.tag=v3.2.1 \
  --atomic \
  --wait \
  --wait-for-jobs \
  --timeout 15m \
  --description "Production deployment v3.2.1 by CI pipeline #1234" \
  --history-max 10
```

### Production Scenario

A production deployment pipeline with validation and rollback:

```bash
#!/bin/bash
RELEASE="customer-api"
CHART="./charts/customer-api"
NAMESPACE="production"
VALUES="./values-prod.yaml"

# Pre-flight: lint + dry run
helm lint "$CHART" --strict --values "$VALUES" || exit 1
helm install "$RELEASE" "$CHART" \
  --namespace "$NAMESPACE" \
  --values "$VALUES" \
  --dry-run | kubectl apply --dry-run=client -f - || exit 1

# Install with full safety net
helm install "$RELEASE" "$CHART" \
  --namespace "$NAMESPACE" \
  --create-namespace \
  --values "$VALUES" \
  --atomic \
  --wait \
  --wait-for-jobs \
  --timeout 15m \
  --description "Deployed by CI/CD pipeline #${CI_PIPELINE_ID}"
```

### Exam Scenario

CKA/CKAD frequently involves:
- Installing a chart with specific values: `helm install my-nginx bitnami/nginx --set service.type=NodePort`
- Installing into a specific namespace: `helm install my-app ./mychart -n exam-ns --create-namespace`
- Using `--dry-run` to verify manifests without deploying
- Using `--atomic` for safe rollback

### Common Mistakes

- Omitting `--create-namespace` when the target namespace doesn't exist.
- Using `--set` for complex nested values — use `--values` or `--set-json` instead.
- Forgetting that `--set` values override entire sections; to add one key, you must provide
  the full nested path (e.g., `--set resources.limits.cpu=500m`).
- Relying on `--wait` without `--timeout`; the default 5-minute timeout may be insufficient.
- Using `--force` casually — it can delete and recreate resources, causing downtime.
- Not using `--atomic` for production deployments; a failed install leaves dangling resources.

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: cannot reuse a name that is still in use` | The release name exists. Use `helm list` to check. Use `helm uninstall <name>` or a different name. |
| `Error: namespaces "x" not found` | Add `--create-namespace` or create the namespace manually with `kubectl create ns x`. |
| `Error: chart "x" version "y" not found` | Run `helm repo update` to refresh the index. Check the version exists. |
| Install hangs on `--wait` | Resources are not becoming ready. Check pod status with `kubectl get pods -n <ns>`. Increase `--timeout`. |
| Template rendering error | Run `helm template --debug` to see the exact error and the partially rendered template. |
| `Error: cannot re-use a name that is still in use` | Use `--replace` if the release was soft-deleted. |

### Related Commands

- `helm upgrade` — modify an existing release
- `helm uninstall` — remove a release
- `helm list` — view installed releases
- `helm template` — render templates locally
- `helm rollback` — revert to a previous revision
- `helm test` — run chart tests

---

## 9. helm upgrade

### Purpose

Upgrades an existing Helm release to a new version of the chart or with new configuration
values. It is the primary command for updating deployed applications.

### Syntax

```
helm upgrade [RELEASE] [CHART] [flags]
```

### Flags

`helm upgrade` shares most flags with `helm install`. All flags from Section 8 apply, plus
the following upgrade-specific flags:

| Flag | Type | Default | Description |
|---|---|---|---|
| `--cleanup-on-fail` | bool | `false` | Delete newly created resources if the upgrade fails. Without this, failed upgrades may leave orphaned resources. |
| `--install`, `-i` | bool | `false` | If the release does not exist, run `helm install` instead. Allows idempotent deploy commands: a single `helm upgrade --install` works for both first-time and subsequent deployments. |
| `--reset-then-reuse-values` | bool | `false` | Reset values to chart defaults, then merge the last release's overridden values on top. This is the Helm v2 behaviour. Useful when migrating from Helm v2 or when you want to carry forward user values but incorporate new chart defaults. |
| `--reset-values` | bool | `false` | Reset ALL values to the chart's built-in defaults. Any previously set `--set` or `--values` from prior installs/upgrades are discarded. Use with caution — you must provide ALL needed overrides again. |
| `--reuse-values` | bool | `false` | Reuse the last release's values exactly as they were. Any new defaults in the updated chart are ignored in favour of the previously set values. Use when you only want to upgrade the chart templates, not the values. |
| `--max-history` | int | `10` | (Deprecated alias for `--history-max`) |
| `--history-max` | int | `10` | Maximum number of revisions to retain. |

### Real-World Examples

```bash
# Basic upgrade
helm upgrade my-release bitnami/nginx

# Upgrade with new values file
helm upgrade my-release bitnami/nginx --values ./values-v2.yaml

# Upgrade to a specific chart version
helm upgrade my-release bitnami/nginx --version 16.0.0

# Idempotent deploy (install if not exists, upgrade if exists)
helm upgrade --install my-app ./mychart --values ./values-prod.yaml

# Upgrade reusing previous values (only change chart templates)
helm upgrade my-release ./mychart --reuse-values

# Reset all values and provide new ones
helm upgrade my-release ./mychart --reset-values --values ./values-v2.yaml

# Upgrade with atomic rollback on failure
helm upgrade my-release ./mychart \
  --atomic \
  --cleanup-on-fail \
  --timeout 10m

# Upgrade with forced resource recreation (dangerous, use sparingly)
helm upgrade my-release ./mychart --force

# Dry run to preview changes
helm upgrade my-release ./mychart --dry-run --debug

# Upgrade with install (CI/CD idempotency)
helm upgrade --install "$SERVICE_NAME" "./charts/$SERVICE_NAME" \
  --namespace "$NAMESPACE" \
  --values "./values-${ENV}.yaml" \
  --set image.tag="${IMAGE_TAG}" \
  --atomic \
  --wait \
  --timeout 15m \
  --description "Deployed commit ${GIT_COMMIT} by ${CI_USER}"
```

### Value Precedence During Upgrade

The `--reuse-values`, `--reset-values`, and `--reset-then-reuse-values` flags control how
previous values interact with the new chart version:

| Flag Behaviour | Chart Defaults | Previous User Values | New `--values` / `--set` |
|---|---|---|---|
| None (default) | Yes (base) | Ignored | Yes (most recent win) |
| `--reuse-values` | Ignored | Yes (base) | Yes (override reuse) |
| `--reset-values` | Yes (base) | Ignored | Yes (override defaults) |
| `--reset-then-reuse-values` | Yes (base) | Yes (merged on top) | Not recommended together |

### Production Scenario

```bash
#!/bin/bash
# Production zero-downtime upgrade
RELEASE="payment-service"
CHART="./charts/payment-service"
NAMESPACE="production"

# Check current revision
CURRENT_REV=$(helm history "$RELEASE" -n "$NAMESPACE" --max 1 -o json | jq -r '.[0].revision')
echo "Current revision: $CURRENT_REV"

# Perform canary check first
helm upgrade "$RELEASE" "$CHART" \
  --namespace "$NAMESPACE" \
  --values ./values-prod.yaml \
  --dry-run | kubectl diff -f -

# If user confirms:
helm upgrade "$RELEASE" "$CHART" \
  --namespace "$NAMESPACE" \
  --values ./values-prod.yaml \
  --atomic \
  --cleanup-on-fail \
  --wait \
  --wait-for-jobs \
  --timeout 15m \
  --history-max 20 \
  --description "Upgrade to v2.1.0 by deployer"

# Verify
helm test "$RELEASE" -n "$NAMESPACE"
```

### Exam Scenario

The CKA/CKAD may test:
- `--install` for idempotent deployment: `helm upgrade --install my-app ./mychart`
- `--reset-values` to discard old overrides: `helm upgrade my-app ./mychart --reset-values --set key=val`
- `--reuse-values` to keep existing configuration: `helm upgrade my-app ./mychart --reuse-values`

### Common Mistakes

- Using `--reuse-values` and `--reset-values` together (they conflict).
- Forgetting `--install` when running a deploy pipeline for the first time, causing failure.
- Not using `--atomic` — a partially failed upgrade leaves the release in a broken state.
- Upgrading without `--cleanup-on-fail` — orphaned resources accumulate (e.g., old Jobs,
  unused PVCs).
- Running `helm upgrade` without new values and expecting new chart defaults to apply
  (by default, previous user values override new chart defaults).

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: UPGRADE FAILED: has no deployed releases` | Use `--install` or run `helm install` first |
| Upgrade succeeds but pods use old image | Verify `--reuse-values` is not unintentionally set; new `--set image.tag` may not override reused values without explicit path |
| `Error: another operation (install/upgrade/rollback) is in progress` | A previous operation is pending. Check with `helm history`. Wait or use `helm rollback` to resolve. |
| Resource quota exceeded after upgrade | The new chart version may create additional resources; check `--dry-run` first |

### Related Commands

- `helm install` — first-time deployment
- `helm rollback` — revert to a previous revision
- `helm history` — view revision history
- `helm diff` (plugin) — show changes between revisions
- `helm test` — verify after upgrade

---

## 10. helm rollback

### Purpose

Rolls back a release to a previous revision. Each `helm install` and `helm upgrade` creates
a numbered revision. Rollback restores the Kubernetes resources to the state of the specified
revision using the values and chart version from that revision.

### Syntax

```
helm rollback RELEASE [REVISION] [flags]
```

If `REVISION` is omitted, Helm rolls back to the previous revision (current - 1).

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--cleanup-on-fail` | bool | `false` | Delete newly created resources if the rollback fails. Prevents orphaned resources. |
| `--dry-run` | bool | `false` | Simulate the rollback. Renders manifests and submits for API validation without making changes. |
| `--force` | bool | `false` | Force resource updates through a delete-and-recreate strategy. Can cause downtime; use only when immutable fields have changed. |
| `--history-max` | int | `10` | Maximum revisions to retain. Older revisions are garbage-collected. |
| `--no-hooks` | bool | `false` | Skip running rollback hooks defined in the chart. |
| `--recreate-pods` | bool | `false` | Restart pods for resources where the pod template has NOT changed. Performs a rolling restart. Implies `--wait`. |
| `--timeout` | duration | `5m0s` | Maximum wait time for the rollback to complete when used with `--wait`. |
| `--wait` | bool | `false` | Wait for all resources to become ready before marking the rollback as successful. |
| `--wait-for-jobs` | bool | `false` | With `--wait`, also wait for Jobs to complete successfully. |

### Real-World Examples

```bash
# Rollback to the previous revision
helm rollback my-release

# Rollback to a specific revision
helm rollback my-release 3

# Dry-run rollback to preview changes
helm rollback my-release 2 --dry-run

# Rollback with safety net
helm rollback my-release 3 \
  --cleanup-on-fail \
  --wait \
  --timeout 10m

# Force resource recreation (immutable field changes)
helm rollback my-release 2 --force --recreate-pods

# Rollback skipping hooks (if hooks cause failure)
helm rollback my-release 2 --no-hooks

# Rollback and keep more history
helm rollback my-release 2 --history-max 20
```

### Production Scenario

Rollback automation in response to alerts:

```bash
#!/bin/bash
RELEASE="critical-service"
NAMESPACE="production"
ROLLBACK_TO=$(( $(helm history "$RELEASE" -n "$NAMESPACE" --max 1 -o json | jq -r '.[0].revision') - 1 ))

echo "ALERT: Rolling back $RELEASE to revision $ROLLBACK_TO"

helm rollback "$RELEASE" "$ROLLBACK_TO" \
  --namespace "$NAMESPACE" \
  --cleanup-on-fail \
  --wait \
  --wait-for-jobs \
  --timeout 10m

# Verify
helm test "$RELEASE" -n "$NAMESPACE"
```

### Exam Scenario

CKAD/CKA: you may be asked to roll back a deployment that was just upgraded to a broken
state. Steps:
```bash
# Check history
helm history my-release

# Rollback to the version before the broken one
helm rollback my-release 2 --wait
```

### Common Mistakes

- Attempting to roll back to a revision that has been garbage-collected (exceeded
  `--history-max`). Increase `--history-max` for important releases.
- Not using `--wait` — rollback returns immediately but pods may still be starting.
- Expecting rollback to restore persistent data — Helm only manages Kubernetes resources,
  not database content.
- Using `--force` routinely — it causes unnecessary pod restarts.

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: revision "5" not found` | The revision has been garbage-collected. Check available revisions with `helm history`. |
| Rollback succeeds but old pods remain | Use `--recreate-pods` if the pod spec changed in ways that don't trigger a rolling update |
| `another operation is in progress` | A pending `install`, `upgrade`, or another `rollback` is holding the lock. Wait or delete the pending release Secret. |

### Related Commands

- `helm history` — list all revisions
- `helm upgrade` — move forward to a new version
- `helm get values --revision N` — inspect values at a specific revision
- `helm status` — check current release state

---

## 11. helm uninstall

### Purpose

Removes a Helm release from the cluster. By default, it deletes all Kubernetes resources
associated with the release AND removes the release history. This is the opposite of
`helm install`.

### Syntax

```
helm uninstall RELEASE_NAME [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--description` | string | `""` | Add a custom description to the uninstall operation. Recorded in the release history. |
| `--dry-run` | bool | `false` | Simulate the uninstall. Shows which resources would be deleted without actually removing them. Output format is similar to `kubectl delete --dry-run`. |
| `--keep-history` | bool | `false` | Keep the release history records even after removing resources. The release name remains reserved and cannot be reused. Useful for auditing. |
| `--no-hooks` | bool | `false` | Skip running uninstall hooks (pre-delete, post-delete). The hooks' resources are still deleted. |
| `--timeout` | duration | `5m0s` | Maximum time to wait for the uninstall to complete. |
| `--wait` | bool | `false` | Wait for all resources to be fully deleted before exiting. Without this, Helm exits immediately after issuing the delete commands. |

### Real-World Examples

```bash
# Basic uninstall
helm uninstall my-release

# Output:
# release "my-release" uninstalled
```

```bash
# Dry run to see what will be deleted
helm uninstall my-release --dry-run

# Uninstall but keep history for auditing
helm uninstall my-release --keep-history

# Wait for full cleanup
helm uninstall my-release --wait --timeout 10m

# Uninstall skipping hooks
helm uninstall my-release --no-hooks

# Uninstall with audit description
helm uninstall my-release --description "Decommissioned per change request CR-45231"

# In CI/CD cleanup
helm uninstall "$RELEASE" -n "$NAMESPACE" --wait --timeout 5m || true
```

### Production Scenario

Orderly decommissioning of a service:

```bash
#!/bin/bash
RELEASE="old-service"
NAMESPACE="production"

# 1. Check what will be deleted
helm uninstall "$RELEASE" -n "$NAMESPACE" --dry-run

# 2. Back up release values for auditing
helm get values "$RELEASE" -n "$NAMESPACE" --all > "${RELEASE}-values-backup.yaml"

# 3. Uninstall with safety
helm uninstall "$RELEASE" \
  -n "$NAMESPACE" \
  --keep-history \
  --wait \
  --timeout 10m \
  --description "Service decommissioned; replaced by new-service"

# 4. Verify no orphaned resources
kubectl get all -n "$NAMESPACE" -l "app.kubernetes.io/instance=$RELEASE"
```

### Exam Scenario

CKA/CKAD: straightforward. `helm uninstall my-release -n my-namespace`. Rarely tested beyond
the basic usage.

### Common Mistakes

- Forgetting `--keep-history` when auditing is required — once history is deleted, it's
  irrecoverable (unless you have etcd backups).
- Assuming `helm uninstall` also deletes the namespace — it does NOT.
- Uninstalling without checking for dependent releases or shared resources.

### Troubleshooting

| Problem | Solution |
|---|---|
| Resources stuck in Terminating state | The resource may have finalizers. Check with `kubectl describe`. Remove finalizers manually if safe. |
| Uninstall hangs | Increase `--timeout`. A pre-delete hook may be stuck. Use `--no-hooks` cautiously. |
| `Error: release: not found` | The release may already be uninstalled or the namespace is wrong. Check with `helm list -A`. |

### Related Commands

- `helm install` — create a release
- `helm list` — find release names
- `helm history` — view release history before and after uninstall

---

## 12. helm list

### Purpose

Lists all Helm releases in a namespace or across all namespaces. Provides filtering by
status, date, name, and labels. Essential for release inventory management.

### Syntax

```
helm list [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--all`, `-a` | bool | `false` | Show all releases regardless of status. Without this, only `deployed` releases are shown. |
| `--all-namespaces`, `-A` | bool | `false` | List releases across all Kubernetes namespaces. |
| `--date`, `-d` | bool | `false` | Sort releases by last updated date (newest first). Default sorting is by release name (alphabetical). |
| `--deployed` | bool | `false` | Show only deployed releases. Redundant without `--all`, but useful as a filter. |
| `--failed` | bool | `false` | Show only failed releases. Must be combined with `--all`. |
| `--filter`, `-f` | string | `""` | Filter releases by name using a regular expression. E.g., `--filter '^nginx'` shows only releases starting with "nginx". |
| `--max`, `-m` | int | `256` | Maximum number of releases to return. Default is high to prevent accidental truncation. |
| `--namespace`, `-n` | string | `""` | Limit listing to a specific namespace. Defaults to the namespace in the current kubeconfig context or `HELM_NAMESPACE`. |
| `--offset`, `-o` | int | `0` | Offset for pagination. Skips the first N releases. Used with `--max` for paginated queries. |
| `--pending` | bool | `false` | Show only pending releases (install/upgrade/rollback in progress). Must be combined with `--all`. |
| `--reverse`, `-r` | bool | `false` | Reverse the sort order. |
| `--selector`, `-l` | string | `""` | Filter releases by Kubernetes label selector. E.g., `--selector 'env=prod,team=platform'`. Matches labels on the release Secret. |
| `--short`, `-q` | bool | `false` | Short output format: prints only release names (one per line). Useful for scripting. |
| `--superseded` | bool | `false` | Show only superseded releases (replaced by a newer revision). Must be combined with `--all`. |
| `--time-format` | string | `"2006-01-02 15:04:05.999999 -0700 MST"` | Go time format string for the `UPDATED` column. Accepts any valid Go `time.Format` layout. |
| `--uninstalled` | bool | `false` | Show only uninstalled releases (soft-deleted with `--keep-history`). Must be combined with `--all`. |
| `--uninstalling` | bool | `false` | Show only releases currently being uninstalled. Must be combined with `--all`. |

#### Output Columns

| Column | Description |
|---|---|
| `NAME` | Release name |
| `NAMESPACE` | Namespace the release is installed in |
| `REVISION` | Current revision number |
| `UPDATED` | Timestamp of the last action (install, upgrade, rollback) |
| `STATUS` | Release status: `deployed`, `failed`, `pending-install`, `pending-upgrade`, `pending-rollback`, `superseded`, `uninstalling`, `uninstalled`, `unknown` |
| `CHART` | Chart name and version (e.g., `nginx-15.2.0`) |
| `APP VERSION` | Application version from the chart's `appVersion` field |

### Real-World Examples

```bash
# List deployed releases in current namespace
helm list

# Output:
# NAME    NAMESPACE REVISION UPDATED                  STATUS   CHART         APP VERSION
# my-app  default   3        2026-07-11 10:30:00...   deployed nginx-15.2.0 1.27.0
```

```bash
# List all releases across all namespaces
helm list -A

# List all releases including failed/uninstalled
helm list --all

# Filter releases by name regex
helm list --filter '^prometheus|^grafana'

# Filter by label selector
helm list --selector 'app.kubernetes.io/managed-by=Helm,env=production'

# Sort by date, newest first
helm list --date

# Sort by date, oldest first
helm list --date --reverse

# Short output (names only)
helm list -q

# List with custom timestamp format (Unix timestamp)
helm list --time-format "2006-01-02"

# Paginated listing
helm list --max 50 --offset 50

# Show only failed releases
helm list --all --failed

# Pipeline: delete all releases in a namespace
helm list -n my-ns -q | xargs helm uninstall -n my-ns
```

### Production Scenario

Release inventory audit script:

```bash
#!/bin/bash
echo "=== All Helm Releases ==="
helm list -A --all

echo ""
echo "=== Failed or Pending Releases ==="
helm list -A --all --failed
helm list -A --all --pending

echo ""
echo "=== Releases Using Outdated Charts ==="
helm list -A -o json | jq -r '.[] | "\(.name)\t\(.namespace)\t\(.chart)"'
```

### Exam Scenario

CKA/CKAD:
- List releases in all namespaces: `helm list -A`
- Find a specific release: `helm list -n my-ns --filter 'my-release'`
- Count releases: `helm list -q | wc -l`

### Common Mistakes

- Expecting `helm list` to show all statuses by default — it only shows `deployed`. Use
  `--all` for comprehensive listing.
- Using `--filter` with full regex when a simple substring would suffice. The filter is a
  regex, so `.` matches any character; escape dots if needed.
- Forgetting that `--max` defaults to 256 — if you have more releases, they may be truncated.

### Troubleshooting

| Problem | Solution |
|---|---|
| No releases shown but you know they exist | Add `--all` to see non-deployed releases, or check you're in the correct namespace |
| `helm list -A` shows nothing | Check `kubectl config current-context` — you may be on the wrong cluster |
| Filter returns nothing | Verify regex syntax; use `--filter 'substring'` for simple matching |

### Related Commands

- `helm history` — revision history for a single release
- `helm status` — detailed status of a single release
- `helm uninstall` — remove a release

---

## 13. helm history

### Purpose

Shows the revision history of a specific release. Each install, upgrade, or rollback creates
a numbered revision. This command lists all stored revisions with their status, chart version,
description, and timestamp.

### Syntax

```
helm history RELEASE_NAME [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--max`, `-m` | int | `256` | Maximum number of revisions to show. The default is generous; use a smaller number for recent history. |
| `--output`, `-o` | string | `"table"` | Output format: `table`, `json`, `yaml`. JSON/YAML output is useful for scripting and auditing. |

#### Output Columns

| Column | Description |
|---|---|
| `REVISION` | Sequential revision number |
| `UPDATED` | Timestamp of the action |
| `STATUS` | `deployed`, `superseded`, `failed`, `pending-install`, `pending-upgrade`, `pending-rollback`, `unknown` |
| `CHART` | Chart name and version used for this revision |
| `APP VERSION` | `appVersion` from the chart at this revision |
| `DESCRIPTION` | Description string (defaults to action type, e.g., "Install complete", "Upgrade complete") |

### Real-World Examples

```bash
# Basic history
helm history my-release

# Output:
# REVISION UPDATED                  STATUS      CHART         APP VERSION DESCRIPTION
# 1        2026-07-10 09:00:00...   superseded  mychart-1.0.0 1.0.0       Install complete
# 2        2026-07-10 14:00:00...   superseded  mychart-1.1.0 1.1.0       Upgrade complete
# 3        2026-07-11 08:00:00...   deployed    mychart-1.2.0 1.2.0       Upgrade complete
```

```bash
# Show only the last 5 revisions
helm history my-release --max 5

# JSON output for scripting
helm history my-release -o json

# YAML output for readability
helm history my-release -o yaml

# Get the current deployed revision number
helm history my-release --max 1 -o json | jq -r '.[0].revision'

# Find the last successful revision
helm history my-release -o json | jq '[.[] | select(.status == "deployed")] | last | .revision'

# Identify failed revisions
helm history my-release -o json | jq '.[] | select(.status == "failed")'
```

### Production Scenario

Auditing deployment history for compliance:

```bash
#!/bin/bash
RELEASE="payment-service"
NAMESPACE="production"

# Full history in machine-readable format
helm history "$RELEASE" -n "$NAMESPACE" -o json | jq '[.[] | {
  revision: .revision,
  updated: .updated,
  status: .status,
  chart: .chart,
  app_version: .app_version,
  description: .description
}]' > "${RELEASE}-audit-$(date +%Y%m%d).json"
```

### Exam Scenario

CKA/CKAD: you may need to check the history to find which revision to roll back to:
```bash
helm history my-release
helm rollback my-release <last-good-revision>
```

### Common Mistakes

- Expecting `helm history` to work for uninstalled releases without `--keep-history`. Once
  a release is fully uninstalled, the history is gone.
- Not using `--max` limit — very old releases with many revisions produce excessive output.

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: release: not found` | The release name is incorrect or the release has been uninstalled without `--keep-history` |
| History shows many `pending-*` statuses | Stuck operations. Investigate with `helm status` and resolve the pending operation |

### Related Commands

- `helm status` — current state of a release
- `helm list` — list all releases
- `helm rollback` — revert to a previous revision
- `helm get values --revision N` — get values at a specific revision

---

## 14. helm status

### Purpose

Displays the current status of a named release. Shows the deployment state, last operation
timestamp, Kubernetes resources created, and any chart-supplied notes.

### Syntax

```
helm status RELEASE_NAME [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--revision` | int | `0` | Show the status of a specific revision. `0` means the latest revision. |
| `--show-desc` | bool | `false` | Show the release description (set via `--description` during install/upgrade). |
| `--show-resources` | bool | `false` | Display a table of all Kubernetes resources created by this release, including their kind, name, and status. |
| `--output`, `-o` | string | `"table"` | Output format for the resources table. Values: `table`, `json`, `yaml`. Only relevant with `--show-resources`. |

### Real-World Examples

```bash
# Basic status
helm status my-release

# Output:
# NAME: my-release
# LAST DEPLOYED: Sat Jul 11 12:00:00 2026
# NAMESPACE: default
# STATUS: deployed
# REVISION: 3
# TEST SUITE:     None
# NOTES:
# (chart NOTES.txt content here)
```

```bash
# Show resources associated with the release
helm status my-release --show-resources

# Output:
# NAME: my-release
# ...
# RESOURCES:
# ==> v1/Service
# NAME            TYPE       CLUSTER-IP   EXTERNAL-IP  PORT(S) AGE
# my-release-nginx ClusterIP  10.0.1.100   <none>       80/TCP  2d
# ==> v1/Deployment
# NAME            READY UP-TO-DATE AVAILABLE AGE
# my-release-nginx 1/1   1          1         2d
```

```bash
# Resource table in JSON for scripting
helm status my-release --show-resources -o json

# Show description
helm status my-release --show-desc

# Check a specific past revision
helm status my-release --revision 2
```

### Production Scenario

Health check in a monitoring script:

```bash
#!/bin/bash
check_release() {
  local release=$1
  local namespace=$2

  STATUS=$(helm status "$release" -n "$namespace" -o json 2>/dev/null | jq -r '.info.status')
  if [[ "$STATUS" != "deployed" ]]; then
    echo "ALERT: $release in $namespace is $STATUS"
    return 1
  fi
  echo "OK: $release is $STATUS"
}

check_release "payment-service" "production"
check_release "user-service" "production"
```

### Exam Scenario

CKA/CKAD: use `helm status` to verify a release was deployed correctly and to find the
service endpoint or access instructions from NOTES.txt.

### Common Mistakes

- Expecting `helm status` to show detailed pod status — it only shows the Helm release status.
  Use `kubectl` for pod-level detail.
- Not realising that a `deployed` status means the Helm operation completed, NOT that all pods
  are necessarily running and healthy.

### Troubleshooting

| Problem | Solution |
|---|---|
| `STATUS: pending-upgrade` | A previous upgrade is stuck. Check `helm history` and resolve. |
| `STATUS: failed` | The install/upgrade failed. Check resources with `--show-resources` and `kubectl describe`. |
| Notes section is empty | The chart's NOTEST.txt may be empty; this is normal for some charts. |

### Related Commands

- `helm list` — overview of all releases
- `helm history` — revision history
- `helm get` — retrieve detailed release information

---

## 15. helm test

### Purpose

Runs the test suite defined in a chart's `templates/tests/` directory. Tests are Kubernetes
Pods that run to completion and report success or failure. Used to validate that a deployed
release is functioning correctly.

### Syntax

```
helm test [RELEASE_NAME] [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--filter` | []string | `[]` | Run only tests matching this name regex. E.g., `--filter 'connection'` runs only test pods whose names contain "connection". Can be specified multiple times. |
| `--logs` | bool | `false` | Print test pod logs to stdout as they run. Useful for debugging test failures. The logs are streamed in real-time. |
| `--timeout` | duration | `5m0s` | Maximum time to wait for all test pods to complete. Tests running beyond this timeout are considered failed. |

### Real-World Examples

```bash
# Run all tests for a release
helm test my-release

# Output (success):
# NAME: my-release
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# REVISION: 3
# TEST SUITE:     my-release-test-connection
# Last Started:   Sat Jul 11 12:05:00 2026
# Last Completed: Sat Jul 11 12:05:05 2026
# Phase:          Succeeded
# NOTES:
# ...

# Output (failure):
# TEST SUITE:     my-release-test-connection
# ...
# Phase:          Failed
# Error:          ...
```

```bash
# Run with real-time logs
helm test my-release --logs

# Run a specific test by name filter
helm test my-release --filter 'connection'

# Run multiple specific tests
helm test my-release --filter 'connection' --filter 'health'

# Longer timeout for slow tests
helm test my-release --timeout 15m

# CI/CD validation
helm test "$RELEASE" -n "$NAMESPACE" --timeout 10m || {
  echo "Tests failed!"
  helm test "$RELEASE" -n "$NAMESPACE" --logs
  exit 1
}
```

### Production Scenario

Post-deployment validation gate:

```bash
#!/bin/bash
RELEASE="customer-api"
NAMESPACE="production"

helm upgrade --install "$RELEASE" ./chart -n "$NAMESPACE" --wait

echo "Running integration tests..."
if ! helm test "$RELEASE" -n "$NAMESPACE" --timeout 10m --logs; then
  echo "Tests failed! Rolling back..."
  helm rollback "$RELEASE" -n "$NAMESPACE" --wait
  exit 1
fi

echo "Tests passed. Deployment successful."
```

### Exam Scenario

CKAD/CKA: may ask you to run chart tests after installing a release:
```bash
helm install my-app ./mychart
helm test my-app
# Check test results
helm test my-app --logs
```

### Common Mistakes

- Expecting `helm test` to work immediately after `helm install` when pods are not yet ready.
  Test pods need the application to be fully running.
- Not using `--logs` when tests fail — without logs, you only see "Failed" with no details.
- Creating test pods that don't clean up after themselves (test pods persist after completion).
- Default 5-minute timeout being too short for tests that provision or migrate data.

### Troubleshooting

| Problem | Solution |
|---|---|
| Tests always in "Running" phase | Increase `--timeout`; the test may need more time |
| `Error: release has no test suite` | The chart's `templates/tests/` directory is missing or empty |
| Test pod fails with `ImagePullBackOff` | Ensure test images are accessible from the cluster |
| Logs not showing | Re-run with `--logs`; logs may have already been garbage-collected if the pod completed long ago |

### Related Commands

- `helm install` — deploy before testing
- `helm upgrade` — upgrade before re-testing
- `helm rollback` — revert if tests fail
- `kubectl logs` — inspect test pod logs manually

---

## 16. helm get

### Purpose

Retrieves information about a release. The `helm get` command has subcommands to fetch
specific types of release data: values, manifest, hooks, notes, metadata, or all combined.

### Subcommands

| Subcommand | Purpose |
|---|---|
| `values` | Get the values (user-supplied + computed) used for the release |
| `manifest` | Get the full rendered Kubernetes manifest for the release |
| `hooks` | Get the hooks defined by the chart |
| `notes` | Get the NOTES.txt content for the release |
| `metadata` | Get release metadata (name, namespace, chart, status, etc.) |
| `all` | Get all of the above in a single output |

### 16.1 helm get values

#### Syntax

```
helm get values RELEASE_NAME [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--all`, `-a` | bool | `false` | Show ALL values, including computed/default values, not just user-supplied overrides. Without this, only user-supplied values are shown. |
| `--output`, `-o` | string | `"table"` | Output format: `table`, `json`, `yaml`. |
| `--revision` | int | `0` | Retrieve values from a specific revision. `0` = latest. |

#### Real-World Examples

```bash
# Show user-supplied values
helm get values my-release

# Show all values (including defaults)
helm get values my-release --all

# Show values as YAML
helm get values my-release --all -o yaml

# Show values from a previous revision
helm get values my-release --revision 2

# Compare values between revisions
diff <(helm get values my-release --revision 2 -o yaml) \
     <(helm get values my-release --revision 3 -o yaml)
```

### 16.2 helm get manifest

#### Syntax

```
helm get manifest RELEASE_NAME [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--revision` | int | `0` | Get manifest from a specific revision. |

#### Real-World Examples

```bash
# Get the full rendered manifest
helm get manifest my-release

# Save manifest to a file for backup/audit
helm get manifest my-release > my-release-manifest.yaml

# Compare manifests across revisions
helm get manifest my-release --revision 2 > rev2.yaml
helm get manifest my-release --revision 3 > rev3.yaml
diff rev2.yaml rev3.yaml
```

### 16.3 helm get hooks

#### Syntax

```
helm get hooks RELEASE_NAME [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--revision` | int | `0` | Get hooks from a specific revision. |

#### Real-World Example

```bash
helm get hooks my-release
```

### 16.4 helm get notes

#### Syntax

```
helm get notes RELEASE_NAME [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--revision` | int | `0` | Get notes from a specific revision. |

#### Real-World Example

```bash
# Re-display install notes without reinstalling
helm get notes my-release
```

### 16.5 helm get metadata

#### Syntax

```
helm get metadata RELEASE_NAME [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--revision` | int | `0` | Get metadata from a specific revision. |

#### Real-World Example

```bash
helm get metadata my-release
# Output:
# NAME: my-release
# NAMESPACE: default
# STATUS: deployed
# ...
```

### 16.6 helm get all

#### Syntax

```
helm get all RELEASE_NAME [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--revision` | int | `0` | Retrieve all information from a specific revision. |
| `--template` | string | `""` | Go template to format the output. Use `{{ .Release.Name }}`, `{{ .Chart }}`, etc. |

#### Real-World Example

```bash
# Get everything about a release
helm get all my-release

# Get all data and parse with jq
helm get all my-release -o json | jq .

# Template output
helm get all my-release --template 'Release: {{ .Release.Name }}, Chart: {{ .Chart }}'
```

### Production Scenario

Backup release configuration before major changes:

```bash
#!/bin/bash
RELEASE="critical-app"
NAMESPACE="production"
BACKUP_DIR="./helm-backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

helm get values "$RELEASE" -n "$NAMESPACE" --all -o yaml > "$BACKUP_DIR/values.yaml"
helm get manifest "$RELEASE" -n "$NAMESPACE" > "$BACKUP_DIR/manifest.yaml"
helm get metadata "$RELEASE" -n "$NAMESPACE" -o json > "$BACKUP_DIR/metadata.json"

echo "Backup saved to $BACKUP_DIR"
```

### Exam Scenario

CKAD/CKA: you may need to retrieve the values used for a release to inspect or modify them:
```bash
helm get values my-release -o yaml
helm get manifest my-release | less
```

### Common Mistakes

- Using `helm get values` without `--all` and assuming you see the complete configuration.
- Not specifying `--revision` when investigating a past deployment — the latest may have
  been changed.
- Trying to use `helm get` for uninstalled releases (unless `--keep-history` was used).

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: release: not found` | Release may be in a different namespace; specify with `-n` |
| Values output is very small | You're seeing only user overrides; use `--all` for the full picture |
| Manifest contains placeholders | The manifest is post-template-render; check the raw chart templates if debugging is needed |

### Related Commands

- `helm list` — find release names
- `helm history` — view revision history
- `helm status` — current release status
- `helm template` — render chart templates locally

---

## 17. helm pull

### Purpose

Downloads a chart from a repository (HTTP or OCI) to a local directory without installing it.
The chart can be saved as a `.tgz` archive (default) or extracted to a directory (`--untar`).

### Syntax

```
helm pull [chart URL | repo/chartname] [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--ca-file` | string | `""` | CA certificate for TLS verification of the chart repository. |
| `--cert-file` | string | `""` | Client certificate for TLS authentication. |
| `--key-file` | string | `""` | Client certificate private key. |
| `--destination`, `-d` | string | `"."` | Directory to save the chart archive. Created if it doesn't exist. |
| `--devel` | bool | `false` | Include development (pre-release) versions when resolving chart versions. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS certificate verification. NOT for production use. |
| `--keyring` | string | `"~/.gnupg/secring.gpg"` | Path to PGP keyring for provenance verification. |
| `--pass-credentials` | bool | `false` | Pass HTTP Basic Auth credentials to all domains. |
| `--password` | string | `""` | Chart repository password. |
| `--prov` | bool | `false` | Also download the provenance file (`.tgz.prov`) alongside the chart. |
| `--repo` | string | `""` | Chart repository URL. Allows pulling by chart name without `helm repo add`. |
| `--untar` | bool | `false` | Extract the chart archive to a directory instead of keeping it as `.tgz`. |
| `--untardir` | string | `""` | Directory to extract the chart into when `--untar` is used. Defaults to the chart name in the current directory. |
| `--username` | string | `""` | Chart repository username. |
| `--verify` | bool | `false` | Verify the chart's provenance file before saving. Requires the signer's public key. |
| `--version` | string | `""` | Specific chart version to pull. If omitted, the latest stable version is pulled. |

### Real-World Examples

```bash
# Pull a chart from an added repository
helm pull bitnami/nginx

# Output:
# Pulled: bitnami/nginx
# Saved: ./nginx-15.2.0.tgz
```

```bash
# Pull a specific version
helm pull bitnami/nginx --version 15.0.0

# Pull and extract (for inspection or modification)
helm pull bitnami/nginx --untar

# Pull and extract to a specific directory
helm pull bitnami/nginx --untar --untardir ./my-nginx

# Pull to a specific destination, keeping as archive
helm pull bitnami/nginx --destination ./charts/

# Pull from a repo URL without adding it first
helm pull nginx --repo https://charts.bitnami.com/bitnami

# Pull and verify provenance
helm pull bitnami/nginx --verify --keyring ~/.gnupg/secring.gpg

# Pull with provenance file
helm pull bitnami/nginx --prov

# Pull from OCI registry
helm pull oci://registry.example.com/charts/nginx --version 1.0.0

# Pull with authentication
helm pull private-repo/nginx \
  --username myuser \
  --password mypassword \
  --destination ./charts/

# Pull pre-release version
helm pull bitnami/nginx --devel --version ">=16.0.0-alpha"

# CI: pull chart for local rendering
helm pull bitnami/nginx --version 15.2.0 --destination /tmp/charts/
helm template my-app /tmp/charts/nginx-15.2.0.tgz --values ./values.yaml
```

### Production Scenario

Air-gapped environment: pull charts, transfer them, and install locally:

```bash
#!/bin/bash
# On internet-connected machine
CHART="bitnami/nginx"
VERSION="15.2.0"
OUTPUT_DIR="./offline-charts"

helm pull "$CHART" --version "$VERSION" --destination "$OUTPUT_DIR"

# Transfer $OUTPUT_DIR to air-gapped environment

# On air-gapped machine
helm install my-nginx "$OUTPUT_DIR/nginx-${VERSION}.tgz" \
  --values ./values.yaml
```

### Exam Scenario

CKA/CKAD: pulling a chart for inspection or offline use is commonly tested:
```bash
helm pull bitnami/nginx --untar
# Examine the chart structure
ls nginx/
cat nginx/values.yaml
```

### Common Mistakes

- Pulling without `--untar` and then trying to `cd` into the `.tgz` file to modify templates.
- Forgetting to run `helm repo update` before pulling — the local index may be stale.
- Using `--verify` without the signer's public key in the keyring.
- Assuming `helm pull` from an OCI registry works without `helm registry login` first
  (for private registries).

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: chart "nginx" version "x" not found` | Run `helm repo update` to refresh the index. Check the version exists. |
| `Error: failed to fetch` | The registry is unreachable or requires authentication |
| `Error: unauthorized` | Use `helm registry login` (OCI) or `--username`/`--password` (HTTP) |
| Extracted chart is empty | Verify `--untar` was used; the default is to save as `.tgz` |

### Related Commands

- `helm push` — upload a chart to a repository
- `helm install` — install directly from a repository
- `helm show` — inspect chart metadata without downloading
- `helm verify` — verify chart provenance

---

## 18. helm push

### Purpose

Pushes a packaged chart (`.tgz` file) to an OCI-compliant registry or a chart repository
(requires a plugin for traditional chart repos like ChartMuseum). Native OCI push is
available in Helm v3.8+.

### Syntax

```
helm push [CHART] [REMOTE] [flags]
```

For OCI registries:
```
helm push mychart-1.0.0.tgz oci://registry.example.com/charts/
```

For traditional chart repositories, a push plugin is required (e.g., `helm push` via
`helm-push` plugin).

### OCI Push Workflow

No additional flags beyond the chart path and OCI URL. Authentication is handled
by `helm registry login`.

### Real-World Examples

```bash
# Login to OCI registry
echo "$TOKEN" | helm registry login ghcr.io -u myuser --password-stdin

# Package the chart
helm package ./mychart --destination ./dist/

# Push to OCI registry
helm push ./dist/mychart-1.0.0.tgz oci://ghcr.io/myorg/charts/

# Output:
# Pushed: ghcr.io/myorg/charts/mychart:1.0.0
# Digest: sha256:abc123def456...

# Push with specific version tag (via packaging --version)
helm package ./mychart --version 2.0.0
helm push ./mychart-2.0.0.tgz oci://ghcr.io/myorg/charts/
```

### Using helm-push Plugin (Traditional Repos)

```bash
# Install push plugin
helm plugin install https://github.com/chartmuseum/helm-push

# Push to ChartMuseum
helm push ./mychart-1.0.0.tgz https://charts.example.com/api/charts

# Push with authentication
helm push ./mychart-1.0.0.tgz https://charts.example.com/api/charts \
  --username admin --password secret
```

### Production Scenario

CI/CD chart publishing pipeline:

```bash
#!/bin/bash
CHART_DIR="./charts/my-service"
VERSION=$(git describe --tags --abbrev=0)

# Login to OCI registry
echo "$OCI_PASSWORD" | helm registry login "$OCI_REGISTRY" \
  -u "$OCI_USER" --password-stdin

# Package
helm package "$CHART_DIR" \
  --version "$VERSION" \
  --app-version "$VERSION" \
  --dependency-update \
  --destination ./dist/

# Push
helm push "./dist/my-service-${VERSION}.tgz" "oci://${OCI_REGISTRY}/charts/"

# Logout (security best practice)
helm registry logout "$OCI_REGISTRY"
```

### Exam Scenario

The Helm Certified Associate exam covers OCI push/pull. Know the sequence: `registry login`
→ `package` → `push`.

### Common Mistakes

- Trying to `helm push` to a traditional HTTP chart repository without the `helm-push` plugin.
- Forgetting to `helm registry login` before pushing to an OCI registry.
- Pushing a chart without first updating dependencies (subcharts may be missing).

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: command "push" does not exist` | For OCI, ensure Helm >= 3.8.0. For traditional repos, install the `helm-push` plugin. |
| `Error: unauthorized: authentication required` | Run `helm registry login` first |
| Chart already exists in registry | Bump the chart version or use `--force` (if supported by the registry) |

### Related Commands

- `helm registry login` — authenticate to an OCI registry
- `helm pull` — download a chart from a registry
- `helm package` — create the `.tgz` archive to push

---

## 19. helm verify

### Purpose

Verifies that a packaged chart has a valid provenance (`.prov`) file and that it was signed
by a trusted PGP key. This ensures chart integrity and authenticity — the chart has not been
tampered with and comes from the claimed author.

### Syntax

```
helm verify [CHART] [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--keyring` | string | `"~/.gnupg/secring.gpg"` | Path to the PGP keyring file containing the public key(s) of trusted signers. |

### Real-World Examples

```bash
# Verify a local chart archive
helm verify ./mychart-1.0.0.tgz

# Output (valid):
# Signed by: Jane Doe <jane@example.com>
# Using Key With Fingerprint: AAAA1111BBBB2222CCCC3333DDDD4444EEEE5555
# Chart Hash Verified: sha256:abcdef1234567890...

# Output (invalid):
# Error: Chart has no provenance file

# Verify with custom keyring
helm verify ./mychart-1.0.0.tgz --keyring /etc/helm/trusted-keys.gpg

# Verify during pull
helm pull bitnami/nginx --verify --keyring ~/.gnupg/secring.gpg
```

### Production Scenario

Enforcing chart provenance verification in CI/CD:

```bash
#!/bin/bash
CHART="bitnami/nginx"
VERSION="15.2.0"
KEYRING="/etc/helm/trusted-signers.gpg"

# Pull with automatic verification
helm pull "$CHART" --version "$VERSION" --verify --keyring "$KEYRING" || {
  echo "Chart provenance verification FAILED!"
  exit 1
}

# Install only if verified
helm install my-nginx --verify --keyring "$KEYRING" "$CHART" --version "$VERSION"
```

### Exam Scenario

Helm Certified Associate: understanding chart signing and verification. Key concepts:
- `.prov` files contain the chart's hash and digital signature
- Verification requires the signer's public key in the keyring
- Verification confirms both integrity (unchanged) and authenticity (correct signer)

### Common Mistakes

- Expecting `helm verify` to work on charts that weren't signed (no `.prov` file exists).
- Using a keyring that doesn't contain the signer's public key.
- Confusing `--verify` during `helm install` (verifies before installing) with `helm verify`
  (manual verification of a downloaded chart).

### Troubleshooting

| Problem | Solution |
|---|---|
| `no provenance file found` | The chart was not signed during packaging. Download the `.prov` file or use a signed chart. |
| `verification failed: openpgp: signature made by unknown entity` | The signer's public key is not in the keyring. Import it with `gpg --import`. |
| `chart hash verification failed` | The chart has been tampered with or corrupted. Re-download from a trusted source. |

### Related Commands

- `helm package --sign` — sign a chart during packaging
- `helm install --verify` — verify before installing
- `helm pull --verify` — verify during download

---

## 20. helm show

### Purpose

Displays information about a chart without downloading the full archive. Subcommands target
specific sections of the chart's metadata and contents.

### Subcommands

| Subcommand | Purpose |
|---|---|
| `all` | Show all information (chart metadata, values, readme) |
| `chart` | Show the `Chart.yaml` file contents |
| `crds` | Show the chart's Custom Resource Definitions (from `crds/` directory) |
| `readme` | Show the chart's README.md |
| `values` | Show the chart's `values.yaml` file |

### Syntax

```
helm show SUBCOMMAND [CHART] [flags]
```

### Common Flags (all subcommands)

| Flag | Type | Default | Description |
|---|---|---|---|
| `--ca-file` | string | `""` | CA certificate for TLS verification of the chart repository. |
| `--cert-file` | string | `""` | Client certificate for TLS authentication. |
| `--devel` | bool | `false` | Include development versions. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS certificate verification. |
| `--key-file` | string | `""` | Client certificate private key. |
| `--keyring` | string | `"~/.gnupg/secring.gpg"` | PGP keyring for verification. |
| `--pass-credentials` | bool | `false` | Pass credentials to all domains. |
| `--password` | string | `""` | Repository password. |
| `--repo` | string | `""` | Repository URL (no need for `helm repo add`). |
| `--username` | string | `""` | Repository username. |
| `--verify` | bool | `false` | Verify chart provenance before showing. |
| `--version` | string | `""` | Specific chart version to show. |

### Real-World Examples

```bash
# Show chart metadata
helm show chart bitnami/nginx

# Output:
# apiVersion: v2
# name: nginx
# version: 15.2.0
# appVersion: 1.27.0
# description: NGINX Open Source is a web server...
# ...

# Show the default values file (to know what you can configure)
helm show values bitnami/nginx

# Show the README for installation instructions
helm show readme bitnami/nginx

# Show CRDs (to understand custom resources the chart will create)
helm show crds bitnami/sealed-secrets

# Show all information at once
helm show all bitnami/nginx | less

# Show a specific version
helm show chart bitnami/nginx --version 14.0.0

# Show from a repo URL without adding it
helm show values nginx --repo https://charts.bitnami.com/bitnami

# Pipeline: inspect values before installation
helm show values bitnami/nginx > default-values.yaml
# Customize the values
vim custom-values.yaml
helm install my-nginx bitnami/nginx -f custom-values.yaml
```

### Production Scenario

Pre-installation inspection workflow:

```bash
#!/bin/bash
CHART="bitnami/nginx"
VERSION="15.2.0"

# 1. Check chart metadata (is it the right chart?)
echo "=== Chart Info ==="
helm show chart "$CHART" --version "$VERSION"

# 2. Review default values
echo "=== Default Values ==="
helm show values "$CHART" --version "$VERSION" > /tmp/chart-values.yaml
# Review and customize
vim /tmp/chart-values.yaml

# 3. Read installation notes
echo "=== README ==="
helm show readme "$CHART" --version "$VERSION" | head -50

# 4. Check CRDs (will this chart install any?)
echo "=== CRDs ==="
helm show crds "$CHART" --version "$VERSION" | head -20

# 5. Install with reviewed values
helm install my-nginx "$CHART" --version "$VERSION" -f /tmp/chart-values.yaml
```

### Exam Scenario

CKAD/CKA: you may need to inspect a chart's default values quickly:
```bash
helm show values bitnami/nginx | grep service.type
```

### Common Mistakes

- Using `helm show all` when only a specific section is needed — it produces a very large
  output.
- Not specifying `--version` and getting docs for a version different from the one you
  intend to install.
- Forgetting to `helm repo update` before `helm show`, potentially viewing stale metadata.

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: failed to download` | Run `helm repo update` first |
| Values shown don't match the installed chart | Check the installed version with `helm list` and specify `--version` |
| `Error: chart "x" not found` | The repo may not be added. Use `helm repo add` or `--repo` flag. |

### Related Commands

- `helm pull --untar` — download and inspect local chart files
- `helm install` — install after inspecting
- `helm search` — find charts before showing details

---

## 21. helm search

### Purpose

Searches for charts in configured repositories (`repo`) or on Artifact Hub (`hub`). This is
the discovery command — use it to find charts before inspecting or installing them.

### Subcommands

| Subcommand | Purpose |
|---|---|
| `repo` | Search charts in locally configured repositories (those added via `helm repo add`) |
| `hub` | Search charts on Artifact Hub (https://artifacthub.io), a community chart aggregator |

### 21.1 helm search repo

#### Syntax

```
helm search repo [keyword] [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--devel` | bool | `false` | Include development versions in results. |
| `--max-col-width` | uint | `50` | Maximum column width for the `DESCRIPTION` field. Increase to see more description text. |
| `--output`, `-o` | string | `"table"` | Output format: `table`, `json`, `yaml`. |
| `--regexp`, `-r` | bool | `false` | Treat the search keyword as a regular expression. Without this, search is a substring match. |
| `--version` | string | `""` | Search for charts matching a specific semver version or constraint (e.g., `>=1.0.0`, `~1.2`). |
| `--versions` | bool | `false` | Show all available versions for each chart, not just the latest. |

#### Output Columns

| Column | Description |
|---|---|
| `NAME` | Fully qualified chart name (`repo/chartname`) |
| `CHART VERSION` | Latest chart version matching the query |
| `APP VERSION` | Application version of the chart |
| `DESCRIPTION` | Short description from Chart.yaml |

#### Real-World Examples

```bash
# Search all repos for "nginx"
helm search repo nginx

# Output:
# NAME                    CHART VERSION APP VERSION DESCRIPTION
# bitnami/nginx           15.2.0        1.27.0      NGINX Open Source is a web server...
# bitnami/nginx-ingress-controller 10.0.0 1.9.0   NGINX Ingress Controller...
```

```bash
# Regex search
helm search repo --regexp '^bitnami/(mysql|postgresql)$'

# Show all versions
helm search repo nginx --versions

# JSON output for scripting
helm search repo nginx -o json | jq '.[].name'

# YAML output
helm search repo nginx -o yaml

# Search with version constraint
helm search repo nginx --version ">=15.0.0"

# Search with wider description column (less truncation)
helm search repo nginx --max-col-width 80

# Include development versions
helm search repo nginx --devel
```

### 21.2 helm search hub

#### Syntax

```
helm search hub [keyword] [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--endpoint` | string | `"https://hub.helm.sh"` | Artifact Hub API endpoint. Override for private Hub instances. |
| `--max-col-width` | uint | `50` | Maximum column width for description. |
| `--output`, `-o` | string | `"table"` | Output format: `table`, `json`, `yaml`. |

#### Real-World Examples

```bash
# Search Artifact Hub for monitoring tools
helm search hub prometheus

# Search with JSON output
helm search hub prometheus -o json | jq '.[].url'

# Search for a specific community chart
helm search hub "sealed secrets"
```

### Production Scenario

Finding the right chart for a production stack:

```bash
#!/bin/bash
# Search configured repos first (faster, vetted)
echo "=== Local Repository Results ==="
helm search repo prometheus

# If not found locally, search Artifact Hub
echo "=== Artifact Hub Results ==="
helm search hub prometheus -o json | jq -r '.[] | "\(.name)\t\(.repository.url)"'

# Once found, inspect and install
helm show values prometheus-community/prometheus > values.yaml
```

### Exam Scenario

CKAD/CKA: searching for charts is commonly tested:
```bash
# Quick search
helm search repo nginx

# Check if a specific chart is available
helm search repo bitnami/nginx -o json | jq .
```

### Common Mistakes

- Searching `hub` instead of `repo` — `hub` searches Artifact Hub (internet required),
  while `repo` searches locally configured repositories (offline-friendly after `repo update`).
- Forgetting `helm repo update` before searching — results may be outdated.
- Using complex regex without `--regexp` — the default is a substring match.

### Troubleshooting

| Problem | Solution |
|---|---|
| `helm search repo` returns nothing | Run `helm repo update` to refresh indexes |
| `helm search hub` returns nothing | Requires internet access; check connectivity |
| Search results are stale | Chart versions may have been released since the last `helm repo update` |

### Related Commands

- `helm show` — inspect a chart's details
- `helm repo update` — refresh local repository indexes
- `helm pull` — download a chart

---

## 22. helm template

### Purpose

Renders chart templates locally and prints the resulting Kubernetes manifests to stdout
without contacting a Kubernetes cluster. This is the "dry run without a cluster" command.
Essential for local development, CI/CD validation, and GitOps workflows (where rendered
manifests are committed to a repository).

### Syntax

```
helm template [NAME] [CHART] [flags]
```

### Flags

`helm template` accepts the same flags as `helm install` that relate to template rendering
and value passing. Cluster-interaction flags (`--wait`, `--atomic`, `--create-namespace`)
are NOT applicable and are not present.

#### Template-Specific Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--api-versions` | []string | `[]` | Comma-separated list of Kubernetes API versions to assume when rendering. Used with `Capabilities.APIVersions`. E.g., `--api-versions 'networking.k8s.io/v1/Ingress'`. |
| `--dependency-update`, `-u` | bool | `false` | Run `helm dependency update` before rendering. |
| `--description` | string | `""` | Description for the release (stored in `.Release.Description`). |
| `--devel` | bool | `false` | Include development versions. |
| `--include-crds` | bool | `false` | Include CRDs from the `crds/` directory in the rendered output. By default, CRDs are NOT included. |
| `--is-upgrade` | bool | `false` | Simulate an upgrade instead of a fresh install. Sets `.Release.IsUpgrade` to `true` in templates, which may affect conditional logic in templates. |
| `--kube-version` | string | `""` | Kubernetes version to assume for `Capabilities.KubeVersion`. E.g., `--kube-version 1.29`. |
| `--name-template` | string | `""` | Go template for the release name (when `--generate-name` is used). |
| `--no-hooks` | bool | `false` | Do not render hook templates. |
| `--output-dir` | string | `""` | Write rendered manifests to files in a directory instead of stdout. Each manifest is written to a separate file named after the resource. |
| `--post-renderer` | string | `""` | Path to a post-renderer executable. Processes rendered YAML before output. |
| `--release-name` | bool | `false` | (Deprecated) |
| `--render-subchart-notes` | bool | `false` | Render NOTES.txt from subcharts. |
| `--set` | []string | `[]` | Set values inline. |
| `--set-file` | []string | `[]` | Set values from file contents. |
| `--set-json` | []string | `[]` | Set JSON values. |
| `--set-string` | []string | `[]` | Set values as strings. |
| `--show-only` | []string | `[]` | Render only specific templates matching the pattern. E.g., `--show-only 'templates/deployment.yaml'`. Glob patterns supported. |
| `--skip-crds` | bool | `false` | (Same as excluding CRDs — default behaviour) |
| `--validate` | bool | `true` | Validate rendered manifests against the Kubernetes schema. Set `--validate=false` if cluster schema validation causes issues. |
| `--values`, `-f` | []string | `[]` | Values files to use. |
| `--version` | string | `""` | Chart version to render. |

### Real-World Examples

```bash
# Basic template rendering
helm template my-release ./mychart

# Render to a file
helm template my-release ./mychart --values ./values-prod.yaml > manifests.yaml

# Render to an output directory (one file per resource)
helm template my-release ./mychart --output-dir ./rendered/

# Render only a specific template
helm template my-release ./mychart --show-only 'templates/deployment.yaml'

# Render with Kubernetes version capabilities
helm template my-release ./mychart --kube-version 1.29

# Render with specific API versions
helm template my-release ./mychart \
  --api-versions 'networking.k8s.io/v1/Ingress' \
  --api-versions 'policy/v1/PodDisruptionBudget'

# Render simulating an upgrade
helm template my-release ./mychart --is-upgrade

# Render from a remote chart
helm template my-release bitnami/nginx \
  --version 15.2.0 \
  --values ./values.yaml

# Render from OCI chart
helm template my-release oci://registry.example.com/charts/nginx \
  --version 1.0.0

# CI: validate rendering for multiple environments
for env in dev staging prod; do
  echo "Rendering for $env..."
  helm template my-app ./mychart \
    -f "values-${env}.yaml" \
    --validate \
    > "manifests-${env}.yaml" || exit 1
done
```

### Production Scenario

GitOps workflow: render manifests and commit to a deployment repository:

```bash
#!/bin/bash
CHART="./charts/my-app"
ENVS=("dev" "staging" "prod")

for env in "${ENVS[@]}"; do
  OUTPUT_DIR="./deploy/${env}"
  mkdir -p "$OUTPUT_DIR"

  helm template my-app "$CHART" \
    -f "values-${env}.yaml" \
    --namespace "$env" \
    --output-dir "$OUTPUT_DIR" \
    --include-crds

  echo "Rendered $env manifests to $OUTPUT_DIR"
done

# Commit to Git
git add deploy/
git commit -m "Update rendered manifests"
```

### Exam Scenario

CKA/CKAD: `helm template` is frequently used for:
- Generating manifests without a cluster: `helm template my-app ./mychart > deploy.yaml`
- Debugging template errors: `helm template my-app ./mychart --debug`
- Previewing changes: `helm template my-app ./mychart -f new-values.yaml | kubectl diff -f -`

### Common Mistakes

- Expecting `helm template` to validate against a live Kubernetes cluster — it does not
  connect to a cluster. Use `helm install --dry-run` for cluster-side validation.
- Forgetting `--include-crds` when the chart depends on CRDs.
- Not using `--is-upgrade` when simulating upgrade behaviour (some templates check this flag).
- Rendering to stdout and losing the output — pipe to a file or use `--output-dir`.

### Troubleshooting

| Problem | Solution |
|---|---|
| Template renders nothing for a specific resource | Check `.Values.*` defaults and conditional `if` blocks; try `--set` to enable features |
| `Error: could not find template` | A named template is missing; check `_helpers.tpl` and dependency charts |
| CRDs not in output | Use `--include-crds` |
| `--validate` fails | Schema validation issue. Either fix the manifest or use `--validate=false` |

### Related Commands

- `helm install --dry-run` — cluster-side dry run
- `helm lint` — structural validation
- `helm get manifest` — retrieve the manifest of an installed release

---

## 23. helm dependency

### Purpose

Manages chart dependencies declared in `Chart.yaml` under the `dependencies` field.
Subcommands handle listing, updating (downloading), and building (downloading + syncing
lock file).

### Subcommands

| Subcommand | Purpose |
|---|---|
| `build` | Build the `charts/` directory from the `Chart.lock` file. Downloads dependencies to match the lock file exactly. |
| `list` | List all dependencies declared in `Chart.yaml` and their status. |
| `update` | Update `Chart.lock` with the latest compatible versions and download dependencies. This modifies the lock file. |

### 23.1 helm dependency build

#### Syntax

```
helm dependency build CHART_PATH [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--keyring` | string | `"~/.gnupg/secring.gpg"` | Keyring for verifying signed dependency charts. |
| `--skip-refresh` | bool | `false` | Do not refresh the local repository cache before building. Use when working offline. |
| `--verify` | bool | `false` | Verify provenance of downloaded dependency charts. |

#### Real-World Examples

```bash
# Build dependencies from lock file
helm dependency build ./mychart

# Build without refreshing repo indexes (offline)
helm dependency build ./mychart --skip-refresh

# Build with verification
helm dependency build ./mychart --verify
```

### 23.2 helm dependency list

#### Syntax

```
helm dependency list CHART_PATH [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--max-col-width` | uint | `50` | Maximum column width for the repository column. |

#### Output Columns

| Column | Description |
|---|---|
| `NAME` | Dependency chart name |
| `VERSION` | Declared version constraint |
| `REPOSITORY` | Repository URL |
| `STATUS` | `ok` (dependency is present and matches lock), `missing` (not downloaded), `unpacked` (present but not synced with lock) |

#### Real-World Examples

```bash
helm dependency list ./mychart

# Output:
# NAME    VERSION REPOSITORY                          STATUS
# common  2.x.x   https://charts.bitnami.com/bitnami  ok
# postgresql 12.x.x https://charts.bitnami.com/bitnami ok
```

### 23.3 helm dependency update

#### Syntax

```
helm dependency update CHART_PATH [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--keyring` | string | `"~/.gnupg/secring.gpg"` | Keyring for verifying signed charts. |
| `--skip-refresh` | bool | `false` | Do not refresh repository cache. |
| `--verify` | bool | `false` | Verify provenance of downloaded charts. |

#### Real-World Examples

```bash
# Update dependencies to latest compatible versions
helm dependency update ./mychart

# Output:
# Getting updates for unmanaged Helm repositories...
# ...Successfully got an update from the "bitnami" chart repository
# Update Complete. Happy Helming!
# Saving 2 charts
# Downloading common from repo https://charts.bitnami.com/bitnami
# Downloading postgresql from repo https://charts.bitnami.com/bitnami
# Deleting outdated charts
```

### Production Scenario

Ensuring dependencies are up to date before packaging:

```bash
#!/bin/bash
CHART="./charts/my-app"

# Update dependencies
helm dependency update "$CHART"

# Verify all dependencies are accounted for
helm dependency list "$CHART" | grep -v "ok" && {
  echo "Dependency mismatch detected!"
  exit 1
}

# Lint with subcharts
helm lint "$CHART" --with-subcharts --strict

# Package
helm package "$CHART" --dependency-update
```

### Exam Scenario

CKAD/CKA: working with chart dependencies. May involve:
```bash
# Update and download dependencies
helm dependency update ./mychart

# Check dependency status
helm dependency list ./mychart
```

### Common Mistakes

- Using `helm dependency build` without a `Chart.lock` file — you need `helm dependency
  update` first to generate the lock file.
- Modifying the `dependencies` section in `Chart.yaml` without running `helm dependency
  update` — the `charts/` directory and `Chart.lock` become out of sync.
- Forgetting `--skip-refresh` in offline environments — repo refresh attempts may fail.

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: no repository definition for xxx` | Add the repository with `helm repo add` before updating dependencies |
| `Error: can't get a valid version` | Check the version constraint in `Chart.yaml` — the constrained version may not exist |
| Dependencies not appearing in `charts/` | Run `helm dependency update` (creates lock file and downloads) |
| Lock file out of sync | Delete `Chart.lock` and run `helm dependency update` to regenerate |

### Related Commands

- `helm package --dependency-update` — update dependencies before packaging
- `helm repo add` — add repositories referenced by dependencies
- `helm repo update` — refresh repository indexes

---

## 24. helm repo

### Purpose

Manages Helm chart repositories — collections of packaged charts served over HTTP. This
command handles adding, listing, removing, updating, and indexing repositories.

### Subcommands

| Subcommand | Purpose |
|---|---|
| `add` | Add a chart repository |
| `index` | Generate an `index.yaml` file for a directory of chart packages |
| `list` | List all configured chart repositories |
| `remove` | Remove a chart repository |
| `update` | Update repository indexes (refresh the local cache) |

### 24.1 helm repo add

#### Syntax

```
helm repo add [NAME] [URL] [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--allow-deprecated-repos` | bool | `false` | Allow adding deprecated chart repositories. Some repos are marked as deprecated; this flag bypasses the block. |
| `--ca-file` | string | `""` | CA certificate for TLS verification. |
| `--cert-file` | string | `""` | Client certificate for TLS authentication. |
| `--force-update` | bool | `false` | Replace (override) the repository if it already exists with the same name. Without this, re-adding an existing repo name returns an error. |
| `--insecure-skip-tls-verify` | bool | `false` | Skip TLS verification. |
| `--key-file` | string | `""` | Client certificate private key. |
| `--no-update` | bool | `false` | Do not fetch the repository index after adding. The repo will be listed but charts won't be searchable until `helm repo update`. |
| `--pass-credentials` | bool | `false` | Pass credentials to all domains. |
| `--password` | string | `""` | Repository password. |
| `--username` | string | `""` | Repository username. |

#### Real-World Examples

```bash
# Add the Bitnami chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Output:
# "bitnami" has been added to your repositories

# Add with authentication
helm repo add private-repo https://charts.example.com \
  --username myuser \
  --password mypassword

# Add without immediately fetching the index (offline)
helm repo add my-repo https://charts.example.com --no-update

# Force overwrite an existing repo
helm repo add bitnami https://charts.bitnami.com/bitnami --force-update

# Add with custom CA for TLS verification
helm repo add secure-repo https://charts.internal.com \
  --ca-file /etc/ssl/certs/internal-ca.crt
```

### 24.2 helm repo index

#### Syntax

```
helm repo index [DIR] [flags]
```

Generates an `index.yaml` file from a directory of `.tgz` chart archives.

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--merge` | string | `""` | Merge the generated index into an existing `index.yaml` file. Preserves existing entries and adds new ones. |
| `--url` | string | `""` | Base URL for chart download links in the index. Required so clients know where to download charts. |

#### Real-World Examples

```bash
# Generate an index for a directory of charts
helm repo index ./charts/ --url https://charts.example.com

# Merge with an existing index
helm repo index ./new-charts/ --merge ./existing-index.yaml --url https://charts.example.com
```

### 24.3 helm repo list

#### Syntax

```
helm repo list [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--output`, `-o` | string | `"table"` | Output format: `table`, `json`, `yaml`. |

#### Output Columns

| Column | Description |
|---|---|
| `NAME` | Repository name |
| `URL` | Repository URL |

#### Real-World Examples

```bash
# List configured repos
helm repo list

# Output:
# NAME    URL
# bitnami https://charts.bitnami.com/bitnami
# stable  https://charts.helm.sh/stable

# JSON output
helm repo list -o json
```

### 24.4 helm repo remove

#### Syntax

```
helm repo remove [REPO_NAME] [flags]
```

No flags.

#### Real-World Examples

```bash
helm repo remove bitnami

# Output:
# "bitnami" has been removed from your repositories
```

### 24.5 helm repo update

#### Syntax

```
helm repo update [flags]
```

#### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--fail-on-repo-update-fail` | bool | `true` | If a single repository fails to update, the entire command fails. Set to `false` to continue updating other repos even if one fails. |

#### Real-World Examples

```bash
# Update all repositories
helm repo update

# Output:
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "bitnami" chart repository
# ...Successfully got an update from the "stable" chart repository
# Update Complete. Happy Helming!

# Update but don't fail if one repo is unreachable
helm repo update --fail-on-repo-update-fail=false
```

### Production Scenario

Setting up repositories in a new environment:

```bash
#!/bin/bash
# Add required repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io

# Update indexes
helm repo update

# Verify
helm repo list
helm search repo nginx
```

### Exam Scenario

CKA/CKAD: repository management is fundamental:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/nginx
helm install my-nginx bitnami/nginx
```

### Common Mistakes

- Adding a repo and immediately searching without `helm repo update` — the index is fetched
  automatically on `add`, but if search fails, run `update`.
- Forgetting `--url` when generating an `index.yaml` — chart download links will be broken.
- Not using `--force-update` when re-adding a repo in automation scripts, causing errors.

### Troubleshooting

| Problem | Solution |
|---|---|
| `Error: looks like "https://example.com" is not a valid chart repository` | The URL doesn't serve an `index.yaml`. Verify the URL and the repository server. |
| `Error: could not find protocol handler for` | The URL scheme is unsupported. Use `https://` or `http://`. |
| Repository index is stale | Run `helm repo update` |
| TLS certificate error | Use `--ca-file` for custom CAs or `--insecure-skip-tls-verify` (non-production only) |

### Related Commands

- `helm search repo` — search charts in configured repositories
- `helm dependency update` — download chart dependencies
- `helm pull` — download a chart from a repository

---

## 25. helm registry login / logout

> These are the same commands described in Section 4. They are referenced here for
> completeness as they are essential for OCI-based chart distribution.

### Syntax

```
helm registry login [HOST] [flags]
helm registry logout [HOST]
```

Refer to [Section 4](#4-helm-registry) for the full flag reference, examples, and
production scenarios.

---

## 26. helm completion

### Purpose

Generates shell auto-completion scripts for Bash, Zsh, Fish, and PowerShell. After sourcing
the generated script, tab-completion works for all Helm commands, subcommands, and flags.

### Subcommands

| Subcommand | Shell |
|---|---|
| `bash` | Bash auto-completion |
| `zsh` | Zsh auto-completion |
| `fish` | Fish auto-completion |
| `powershell` | PowerShell auto-completion |

### Syntax

```
helm completion [SHELL] [flags]
```

These subcommands have no flags.

### Real-World Examples

```bash
# Generate Bash completion and source it
helm completion bash > /etc/bash_completion.d/helm
source <(helm completion bash)

# Generate Zsh completion
helm completion zsh > "${fpath[1]}/_helm"

# For Oh My Zsh users: ensure the completion file is in fpath
mkdir -p ~/.oh-my-zsh/completions
helm completion zsh > ~/.oh-my-zsh/completions/_helm

# Fish completion
helm completion fish > ~/.config/fish/completions/helm.fish

# PowerShell completion
helm completion powershell > $PROFILE.CurrentUserCurrentHost

# Persistent Bash completion (add to ~/.bashrc)
echo 'source <(helm completion bash)' >> ~/.bashrc
```

### Production Scenario

When provisioning developer workstations or CI runner images, include Helm completion
for a better developer experience:

```bash
# In a Dockerfile or setup script
helm completion bash > /etc/bash_completion.d/helm
echo 'source /etc/bash_completion.d/helm' >> /etc/bash.bashrc
```

### Exam Scenario

Not typically tested, but useful in the exam environment for faster command entry.

### Common Mistakes

- Generating completion for the wrong shell (e.g., `bash` completion for `zsh`).
- Not sourcing the completion file after generating it.
- Generating completion for a non-existent directory (ensure `fpath` is configured for Zsh).

### Troubleshooting

| Problem | Solution |
|---|---|
| Tab completion doesn't work | Ensure the completion script is sourced in your shell's rc file |
| `helm: command not found` after completion | Completion only helps with arguments; Helm must still be installed and in `$PATH` |

### Related Commands

- `helm help` — shows available commands and their purposes

---

## 27. helm help

### Purpose

Displays global help information for Helm. Running `helm help` (or `helm --help`) shows the
list of all available commands. `helm help [command]` shows help for a specific command.

### Syntax

```
helm help [command] [flags]
```

This command accepts no flags.

### Real-World Examples

```bash
# Global help
helm help

# Help for a specific command
helm help install

# Alternative syntax (equivalent)
helm install --help

# Get help for a subcommand
helm help repo add
helm repo add --help
```

### Production Scenario

When unsure about a command's flags, consult `helm help` before running in production:

```bash
helm help upgrade | less
```

### Exam Scenario

The built-in help is available during the exam. Use it:
```bash
helm install --help | grep -i "dry-run"
```

---

## 28. Summary Tables

### 28.1 Flag Categories

| Category | Flags | Applies To |
|---|---|---|
| **Value Overrides** | `--set`, `--set-file`, `--set-json`, `--set-string`, `--values`, `-f` | install, upgrade, template, lint |
| **Chart Selection** | `--version`, `--devel`, `--repo` | install, upgrade, template, pull, show |
| **Release Safety** | `--atomic`, `--cleanup-on-fail`, `--force`, `--dry-run` | install, upgrade, rollback |
| **Waiting Behaviour** | `--wait`, `--wait-for-jobs`, `--timeout` | install, upgrade, rollback, uninstall |
| **Hooks Control** | `--no-hooks` | install, upgrade, rollback, uninstall |
| **History Management** | `--history-max`, `--keep-history` | install, upgrade, rollback, uninstall |
| **TLS / Authentication** | `--ca-file`, `--cert-file`, `--key-file`, `--insecure-skip-tls-verify`, `--username`, `--password`, `--pass-credentials` | install, upgrade, pull, show |
| **Chart Verification** | `--verify`, `--keyring` | install, pull |
| **Output Control** | `--output`, `-o`, `--short`, `-q`, `--quiet` | list, history, search, repo list |
| **Namespace Control** | `--namespace`, `-n`, `--all-namespaces`, `-A`, `--create-namespace` | install, upgrade, list |
| **Name Management** | `--generate-name`, `--name-template`, `--replace` | install, template |
| **Template Rendering** | `--api-versions`, `--kube-version`, `--is-upgrade`, `--include-crds`, `--show-only`, `--validate`, `--output-dir`, `--post-renderer` | template |

### 28.2 Output Format Options (`--output`, `-o`)

| Value | Description | Supported By |
|---|---|---|
| `table` | Human-readable table (default) | list, history, search, repo list, status |
| `json` | Machine-readable JSON | list, history, search, repo list, status, get values |
| `yaml` | Human-readable YAML | list, history, search, repo list, status, get values |

### 28.3 Release Status Values

| Status | Meaning |
|---|---|
| `deployed` | Release is installed and operational |
| `failed` | Install, upgrade, or rollback did not complete successfully |
| `pending-install` | Install operation is in progress |
| `pending-upgrade` | Upgrade operation is in progress |
| `pending-rollback` | Rollback operation is in progress |
| `superseded` | Release has been replaced by a newer revision |
| `uninstalling` | Uninstall operation is in progress |
| `uninstalled` | Release has been soft-deleted (history kept via `--keep-history`) |
| `unknown` | Helm cannot determine the release state |

### 28.4 Time Format Values

Go `time.Format` layout reference (for `--time-format` flag):

| Format String | Example Output |
|---|---|
| `2006-01-02 15:04:05.999999 -0700 MST` | `2026-07-11 12:00:00.000000 +0000 UTC` (default) |
| `2006-01-02` | `2026-07-11` |
| `15:04:05` | `12:00:00` |
| `2006-01-02T15:04:05Z07:00` | `2026-07-11T12:00:00Z` |
| `Mon, 02 Jan 2006 15:04:05 MST` | `Sat, 11 Jul 2026 12:00:00 UTC` |
| `1136239445` (Unix timestamp) | (not a Go layout; use `jq` for Unix timestamps) |

### 28.5 Duration Format

Durations are specified in Go duration string format:

| Unit | Example |
|---|---|
| Seconds (`s`) | `30s` |
| Minutes (`m`) | `5m` |
| Hours (`h`) | `1h` |
| Combination | `1h30m15s`, `5m0s`, `10m` |

### 28.6 Helm Exit Codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | General error |
| `2` | Misuse of shell builtins (rare) |
| `126` | Command invoked cannot execute |
| `127` | Command not found |

### 28.7 Helm Configuration Files

| File | Purpose |
|---|---|
| `~/.config/helm/repositories.yaml` | List of configured chart repositories |
| `~/.cache/helm/repository/` | Cached repository index files |
| `~/.config/helm/registry/config.json` | OCI registry authentication credentials |
| `~/.local/share/helm/plugins/` | Installed Helm plugins |
| `~/.local/share/helm/starters/` | Chart starter templates |

---

*This concludes Chapter 9: Helm Commands — Complete Reference.*
