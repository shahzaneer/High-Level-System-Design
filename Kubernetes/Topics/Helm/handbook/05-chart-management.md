# Chapter 5: Chart Management

Helm charts are the packaging format for Kubernetes applications. This chapter covers chart structure, development, dependency management, linting, templating, versioning, and best practices.

---

## Complete Chart Directory Structure

A typical Helm chart created with `helm create mychart` produces the following structure:

```
mychart/
├── .helmignore          # File patterns to exclude from packaging
├── Chart.yaml           # Chart metadata and specification
├── Chart.lock           # Dependency lock file (generated, do not edit)
├── values.yaml          # Default configuration values
├── charts/              # Subchart dependencies (populated by helm dependency update)
├── crds/                # Custom Resource Definitions
├── templates/           # Kubernetes manifest templates
│   ├── NOTES.txt        # Post-install help text
│   ├── _helpers.tpl     # Named template partials and reusable functions
│   ├── deployment.yaml  # Example deployment template
│   ├── service.yaml     # Example service template
│   ├── hpa.yaml         # Example HorizontalPodAutoscaler
│   ├── ingress.yaml     # Example Ingress template
│   ├── serviceaccount.yaml
│   └── tests/           # Helm test definitions
│       └── test-connection.yaml
└── README.md            # (optional) Chart documentation
```

Every file and directory has a specific role. Understanding each is essential for chart development.

---

## Chart.yaml — Every Field Explained

`Chart.yaml` is the mandatory metadata file. It must be at the root of every chart.

### Required Fields

| Field | Type | Description |
|---|---|---|
| `apiVersion` | string | The Helm chart API version. `v1` for Helm 3 (charts using old format), `v2` for modern Helm 3 charts. Must be `v2` to use `dependencies`, `type`, or OCI functionality. |
| `name` | string | The chart name. Must be lowercase, alphanumeric with hyphens. Used as the release name prefix. Example: `my-nginx` |
| `version` | string | A SemVer 2 version string. Used for chart versioning. Example: `1.2.3` |

### Optional Fields

| Field | Type | Description |
|---|---|---|
| `appVersion` | string | The version of the application this chart packages. Informational; displayed by `helm search`. Example: `"1.26.0"` |
| `description` | string | A single-sentence description of the chart. Displayed in search results. |
| `type` | string | Chart type: `application` (default) or `library`. Library charts provide utilities and named templates; they do not produce release objects. |
| `keywords` | []string | List of keywords for search indexing. Example: `["nginx", "web", "http", "proxy"]` |
| `home` | string | URL of the project's home page. |
| `sources` | []string | List of URLs to the project's source code. |
| `maintainers` | []object | List of maintainer objects, each containing `name` (required), `email`, and `url`. |
| `icon` | string | URL to an SVG or PNG icon for the chart. Displayed in chart UIs and Artifact Hub. |
| `deprecated` | bool | Marks the chart as deprecated. `helm search` shows a deprecation warning. |
| `annotations` | map[string]string | Arbitrary key-value metadata. Used by Artifact Hub for categorization. |
| `kubeVersion` | string | SemVer constraint for compatible Kubernetes versions. Example: `">=1.25.0-0 <1.30.0-0"` |
| `dependencies` | []object | List of subchart dependencies. Each has `name`, `version`, `repository`, and optionally `condition`, `tags`, `import-values`, `alias`. |

### Complete Example

```yaml
apiVersion: v2
name: my-nginx
version: 1.0.0
appVersion: "1.26.0"
description: A Helm chart for deploying NGINX on Kubernetes
type: application
keywords:
  - nginx
  - web
  - http
  - reverse-proxy
home: https://nginx.org
sources:
  - https://github.com/nginx/nginx
maintainers:
  - name: devops-team
    email: devops@example.com
    url: https://example.com/team
icon: https://nginx.org/nginx.png
deprecated: false
annotations:
  category: Infrastructure
  artifacthub.io/changes: |
    - Initial release of NGINX chart
kubeVersion: ">=1.25.0"
dependencies:
  - name: redis
    version: "18.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
    tags:
      - caching
  - name: postgresql
    version: "15.x.x"
    repository: oci://registry-1.docker.io/bitnamicharts
    condition: postgresql.enabled
    alias: database
    import-values:
      - child: service.ports
        parent: databasePorts
```

### Dependency Field Details

| Dependency Field | Description |
|---|---|
| `name` | Chart name in the repository. Must match the name in the dependency's `Chart.yaml`. |
| `version` | SemVer constraint. `"18.x.x"` matches any 18.0.0+ version. `"18.0.0"` pins to exact. `">=15.0.0,<16.0.0"` for ranges. |
| `repository` | The repository URL. Can be a traditional Helm repo URL or an OCI reference (`oci://...`). |
| `condition` | Points to a boolean in `values.yaml` (dot-separated path). If false, the dependency is skipped. |
| `tags` | List of tags checked against `--set tags.caching=true`. If all tags match, the dependency is included. |
| `alias` | Renames the dependency chart. The alias is used for value overrides. |
| `import-values` | Imports child chart values into the parent. `child` is the path in the child; `parent` is the path in the parent. |

**Exam Tip:** If `condition` evaluates to `false`, the dependency is not installed, even if `tags` would match. Both `condition` and `tags` must allow inclusion for the dependency to be installed.

---

## values.yaml — Purpose, Structure, and Schema

### Purpose

`values.yaml` defines the default configuration for a chart. Users override these defaults at install or upgrade time using `--set`, `--values` (or `-f`), or multiple values files merged in order.

### Structure

Values files are YAML maps. Any key in `values.yaml` is accessible in templates via `.Values.<key>`.

```yaml
# values.yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.26.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  hosts:
    - host: nginx.example.local
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

nodeSelector: {}
tolerations: []
affinity: {}

config:
  workerProcesses: auto
  workerConnections: 1024

redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    password: ""
```

### Naming Conventions

- Use `camelCase` for keys (convention, not enforced): `replicaCount`, not `replica_count`.
- Use lowercase for top-level keys: `image`, `service`, `ingress`.
- Group related settings under a parent key: `image.repository`, `image.tag`, `image.pullPolicy`.
- Prefer nested maps over flat keys: `image.repository` over `imageRepository`.

### JSON Schema Validation

Helm supports validating `values.yaml` against a JSON Schema. Create a `values.schema.json` at the chart root:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100
    },
    "image": {
      "type": "object",
      "properties": {
        "repository": { "type": "string" },
        "tag": { "type": "string" },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      },
      "required": ["repository", "tag"]
    }
  },
  "required": ["replicaCount", "image"]
}
```

Helm validates values against this schema on `helm install`, `helm upgrade`, and `helm lint`. Schema violations result in an error.

**Production Note:** Always include `values.schema.json` in production charts. It catches misconfigurations before deployment and serves as documentation for supported values.

---

## templates/ Directory

The `templates/` directory contains Kubernetes manifest files written in Go template syntax. Helm combines these templates with `values.yaml` to produce valid Kubernetes YAML.

### Naming Conventions

- Templates are `.yaml`, `.yml`, `.tpl`, or `.txt` files.
- Kubernetes resource templates typically use lowercase with hyphens: `deployment.yaml`, `service-account.yaml`.
- `NOTES.txt` is a special template rendered and displayed post-install.
- `_helpers.tpl` (or any file starting with `_`) contains named template definitions (partials). Files starting with `_` are NOT rendered as standalone Kubernetes resources.

### YAML Separators

Each template file should begin with `---`:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
```

The `---` separator is optional but recommended for readability when Helm concatenates multiple templates.

### Go Template Basics

Helm templates use Go's `text/template` and `html/template` packages plus the Sprig function library (70+ functions).

**Accessing values:**

```
{{ .Values.replicaCount }}
{{ .Values.image.repository }}
{{ .Release.Name }}
{{ .Release.Namespace }}
{{ .Chart.Name }}
{{ .Chart.Version }}
```

**Conditionals:**

```
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

The `{{-` syntax trims leading whitespace; `-}}` trims trailing whitespace.

**Loops:**

```
{{- range .Values.ingress.hosts }}
  - host: {{ .host | quote }}
{{- end }}
```

**Functions:**

```
name: {{ include "mychart.fullname" . | trunc 63 | trimSuffix "-" }}
{{- with .Values.imagePullSecrets }}
imagePullSecrets:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

### Built-in Objects

| Object | Description |
|---|---|
| `.Values` | Merged values from `values.yaml`, `--set`, and `-f` files |
| `.Release` | Release metadata: `.Release.Name`, `.Release.Namespace`, `.Release.Service` (always "Helm"), `.Release.IsInstall`, `.Release.IsUpgrade`, `.Release.Revision` |
| `.Chart` | Contents of `Chart.yaml`: `.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion`, `.Chart.Description`, etc. |
| `.Files` | Access to non-template files in the chart (config files, scripts): `.Files.Get`, `.Files.Glob`, `.Files.AsConfig`, `.Files.AsSecrets` |
| `.Capabilities` | Kubernetes cluster capabilities: `.Capabilities.KubeVersion`, `.Capabilities.APIVersions` |
| `.Template` | Current template metadata: `.Template.Name`, `.Template.BasePath` |
| `.Subcharts` | Access to subchart globals in library charts |
| `.Library` | Access to library chart utilities |

### The `required` Function

Enforce that a value is provided:

```
{{ required "A valid image.repository is required!" .Values.image.repository }}
```

### The `fail` Function

Abort rendering with a custom message:

```
{{- if not .Values.config.secretKey }}
{{- fail "config.secretKey is required" }}
{{- end }}
```

### Template Context (the Dot)

In templates, `.` (the dot) represents the current scope. It is reset inside `range`, `with`, and `define` blocks:

```
{{- range .Values.ingress.hosts }}
  host: {{ .host }}     # Dot is now a host object
  name: {{ $.Release.Name }}  # Use $ to access root scope
{{- end }}
```

---

## charts/ Directory

The `charts/` directory contains chart dependencies — subcharts that are bundled with the parent chart.

### Two Ways to Manage Dependencies

**Method 1: Declarative (recommended)**

List dependencies in `Chart.yaml` under the `dependencies` key, then run:

```bash
helm dependency update
```

This downloads dependency charts into `charts/` and generates `Chart.lock`.

**Method 2: Manual**

Manually place chart `.tgz` files or expanded chart directories into `charts/`. This method bypasses `Chart.lock` and is less reproducible. Avoid in production.

### Subchart Value Overrides

Override subchart values from the parent chart:

```yaml
# parent values.yaml
redis:
  enabled: true
  architecture: standalone
  auth:
    password: "supersecret"
```

The subchart's `values.yaml` is merged under its chart name key.

### Global Values

Pass values to ALL subcharts using the `global` key:

```yaml
# parent values.yaml
global:
  imageRegistry: registry.internal.example.com
  imagePullSecrets:
    - name: internal-registry
  storageClass: fast-ssd
```

Subcharts access: `.Values.global.imageRegistry`.

**Note:** Overuse of globals creates tight coupling between parent and child charts. Use sparingly.

---

## _helpers.tpl — Named Templates

`_helpers.tpl` is a convention (not a requirement) for storing reusable template snippets. Named templates are defined with `define` and used with `include` or `template`.

### Defining a Named Template

```
{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to 63.
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
```

### Using a Named Template

```
name: {{ include "mychart.fullname" . }}
```

The `include` function passes the root context (`.`) to the template, making all objects available. Use `include` over `template` because `include` returns a string (pipeline-friendly), while `template` writes output inline.

### Common Helpers

Every well-structured chart defines these helpers:

```yaml
{{- define "mychart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Best Practices for Helpers

- Always use `include` over `template` for pipeline compatibility.
- Pass the root context (`.`) to `include` so helpers can access `.Values`, `.Release`, etc.
- Keep helper names scoped: `"mychart.labels"`, not just `"labels"`.
- Place helpers in files starting with `_` (e.g., `_helpers.tpl`) so they aren't rendered as standalone resources.
- Use `{{-` and `-}}` to control whitespace within helpers.

---

## NOTES.txt

`NOTES.txt` is a special template rendered and displayed to the user after `helm install` or `helm upgrade`.

### Purpose

- Provide post-install instructions (how to access the application)
- Display generated credentials (warn users to save them)
- Show next steps (DNS configuration, ingress setup, generating TLS certs)

### Example

```
{{- if .Values.ingress.enabled }}
{{- range .Values.ingress.hosts }}
You can access the application at:
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ .host }}
{{- end }}
{{- else }}
===========================================================================
To access the application from outside the cluster, run:
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} \
    -l "app.kubernetes.io/name={{ include "mychart.name" . }}" \
    -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:80
  echo "Visit http://127.0.0.1:8080 to use the application"
===========================================================================
{{- end }}

{{- if contains "bitnami" .Chart.Name }}
**Warning:** This chart requires Bitnami's container images.
Ensure your cluster has access to docker.io/bitnami.
{{- end }}
```

### Template Functions Commonly Used in NOTES.txt

| Function | Usage |
|---|---|
| `include` | Insert a named template |
| `quote` | Wrap a value in quotes |
| `default` | Fallback value |
| `contains` | Check if string contains substring |
| `lookup` | Query the Kubernetes API (limited; use with caution) |

**Note:** `NOTES.txt` is not a Kubernetes resource. It is rendered to stdout only.

---

## .helmignore

The `.helmignore` file uses the same syntax as `.gitignore` to exclude files from the packaged chart (`.tgz`).

### Default .helmignore (generated by `helm create`)

```
# Patterns to ignore when building packages.
.DS_Store
# Common VCS dirs
.git/
.gitignore
.bzr/
.bzrignore
.hg/
.hgignore
.svn/
# Common backup files
*.swp
*.bak
*.tmp
*.orig
*~
# Various IDE files
.idea/
*.tmproj
.vscode/
```

### Additional Useful Patterns

```
# Exclude test files from packaging
tests/
# Exclude local values overrides
values-dev.yaml
values-prod.yaml
# Exclude CI configurations
.github/
.gitlab-ci.yml
Jenkinsfile
# Exclude documentation
docs/
# Exclude Helm-specific test files
helm-config.yaml
```

**Production Note:** Review `.helmignore` before every release. Accidentally packaging local override files or CI config can expose sensitive information.

---

## Chart.lock

`Chart.lock` is the dependency lock file. It is generated by `helm dependency update` and should be committed to version control.

### Format

```yaml
dependencies:
  - name: redis
    repository: https://charts.bitnami.com/bitnami
    version: 18.6.0
  - name: postgresql
    repository: https://charts.bitnami.com/bitnami
    version: 15.4.0
digest: sha256:abc123def456789...
generated: "2024-03-15T10:30:00.000000000Z"
```

### Purpose

- Locks dependency versions to exact versions (not semver ranges).
- Ensures reproducible builds — every developer gets the exact same subchart versions.
- The `digest` field is a SHA-256 hash of the dependency charts' contents for integrity verification.

### Regenerating

```bash
# Rebuild lock file from Chart.yaml semver ranges
helm dependency build

# Rebuild and download charts
helm dependency update
```

**Difference between `update` and `build`:**
- `helm dependency update`: Downloads chart packages to `charts/` AND regenerates `Chart.lock`.
- `helm dependency build`: Regenerates `Chart.lock` from `Chart.yaml` without re-downloading (uses existing `charts/` content).

**Exam Tip:** If `Chart.lock` is present, `helm dependency build` uses it to download exact versions. If absent, `build` resolves semver ranges from `Chart.yaml`.

---

## crds/ Directory

Custom Resource Definitions (CRDs) placed in the `crds/` directory receive special treatment from Helm.

### Helm's CRD Handling

1. CRDs are installed **before** any templates in the `templates/` directory.
2. CRDs are **never** updated or deleted on `helm upgrade` or `helm uninstall`. This prevents accidental data loss (CRD deletion cascades to all custom resources).
3. CRDs are rendered as-is — templates in `crds/` are NOT processed by the Go template engine. You cannot use `{{ .Values }}`, `{{ .Release }}`, etc., in CRD files.
4. Only YAML files in `crds/` are processed. JSON files are ignored.

### Example CRD

```yaml
# crds/certificate.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: certificates.cert-manager.io
spec:
  group: cert-manager.io
  names:
    kind: Certificate
    listKind: CertificateList
    plural: certificates
    singular: certificate
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
```

### CRD Lifecycle

| Operation | Behavior |
|---|---|
| `helm install` | CRDs are installed. |
| `helm upgrade` | CRDs are NOT updated. New CRDs in the chart are NOT installed. |
| `helm uninstall` | CRDs are NOT deleted. |
| `helm rollback` | CRDs are NOT rolled back. |

**Warning:** There is a known limitation: if a chart has CRDs and you upgrade it with a new CRD, the new CRD is silently skipped. You must manually apply new CRDs. This is by design to protect existing custom resources.

**Production Note:** Many operators (cert-manager, Prometheus Operator, Istio) use separate Helm charts for CRDs and the operator deployment. This separation allows the CRD chart to be installed once, while the operator chart can be upgraded independently.

---

## tests/ Directory

The `templates/tests/` directory contains test pod templates. Helm tests are Kubernetes Pods (or Jobs) that validate a release is working correctly.

### Test Pod Template

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test-connection"
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "mychart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

### Running Tests

```bash
# Run all tests for a release
helm test my-release

# Run and show logs on failure
helm test my-release --logs

# Run a specific test
helm test my-release --filter "name=test-connection"

# Set a timeout (default: 300s)
helm test my-release --timeout 30s
```

### Test Hook Annotations

The key annotation is `"helm.sh/hook": test`. Additional hook annotations:

| Annotation | Description |
|---|---|
| `helm.sh/hook: test` | Marks the resource as a test. |
| `helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded` | Clean up test resources after success. |
| `helm.sh/hook-weight: "5"` | Control execution order (lower = earlier). |

**Exam Tip:** Test pods are executed on `helm test`, NOT on `helm install`. A passing Helm install does NOT mean the chart is functional — you must `helm test` to validate.

---

## `helm create` — Scaffolding a Chart

### Syntax

```
helm create [NAME] [flags]
```

### Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `-p`, `--starter` | string | (none) | Use a starter chart template (from `$HELM_DATA_HOME/starters/`) |
| `--skip-crds` | bool | false | Omit the `crds/` directory from the scaffold |

### Generated Files Explained

```bash
helm create my-app
```

| File/Directory | Purpose |
|---|---|
| `Chart.yaml` | Chart metadata with `apiVersion: v2`, name, version, appVersion. Filled with placeholders. |
| `values.yaml` | Default configuration with `replicaCount`, `image`, `service`, `ingress`, `resources`, etc. |
| `charts/` | Empty directory for subchart dependencies. |
| `templates/` | Directory for Kubernetes resource templates. |
| `templates/NOTES.txt` | Post-install message template showing how to access the deployed app. |
| `templates/_helpers.tpl` | Standard named templates: `mychart.name`, `mychart.fullname`, `mychart.chart`, `mychart.labels`, `mychart.selectorLabels`. |
| `templates/deployment.yaml` | Deployment resource with liveness/readiness probes, image pull policy, and securityContext. |
| `templates/service.yaml` | Service resource exposing the application. |
| `templates/hpa.yaml` | HorizontalPodAutoscaler (disabled by default via `autoscaling.enabled`). |
| `templates/ingress.yaml` | Ingress resource (disabled by default via `ingress.enabled`). |
| `templates/serviceaccount.yaml` | ServiceAccount (enabled by default via `serviceAccount.create`). |
| `templates/tests/test-connection.yaml` | Test pod to verify basic HTTP connectivity to the service. |
| `.helmignore` | Default ignore patterns for packaging. |

### Using a Starter Chart

Create reusable chart templates (starters):

```bash
# Create a custom starter from an existing chart
cp -r my-nginx $HELM_DATA_HOME/starters/nginx-starter

# Use the starter to create a new chart
helm create my-app --starter nginx-starter
```

**Note:** Helm starters are simply chart templates stored in `$HELM_DATA_HOME/starters/<name>/`. The `helm create` command copies the starter and replaces the chart name placeholder.

---

## `helm package` — Creating a .tgz

### Syntax

```
helm package [CHART_PATH] [flags]
```

### Flags

| Flag | Type | Description |
|---|---|---|
| `--app-version` | string | Override `appVersion` in the packaged chart |
| `-u`, `--dependency-update` | bool | Run `helm dependency update` before packaging |
| `-d`, `--destination` | string | Output directory for the `.tgz` file (default `.`) |
| `--key` | string | PGP private key for signing the chart |
| `--keyring` | string | PGP keyring path (default `~/.gnupg/pubring.gpg`) |
| `--passphrase-file` | string | File containing the PGP passphrase |
| `--sign` | bool | Sign the chart using PGP |
| `--version` | string | Override the `version` in the packaged chart |

### Examples

```bash
# Basic package
helm package ./my-chart
# Creates: my-chart-0.1.0.tgz

# Package with specific output directory
helm package ./my-chart --destination ./releases

# Package with version override
helm package ./my-chart --version 1.0.0
# Creates: my-chart-1.0.0.tgz

# Package and update dependencies
helm package ./my-chart --dependency-update

# Package and sign
helm package ./my-chart --sign --key "devops@example.com" \
  --passphrase-file ./passphrase.txt
# Creates: my-chart-0.1.0.tgz and my-chart-0.1.0.tgz.prov (provenance file)
```

**Production Note:** Always use `--dependency-update` in CI/CD pipelines to ensure the packaged chart includes the latest compatible subchart versions as defined in `Chart.lock`.

---

## `helm show` — Inspect Chart Metadata

### Subcommands

| Command | Description |
|---|---|
| `helm show chart [CHART]` | Display `Chart.yaml` contents |
| `helm show values [CHART]` | Display `values.yaml` contents |
| `helm show readme [CHART]` | Display `README.md` contents |
| `helm show all [CHART]` | Display all three above |

### Syntax

```
helm show <subcommand> [CHART_REFERENCE] [flags]
```

`CHART_REFERENCE` can be:
- A chart reference: `bitnami/nginx`
- A local path: `./my-chart`
- A packaged chart: `./my-chart-0.1.0.tgz`
- An OCI reference: `oci://registry.example.com/charts/nginx`

### Flags

| Flag | Type | Description |
|---|---|---|
| `--ca-file` | string | CA certificate for TLS |
| `--cert-file` | string | Client certificate for TLS |
| `--devel` | bool | Include pre-release versions |
| `--insecure-skip-tls-verify` | bool | Skip TLS verification |
| `--jsonpath` | string | JSONPath expression to filter output |
| `--key-file` | string | Client private key for TLS |
| `--keyring` | string | GPG keyring for signature verification |
| `--password` | string | Repository password |
| `--repo` (or `--url`) | string | Repository URL (if chart reference is just a name) |
| `--username` | string | Repository username |
| `--verify` | bool | Verify the chart's provenance signature |
| `--version` | string | Specific chart version |

### Examples

```bash
# Show chart metadata from a repository
helm show chart bitnami/nginx

# Show default values
helm show values bitnami/nginx

# Show values for a specific version
helm show values bitnami/nginx --version 15.0.0

# Show README
helm show readme bitnami/nginx

# Extract a specific field with jsonpath
helm show chart bitnami/nginx --jsonpath '{.version}'
# Output: 15.0.0

# Show values from a local chart
helm show values ./my-chart

# Show values from a packaged chart
helm show values ./my-chart-0.1.0.tgz

# Show values with devel (pre-release) versions included
helm show values my-chart --devel

# Verify signed chart provenance before showing
helm show all bitnami/nginx --verify
```

---

## `helm pull` — Download a Chart

### Syntax

```
helm pull [CHART_REFERENCE] [flags]
```

### Flags

| Flag | Type | Description |
|---|---|---|
| `--ca-file` | string | CA certificate for TLS |
| `--cert-file` | string | Client certificate for TLS |
| `--devel` | bool | Include pre-release versions |
| `-d`, `--destination` | string | Output directory (default `.`) |
| `--insecure-skip-tls-verify` | bool | Skip TLS verification |
| `--key-file` | string | Client private key for TLS |
| `--keyring` | string | GPG keyring for signature verification |
| `--password` | string | Repository password |
| `--prov` | bool | Download the provenance file (`.prov`) |
| `--repo` (or `--url`) | string | Repository URL |
| `--untar` | bool | Extract the chart to a directory after downloading |
| `--untardir` | string | Directory to untar into (defaults to chart name) |
| `--username` | string | Repository username |
| `--verify` | bool | Verify chart provenance before downloading |
| `--version` | string | Specific chart version (required if multiple versions exist) |

### Examples

```bash
# Download a chart .tgz
helm pull bitnami/nginx --version 15.0.0
# Creates: nginx-15.0.0.tgz

# Download and extract
helm pull bitnami/nginx --version 15.0.0 --untar
# Creates: nginx/ directory

# Download to a specific directory
helm pull bitnami/nginx --version 15.0.0 --destination ./charts

# Download with provenance verification
helm pull bitnami/nginx --version 15.0.0 --verify --prov
# Creates: nginx-15.0.0.tgz and nginx-15.0.0.tgz.prov

# Download pre-release version
helm pull istio/base --devel

# Pull from OCI registry
helm pull oci://registry-1.docker.io/bitnamicharts/nginx --version 15.0.0
```

---

## `helm dependency` — Manage Chart Dependencies

### `helm dependency update`

Download dependency charts into `charts/` and update `Chart.lock`.

```bash
helm dependency update ./my-chart
```

| Flag | Type | Description |
|---|---|---|
| `--keyring` | string | GPG keyring for verifying signed dependencies |
| `--skip-refresh` | bool | Skip `helm repo update` before resolving dependencies |
| `--verify` | bool | Verify dependency chart signatures |

**What happens internally:**
1. Runs `helm repo update` (unless `--skip-refresh`) to refresh repository indexes.
2. Reads `dependencies` from `Chart.yaml`.
3. For each dependency, resolves the semver constraint from `version` to an exact version.
4. Downloads the chart `.tgz` into `charts/`.
5. Generates/updates `Chart.lock` with resolved versions and digests.

### `helm dependency build`

Rebuild `Chart.lock` from `Chart.yaml` without re-downloading charts.

```bash
helm dependency build ./my-chart
```

| Flag | Type | Description |
|---|---|---|
| `--keyring` | string | GPG keyring for verifying signed dependencies |
| `--verify` | bool | Verify dependency chart signatures |

**Note:** If `Chart.lock` exists, `build` uses the locked versions. If absent, it resolves from `Chart.yaml`.

### `helm dependency list`

List all dependencies and their status.

```bash
helm dependency list ./my-chart
```

| Flag | Type | Description |
|---|---|---|
| `--max-col-width` | uint | Maximum column width (default 80) |

Example output:

```
NAME            VERSION     REPOSITORY                                  STATUS
redis           18.6.0      https://charts.bitnami.com/bitnami          ok
postgresql      15.4.0      https://charts.bitnami.com/bitnami          missing
```

Status values:
- `ok`: Dependency is present in `charts/`.
- `missing`: Dependency needs to be downloaded (`helm dependency update`).
- `unpacked`: Dependency is an unpacked directory in `charts/` instead of a `.tgz`.

---

## `helm lint` — Validate a Chart

### Syntax

```
helm lint [CHART_PATH] [flags]
```

### All Checks Performed

1. **Chart.yaml validation:** Required fields (`apiVersion`, `name`, `version`), valid types, valid semver.
2. **values.schema.json validation:** If present, validates `values.yaml` against the JSON Schema.
3. **Template rendering:** Renders all templates with default values to catch syntax errors.
4. **YAML parsing:** Each rendered template is parsed as valid YAML.
5. **Kubernetes object validation:** Checks for required fields (`apiVersion`, `kind`, `metadata.name`).
6. **GO template errors:** Invalid template expressions, missing functions, undefined values.
7. **Name length:** Warns if `.Release.Name` combined with chart name exceeds 53 characters.
8. **Icon URL validity:** Warns if the icon URL appears invalid.

### Flags

| Flag | Type | Description |
|---|---|---|
| `--debug` | bool | Enable verbose debug output (show rendered templates) |
| `-f`, `--values` | []string | Additional values files to use for linting |
| `--with-subcharts` | bool | Lint subcharts in `charts/` as well |
| `--set` | []string | Set values on the command line for linting |
| `--set-file` | []string | Set values from files for linting |
| `--set-string` | []string | Set STRING values for linting |
| `--strict` | bool | Treat warnings as errors |
| `-n`, `--namespace` | string | Namespace to use for linting (affects `.Release.Namespace`) |
| `-q`, `--quiet` | bool | Print only error/warning messages |

### Examples

```bash
# Basic lint
helm lint ./my-chart

# Lint with custom values
helm lint ./my-chart --values values-prod.yaml

# Lint with multiple values
helm lint ./my-chart -f values-base.yaml -f values-prod.yaml

# Lint with --set overrides
helm lint ./my-chart --set image.tag=1.27.0 --set replicaCount=5

# Lint with subcharts
helm lint ./my-chart --with-subcharts

# Strict mode
helm lint ./my-chart --strict

# Debug output
helm lint ./my-chart --debug
```

Example output:

```
==> Linting ./my-chart
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

**Production Note:** Run `helm lint --strict` in CI/CD pipelines. Fail the pipeline on any lint warning. This catches issues before deployment.

---

## `helm verify` — Verify a Signed Chart

### Syntax

```
helm verify [CHART_FILE] [flags]
```

### Flags

| Flag | Type | Description |
|---|---|---|
| `--keyring` | string | GPG keyring path (default `~/.gnupg/pubring.gpg`) |

### Example

```bash
# Verify a signed chart
helm verify ./nginx-15.0.0.tgz
```

Output on success:

```
Signed by: Bitnami <containers@bitnami.com>
Using Keychain: /home/user/.gnupg/pubring.gpg
Chart Hash Verified: sha256:abc123...
```

Output on failure:

```
Error: Chart has no provenance file
```

### Provenance Files

A provenance file (`.prov`) contains:
- A digital signature of the chart package
- Chart metadata for integrity verification
- The signing identity

To verify, Helm checks:
1. The signature is valid and trusted using the GPG keyring.
2. The chart hash in the provenance file matches the actual chart hash.

**Note:** Chart signing is optional. Most public repositories do not sign charts. Bitnami is a notable exception — all Bitnami charts are signed.

---

## `helm template` — Render Templates Locally

### Syntax

```
helm template [NAME] [CHART] [flags]
```

### Purpose

Render chart templates locally without contacting a Kubernetes cluster. Used for:
- Debugging template output
- Integrating with GitOps tools (ArgoCD, Flux)
- Pre-commit validation
- Generating manifests for `kubectl apply`

### Flags

| Flag | Type | Description |
|---|---|---|
| `-a`, `--api-versions` | []string | Kubernetes API versions used for `Capabilities.APIVersions` |
| `--ca-file` | string | CA certificate for TLS (when chart reference is from a repo) |
| `--cert-file` | string | Client certificate for TLS |
| `--create-namespace` | bool | Include resource for creating the namespace |
| `--dependency-update` | bool | Run `helm dependency update` before rendering |
| `--description` | string | Custom release description |
| `--devel` | bool | Include pre-release versions |
| `-s`, `--show-only` | []string | Show only specific templates (glob patterns) |
| `--debug` | bool | Enable debug output |
| `--dry-run` | bool | Simulate an install (same as `template`) |
| `--include-crds` | bool | Include CRDs in the output |
| `--insecure-skip-tls-verify` | bool | Skip TLS verification |
| `--is-upgrade` | bool | Set `.Release.IsUpgrade` to `true` instead of `false` |
| `--key-file` | string | Client private key for TLS |
| `--keyring` | string | GPG keyring for verification |
| `--kube-version` | string | Kubernetes version for `Capabilities.KubeVersion` |
| `--no-hooks` | bool | Omit hook resources from output |
| `--output-dir` | string | Write rendered templates to a directory instead of stdout |
| `--password` | string | Repository password |
| `-f`, `--values` | []string | Values files |
| `--set` | []string | Set values on command line |
| `--set-file` | []string | Set values from files |
| `--set-string` | []string | Set STRING values |
| `--skip-schema-validation` | bool | Skip JSON schema validation |
| `--username` | string | Repository username |
| `--validate` | bool | Validate rendered YAML against Kubernetes schemas (default true) |
| `--version` | string | Specific chart version |

### Examples

```bash
# Basic template rendering
helm template my-release ./my-chart

# Render with custom values
helm template my-release ./my-chart --values values-prod.yaml

# Render with --set overrides
helm template my-release ./my-chart \
  --set image.tag=1.27.0 \
  --set replicaCount=3

# Render with specific Kubernetes version
helm template my-release ./my-chart --kube-version 1.29

# Render with specific API versions available
helm template my-release ./my-chart \
  --api-versions networking.k8s.io/v1

# Show only specific templates
helm template my-release ./my-chart --show-only templates/deployment.yaml

# Show multiple templates with glob
helm template my-release ./my-chart \
  --show-only 'templates/{deployment,service}.yaml'

# Write to output directory (one file per template)
helm template my-release ./my-chart --output-dir ./rendered

# Render from a repository chart
helm template my-release bitnami/nginx --version 15.0.0 \
  --set service.type=NodePort

# Render with CRDs included
helm template my-release ./my-chart --include-crds

# Debug template (includes --debug and --dry-run)
helm template my-release ./my-chart --debug
```

**Production Note:** Use `helm template` with `--output-dir` in GitOps pipelines to generate manifests that can be committed to a config repository. ArgoCD and Flux consume these rendered manifests.

---

## Installing from Different Chart Sources

### From a Local Chart Directory

```bash
helm install my-release ./my-chart
```

Use during development. The chart is read directly from the filesystem.

### From a Packaged Chart (.tgz)

```bash
helm install my-release ./my-chart-0.1.0.tgz
```

Use for local distribution, air-gapped environments, and reproducible installs with a specific chart artifact.

### From a Repository

```bash
# Add the repository first
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install from the repository
helm install my-release bitnami/nginx --version 15.0.0
```

Use for standard production deployments. The chart is downloaded from the repository's index and cache.

### From an OCI Registry

```bash
# Login to the OCI registry
helm registry login registry.example.com

# Install from OCI
helm install my-release oci://registry.example.com/charts/nginx \
  --version 0.1.0
```

### From a URL (Direct Download)

```bash
helm install my-release https://charts.example.com/nginx-15.0.0.tgz
```

Use for one-off installs from a direct URL.

### Comparison

| Source | Use Case | Requires Repo Add? | Supports Version Constraint? |
|---|---|---|---|
| Local directory | Development, testing | No | No |
| Packaged .tgz | Distributable artifact | No | No |
| Repository | Standard production | Yes | Yes |
| OCI Registry | Modern artifact management | No (login required) | Yes (--version) |
| Direct URL | One-off, CI/CD | No | No (exact URL) |

---

## Versioning Charts

### Semantic Versioning (SemVer 2)

Chart versions follow the format `MAJOR.MINOR.PATCH`:

- **MAJOR**: Incompatible API changes, significant chart restructuring, breaking value changes.
- **MINOR**: New features, backward-compatible additions (new templates, new values).
- **PATCH**: Bug fixes, documentation updates, minor template fixes.

Example progression: `1.0.0` → `1.0.1` → `1.1.0` → `2.0.0`.

### Pre-Release Versions

Append a hyphen and pre-release identifier:

```
1.0.0-alpha
1.0.0-alpha.1
1.0.0-beta.2
1.0.0-rc.1
```

Pre-release versions are hidden from `helm search repo` unless `--devel` is used.

### Build Metadata

Append a plus sign and build metadata:

```
1.0.0+build.20240315
1.0.0-alpha.1+sha.abc123
```

Build metadata is ignored for version comparison purposes. `1.0.0+build.1` is considered equal to `1.0.0+build.2` for precedence.

### `appVersion` vs. `version`

- `version` (`Chart.yaml`): Version of the Helm chart itself. Changes when chart templates, values structure, or defaults change.
- `appVersion` (`Chart.yaml`): Version of the application the chart deploys. Changes when the container image version changes.

Example scenario:

| Case | version | appVersion |
|---|---|---|
| Chart bug fix, same app | `1.0.1` (patch) | `1.26.0` (unchanged) |
| New chart feature, same app | `1.1.0` (minor) | `1.26.0` (unchanged) |
| App version bump, chart unchanged | `1.1.1` (patch) | `1.27.0` (changed) |
| Major chart restructure + new app | `2.0.0` (major) | `2.0.0` (changed) |

**Production Note:** Bump `version` on every chart change, even for documentation fixes. This ensures `helm upgrade` recognizes that a new chart version is available. Use `--version` to pin exact chart versions in CI/CD.

---

## Best Practices for Chart Development

### 1. Use Standard Labels

Always include Kubernetes recommended labels:

```yaml
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ include "mychart.chart" . }}
```

### 2. Make Charts Composable

- Use `condition` and `tags` on dependencies so users can enable/disable optional features.
- Use `ingress.enabled`, `persistence.enabled`, `autoscaling.enabled` booleans in `values.yaml`.

### 3. Provide Sensible Defaults

- Set CPU/memory `requests` and `limits` to reasonable defaults.
- Set security contexts: `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`.
- Default `pullPolicy` to `IfNotPresent`, not `Always`.

### 4. Use `_helpers.tpl` for Reusable Logic

- Never duplicate template logic.
- Define resource names, labels, and selectors in helpers.
- Use helpers for common patterns like image pull secrets, volumes, and environment variables.

### 5. Validate Values

- Provide a `values.schema.json` for all production charts.
- Use `required` in templates for mandatory values (e.g., `image.repository`).
- Use `fail` for unrecoverable configuration errors.

### 6. Support Multiple Kubernetes Versions

- Use `Capabilities.KubeVersion` to conditionally include/exclude API versions.
- Check `Capabilities.APIVersions` to detect available APIs.
- Set `kubeVersion` in `Chart.yaml` to declare compatibility.

### 7. Write Tests

Every chart should have at least one test:

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded,before-hook-creation
spec:
  containers:
    - name: test
      image: busybox
      command: ['wget']
      args: ['{{ include "mychart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

### 8. Use NOTES.txt for Post-Install Instructions

- Tell users how to access the application.
- Display generated passwords with a warning to change them.
- Show relevant `kubectl` commands for common tasks.
- Include links to documentation.

### 9. Git-Friendly Chart Management

- Commit `Chart.yaml` and `values.yaml` — never commit `values-prod.yaml` with secrets.
- Commit `Chart.lock` for reproducible builds.
- Do NOT commit `charts/*.tgz` — these are generated artifacts. Add them to `.gitignore`.
- Do NOT commit rendered template output directories.

### 10. CI/CD Integration

```bash
# Pre-commit / PR checks
helm lint ./my-chart --strict
helm template test-release ./my-chart --debug
helm dependency update ./my-chart
helm package ./my-chart

# Release pipeline
helm package ./my-chart --dependency-update --version "$VERSION"
helm push ./my-chart-$VERSION.tgz oci://registry.example.com/charts
```

### 11. Documentation

- Include a comprehensive `README.md` with:
  - Chart description and use case
  - Prerequisites (Kubernetes version, required CRDs)
  - All configurable values with defaults and descriptions
  - Installation instructions with examples
  - Upgrade instructions and migration notes
- Add `annotations` with Artifact Hub metadata:

```yaml
annotations:
  artifacthub.io/changes: |
    - Added horizontal pod autoscaling support
    - Fixed service port mapping
  artifacthub.io/images: |
    - name: nginx
      image: nginx:1.26.0
  artifacthub.io/license: Apache-2.0
```

### 12. Security

- Never hardcode secrets in `values.yaml`. Use placeholders or external secret management.
- Use `helm-secrets` plugin for encrypted values.
- Sign production charts and verify signatures before deployment.
- Scan container images referenced in charts for vulnerabilities.
- Set restrictive `securityContext` defaults.
- Avoid running containers as root.
- Use `readOnlyRootFilesystem: true` when possible.

### 13. Test Across Versions

Test chart installation across multiple Kubernetes versions:

```bash
# Using kind (Kubernetes in Docker)
kind create cluster --name test-1.29 --image kindest/node:v1.29.0
helm install test-release ./my-chart --kube-context kind-test-1.29
helm test test-release
kind delete cluster --name test-1.29
```

### 14. Handle Upgrades Gracefully

- Use `helm.sh/hook` annotations for migration jobs (`pre-upgrade`, `post-upgrade`).
- Avoid breaking changes in `values.yaml` structure without a major version bump.
- Document migration steps when values structure changes.
- Test upgrades from the previous version to the new version:

```bash
# Install previous version
helm install test-release ./my-chart --version 1.0.0

# Upgrade to new version
helm upgrade test-release ./my-chart --version 1.1.0
```

**Production Note:** The most common chart-related production issue is a failed upgrade due to immutable field changes (e.g., changing a Deployment's selector). Always test upgrades in a staging environment before deploying to production.
