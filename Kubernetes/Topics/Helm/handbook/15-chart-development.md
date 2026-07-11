# Chapter 15: Chart Development

Chart development is the core skill in the Helm ecosystem. This chapter provides a comprehensive guide to creating, structuring, testing, distributing, and maintaining production-quality Helm charts.

---

## Chart Creation — helm create Scaffolding

### Generating a New Chart

```bash
helm create mychart

# Resulting structure:
# mychart/
# ├── .helmignore
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# └── templates/
#     ├── NOTES.txt
#     ├── _helpers.tpl
#     ├── deployment.yaml
#     ├── service.yaml
#     ├── hpa.yaml
#     ├── ingress.yaml
#     ├── serviceaccount.yaml
#     └── tests/
#         └── test-connection.yaml
```

### Scaffolding Walkthrough

Each generated file has a specific purpose:

| File | Purpose | What to Modify |
|------|---------|----------------|
| `.helmignore` | Exclude files from the packaged chart (like `.gitignore`). | Add patterns for local development files, CI configs, and secrets. |
| `Chart.yaml` | Chart metadata: name, version, description, dependencies. | Update name, version, description, maintainers, dependencies. |
| `values.yaml` | Default configuration values. | Define every configurable parameter with sensible defaults and comments. |
| `charts/` | Directory for manually vendored or auto-downloaded dependencies. | Populated by `helm dependency update`. Rarely edited manually. |
| `templates/NOTES.txt` | Post-install/post-upgrade message shown to users. | Write clear instructions: how to access the app, credentials, next steps. |
| `templates/_helpers.tpl` | Reusable named templates (partials) for labels, selectors, and common patterns. | Define `mychart.labels`, `mychart.selectorLabels`, `mychart.name`, `mychart.fullname`. |
| `templates/deployment.yaml` | Kubernetes Deployment manifest template. | Customize for your application container, probes, resources, volumes. |
| `templates/service.yaml` | Kubernetes Service manifest template. | Set service type, ports, annotations. |
| `templates/hpa.yaml` | HorizontalPodAutoscaler template. | Conditionally enable via values. |
| `templates/ingress.yaml` | Ingress resource template. | Conditionally enable via values, configure TLS, hosts. |
| `templates/serviceaccount.yaml` | ServiceAccount template. | Conditionally create, control automated mounting. |
| `templates/tests/` | Helm test pod definitions. | Write meaningful integration tests for your application. |

### Modifying Generated Files

```yaml
# values.yaml — add your application-specific configuration:
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

imagePullSecrets: []

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: myapp.example.com
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

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# Add your own custom configuration sections:
app:
  config:
    logLevel: info
    maxConnections: 1000
  features:
    newUI: false
    experimentalCache: false
```

---

## Forking Charts — Working with Community Charts

### When to Fork

| Scenario | Fork? | Alternative |
|----------|-------|-------------|
| Minor customization (labels, annotations) | No | Use `--set`, values files, or post-renderer. |
| Moderate changes (new template file, modified probes) | No | Submit a PR upstream or maintain a wrapper chart. |
| Major changes (restructuring templates, new features) | **Yes** | Fork, customize, and maintain independently. |
| Upstream is unmaintained or abandoned | **Yes** | Fork and continue maintenance. |
| Organization policy requires internal-only charts | **Yes** | Fork, audit, and host internally. |

### Maintaining Forks

```bash
# Clone the upstream chart
git clone https://github.com/bitnami/charts.git
cd charts/bitnami/nginx

# Create your fork
git remote add myorg git@github.com:myorg/helm-nginx.git
git push myorg main

# Periodically sync from upstream
git fetch upstream
git merge upstream/main

# Resolve conflicts manually
# Test the merged chart thoroughly
helm lint ./ --strict
helm template test-release ./ | kubectl apply --dry-run=client -f -
```

### Upstream Sync Strategies

| Strategy | Description | Effort | Risk |
|----------|-------------|--------|------|
| Periodic merge | Merge upstream changes on a schedule (e.g., monthly). | Medium | Merge conflicts. |
| Cherry-pick | Pick specific upstream fixes or features. | Low | May miss important security patches. |
| Rewrite | Completely rewrite the chart based on upstream patterns. | High | Lowest dependency on upstream. Full control. |
| Wrapper chart | Create a parent chart that depends on the community chart as a subchart. | Low | Limited customization. Cannot change subchart templates. |

---

## Chart Structure Best Practices

### Directory Layout

```
mychart/
├── .helmignore              # Files to exclude from package
├── Chart.yaml               # Chart metadata
├── Chart.lock               # Lock file (auto-generated, committed to VCS for reproducibility)
├── values.yaml              # Default values with extensive comments
├── values.schema.json       # JSON Schema for values validation
├── README.md                # Chart documentation
├── CONTRIBUTING.md          # (optional) Contribution guidelines
├── CHANGELOG.md             # (optional) Version history
├── LICENSE                  # (optional) License file
├── crds/                    # Custom Resource Definitions (applied before templates)
├── templates/               # All Kubernetes resource templates
│   ├── NOTES.txt            # Post-install message
│   ├── _helpers.tpl         # Named template partials
│   ├── deployment.yaml      # One resource per file
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── pvc.yaml
│   ├── serviceaccount.yaml
│   ├── role.yaml
│   ├── rolebinding.yaml
│   ├── networkpolicy.yaml
│   ├── poddisruptionbudget.yaml
│   └── tests/
│       └── test-connection.yaml
└── charts/                  # Subchart dependencies (populated by helm dep update)
```

### File Naming Conventions

| Convention | Example | Rationale |
|------------|---------|-----------|
| Resource type as filename | `deployment.yaml`, `service.yaml` | Self-documenting; easy to find resources. |
| Use hyphens, not underscores | `network-policy.yaml`, NOT `network_policy.yaml` | Consistent with Kubernetes naming. |
| `_` prefix for partials/helpers | `_helpers.tpl`, `_pod.tpl` | Template engine skips files starting with `_`. |
| `NOTES.txt` in templates/ | `templates/NOTES.txt` | Rendered after install/upgrade; only this exact name is used. |
| `tests/` subdirectory | `templates/tests/test-*.yaml` | Tests are executed by `helm test`; names must start with `test-`. |

### Template Organization

```
templates/
├── _helpers.tpl             # All global helpers, labels, selectors
├── _capabilities.tpl        # Helper functions for .Capabilities checks
├── deployment.yaml          # One Kubernetes resource per file
├── service.yaml
├── ingress.yaml
├── configmap.yaml           # Application configuration
├── secret.yaml              # Secrets (references ExternalSecret or Vault, not raw values)
├── serviceaccount.yaml
├── role.yaml
├── rolebinding.yaml
└── tests/
    └── test-connection.yaml
```

---

## Subcharts

### What Are Subcharts

A subchart is a Helm chart that is a dependency of another chart. It is installed, upgraded, and rolled back as part of the parent release.

```
Parent Chart (my-app)
    ├── subchart: PostgreSQL (bitnami/postgresql)
    ├── subchart: Redis (bitnami/redis)
    └── subchart: Nginx (bitnami/nginx)
```

### How Subcharts Work

1. Declared in `Chart.yaml` under `dependencies:`.
2. Downloaded to `charts/` by `helm dependency update`.
3. Rendered and applied as part of the parent release.
4. Values are namespaced under the subchart's name.

```yaml
# parent-chart/Chart.yaml
apiVersion: v2
name: my-app
version: 1.0.0
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    condition: postgresql.enabled
  - name: redis
    version: "18.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    condition: redis.enabled
```

```yaml
# parent-chart/values.yaml
postgresql:
  enabled: true
  auth:
    username: myapp
    database: myapp
    password: changeme

redis:
  enabled: true
  architecture: standalone
  auth:
    password: changeme
```

### Parent-Child Relationship

- The parent chart's values are **not** automatically available to subcharts.
- Subcharts have their own `values.yaml` merged with the parent's values under the subchart's key.
- Subchart values are isolated; a subchart cannot read another subchart's values without explicit import.
- Subcharts are released as part of the parent; there is no independent release for a subchart.

### Global Values

Global values are accessible by **all** charts (parent and all subcharts) in the release:

```yaml
# parent values.yaml
global:
  imageRegistry: my-registry.example.com
  imagePullSecrets:
    - name: regcred
  storageClass: fast-ssd
  environment: production

postgresql:
  # Subchart can access global values via .Values.global
  primary:
    persistence:
      storageClass: ""  # Uses global.storageClass as default
```

In the subchart template:

```yaml
# In postgresql subchart template:
image: "{{ .Values.global.imageRegistry }}/postgresql:15"
imagePullSecrets:
  - name: {{ .Values.global.imagePullSecrets | first }}
```

---

## Chart Dependencies

### Chart.yaml Dependencies Section

```yaml
apiVersion: v2
name: my-app
version: 1.0.0
dependencies:
  - name: postgresql
    version: ">=15.0.0 <16.0.0"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    condition: postgresql.enabled
    tags:
      - database
    alias: main-db
    import-values:
      - child: service.ports.postgresql
        parent: dbPort

  - name: postgresql
    version: ">=15.0.0 <16.0.0"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    condition: analyticsDb.enabled
    tags:
      - analytics
    alias: analytics-db

  - name: common
    version: "2.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    tags:
      - library
```

### Version Constraints (SemVer Ranges)

| Constraint | Meaning | Example Matches |
|------------|---------|-----------------|
| `15.0.0` | Exact version | Only `15.0.0` |
| `>=15.0.0 <16.0.0` | Range | `15.0.0`, `15.5.3`, `15.9.9` |
| `~15.0.0` | Patch-level changes only | `15.0.0`, `15.0.1`, `15.0.99` (not `15.1.0`) |
| `^15.0.0` | Compatible changes (minor + patch) | `15.0.0`, `15.5.0`, `15.9.9` (not `16.0.0`) |
| `15.x.x` | Wildcard minor and patch | Any `15` major version |
| `*` or `>=0.0.0` | Any version | All versions |

**Production Note:** Always pin to a specific minor or patch range (e.g., `~15.5.0` or `>=15.0.0 <16.0.0`). Using `*` or loose ranges can result in surprise breaking changes when `helm dependency update` pulls a new version.

### Repository References

```yaml
# HTTP repository
repository: "https://charts.bitnami.com/bitnami"

# OCI repository (Helm 3.8+)
repository: "oci://registry-1.docker.io/bitnamicharts"

# Local file path (for development)
repository: "file://../my-local-chart"

# Alias for a repo already added via helm repo add
# The @ syntax references a repo by its local alias:
repository: "@bitnami"
```

---

## Condition and Tags

### Conditions

A condition controls whether a dependency is included based on a values path:

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    condition: postgresql.enabled
```

```yaml
# values.yaml
postgresql:
  enabled: true   # When true: dependency is included
                  # When false: dependency is skipped
```

The condition path is evaluated against the parent chart's values. The last element of the path must be a boolean.

### Tags

Tags provide an alternative, group-based way to control dependencies:

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    tags:
      - database
      - persistence
  - name: redis
    tags:
      - cache
      - persistence
```

```yaml
# values.yaml
tags:
  database: true      # Enables postgresql
  cache: false        # Disables redis
  persistence: false  # Overrides both, disabling both postgresql AND redis
```

**Note:** Tags are evaluated first, then conditions. If a tag resolves to `false`, the condition is not evaluated. Tag resolution: if **any** tag in the list is `true`, the chart is enabled (OR logic). If **all** tags are `false`, the chart is disabled.

### Resolution Order

```
1. Evaluate tags:
   - If ANY tag is true → skip to step 3 (chart is ENABLED)
   - If ALL tags are false → chart is DISABLED

2. If tags are not set or don't resolve to false:
   - Evaluate condition:
     - If condition path exists and is false → DISABLED
     - If condition path exists and is true → ENABLED
     - If condition path does not exist → ENABLED (default)

3. Result
```

---

## Aliases — Multiple Instances of the Same Dependency

Aliases allow you to depend on the same chart multiple times with different configurations:

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    alias: main-db

  - name: postgresql
    version: "15.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    alias: analytics-db

  - name: postgresql
    version: "15.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    alias: cache-db
```

```yaml
# values.yaml — each alias gets its own configuration namespace:
main-db:
  auth:
    database: main_app
    username: main_user
    password: main_password
  primary:
    persistence:
      size: 50Gi

analytics-db:
  auth:
    database: analytics
    username: analytics_user
    password: analytics_password
  primary:
    persistence:
      size: 200Gi

cache-db:
  auth:
    database: cache
    username: cache_user
    password: cache_password
  primary:
    persistence:
      enabled: false  # Ephemeral cache DB
```

**Note:** Without an alias, if you depend on the same chart twice, the second entry overwrites the first. Aliases create independent instances with independent configurations, templates prefixed by the alias name, and no shared state.

---

## Library Charts

### What Are Library Charts

A library chart is a chart with `type: library` in `Chart.yaml`. It contains no Kubernetes resource templates — only template helpers (named templates) that other charts can import and use.

```yaml
# Chart.yaml for a library chart
apiVersion: v2
name: my-lib
version: 1.0.0
type: library
description: Shared template helpers for my organization
```

### How They Differ from Application Charts

| Feature | Application Chart | Library Chart |
|---------|-------------------|---------------|
| `type` | `application` (default) | `library` |
| Installable | Yes | No (cannot be installed as a release) |
| Templates directory | Contains Kubernetes resource templates | Contains only `_*.tpl` helper files |
| Output | Rendered Kubernetes manifests | No rendered output (only functions) |
| Dependency usage | As a deployed service | As a set of reusable template functions |
| Versioning | Normal SemVer | Normal SemVer |

### Use Cases

- Standardizing labels and selectors across all organization charts.
- Standardizing resource configurations (limits, requests, probes).
- Providing common patterns (ingress annotations, security contexts).
- Sharing complex template logic (certificate management, DNS naming conventions).

### Creating a Library Chart

```yaml
# my-lib/Chart.yaml
apiVersion: v2
name: my-lib
version: 1.0.0
type: library
```

```yaml
# my-lib/templates/_labels.tpl
{{/* Generate standard labels for all organization resources */}}
{{- define "my-lib.labels" -}}
app.kubernetes.io/name: {{ include "my-lib.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ include "my-lib.chart" . }}
{{- end -}}

{{- define "my-lib.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-lib.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{- define "my-lib.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "my-lib.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

```yaml
# my-lib/templates/_resources.tpl
{{- define "my-lib.resources" -}}
{{- with .Values.resources }}
resources:
  {{- toYaml . | nindent 2 }}
{{- else }}
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
{{- end }}
{{- end -}}

{{- define "my-lib.probes" -}}
{{- if .Values.probes.liveness }}
livenessProbe:
  httpGet:
    path: {{ .Values.probes.livenessPath | default "/healthz" }}
    port: {{ .Values.service.port }}
  initialDelaySeconds: {{ .Values.probes.initialDelaySeconds | default 30 }}
  periodSeconds: {{ .Values.probes.periodSeconds | default 10 }}
{{- end }}
{{- if .Values.probes.readiness }}
readinessProbe:
  httpGet:
    path: {{ .Values.probes.readinessPath | default "/ready" }}
    port: {{ .Values.service.port }}
  initialDelaySeconds: {{ .Values.probes.initialDelaySeconds | default 5 }}
  periodSeconds: {{ .Values.probes.periodSeconds | default 5 }}
{{- end }}
{{- end -}}
```

### Using a Library Chart

```yaml
# application-chart/Chart.yaml
apiVersion: v2
name: my-app
version: 1.0.0
dependencies:
  - name: my-lib
    version: "1.x.x"
    repository: "oci://my-registry.example.com/charts"
```

```yaml
# application-chart/templates/deployment.yaml — using library templates:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-lib.name" . }}
  labels:
    {{- include "my-lib.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-lib.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-lib.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          {{- include "my-lib.resources" . | nindent 10 }}
          {{- include "my-lib.probes" . | nindent 10 }}
```

---

## Importing Child Values

The `import-values` directive exposes subchart values to the parent chart:

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    import-values:
      # Import a specific child value as a parent value
      - child: service.ports.postgresql
        parent: dbPort

      # Import all values under a child path as parent values
      - child: auth
        parent: dbAuth

      # Import all exports from the child chart
      - child: exports
        parent: imports
```

```yaml
# After import, the parent can reference:
# .Values.dbPort -> the postgresql service port
# .Values.dbAuth.username -> the postgresql auth username
# .Values.dbAuth.password -> the postgresql auth password
```

### Subchart Exports

The subchart must explicitly export values for them to be importable:

```yaml
# In postgresql/values.yaml:
exports:
  database: myapp
  port: 5432
  username: myuser
```

---

## Global Values — Sharing Across All Charts

```yaml
# parent values.yaml
global:
  imageRegistry: my-registry.example.com
  imagePullSecrets:
    - name: regcred
  storageClass: ssd
  environment: production
  commonLabels:
    team: platform
    cost-center: engineering
```

In any chart (parent or subchart):

```yaml
# Accessing global values:
image: "{{ .Values.global.imageRegistry }}/myapp:{{ .Chart.AppVersion }}"
imagePullSecrets:
  {{- range .Values.global.imagePullSecrets }}
  - name: {{ .name }}
  {{- end }}
```

**Warning:** Overusing globals creates tight coupling between charts. A subchart that depends on `global.imageRegistry` cannot be used independently. Reserve globals for truly cross-cutting concerns (image registries, environment names, organization labels).

---

## Chart Versioning

### Semantic Versioning

```
 MAJOR . MINOR . PATCH - PRE-RELEASE + BUILD
   1   .   2   .   3   - alpha.1    + 20240101

| Change Type         | Version Bump | Example          |
|---------------------|-------------|------------------|
| Backward-incompatible | MAJOR      | 1.2.3 → 2.0.0   |
| New feature (backward-compatible) | MINOR | 1.2.3 → 1.3.0 |
| Bug fix (backward-compatible) | PATCH   | 1.2.3 → 1.2.4   |
| Pre-release         | PRE-RELEASE  | 1.2.3 → 1.2.3-rc.1 |
| Build metadata      | BUILD        | 1.2.3+20240101    |
```

### Version Bumping Strategies

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| Manual | Update `version` in `Chart.yaml` before each release. | Small teams, infrequent releases. |
| CI-triggered | CI pipeline bumps version based on commit messages (conventional commits). | Automated release pipelines. |
| Git tag-based | Use `git describe` to derive the version. | GitOps workflows. |
| Chart Releaser | Use `chart-releaser` tool to automatically detect version changes and create GitHub Releases. | GitHub-hosted chart repositories. |

```bash
# Manual version bump
sed -i 's/^version: .*/version: 1.3.0/' Chart.yaml
helm package .
helm push mychart-1.3.0.tgz oci://my-registry.example.com/charts
```

---

## Chart Documentation

### README.md

Every chart should have a comprehensive `README.md`:

```markdown
# MyApp Helm Chart

A Helm chart for deploying MyApp on Kubernetes.

## TL;DR

helm repo add myrepo https://charts.example.com
helm install my-release myrepo/myapp

## Prerequisites

- Kubernetes 1.25+
- Helm 3.12+
- PV provisioner support (if using persistence)

## Installing the Chart

helm install my-release myrepo/myapp --values ./custom-values.yaml

## Uninstalling the Chart

helm uninstall my-release

## Parameters

### Global parameters

| Name | Description | Value |
|------|-------------|-------|
| `global.imageRegistry` | Global Docker image registry | `""` |
| `global.storageClass` | Global StorageClass for PVCs | `""` |

### Common parameters

| Name | Description | Value |
|------|-------------|-------|
| `replicaCount` | Number of replicas | `3` |
| `image.repository` | Image repository | `nginx` |
| `image.tag` | Image tag | `1.25` |

## Upgrading

### From 1.x to 2.x

Breaking changes:
- The `service.port` parameter has been renamed to `service.ports.http`.
- Ingress API version updated from `networking.k8s.io/v1beta1` to `networking.k8s.io/v1`.

Migration steps:
1. Update `service.port` to `service.ports.http` in your values file.
2. If using Ingress, ensure your Kubernetes cluster is 1.19+.
3. Test with `helm upgrade --dry-run` before proceeding.
```

### values.yaml Comments

```yaml
# -- Number of replicas to deploy
replicaCount: 3

image:
  # -- Image repository (without tag)
  repository: nginx
  # -- Image tag (immutable tags recommended)
  tag: "1.25.3"
  # -- Image pull policy
  pullPolicy: IfNotPresent

service:
  # -- Service type
  type: ClusterIP
  # -- Service port
  port: 80

# -- Resource requests and limits
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# -- Pod annotations
podAnnotations: {}

# -- Node selector for pod scheduling
nodeSelector: {}

# -- Tolerations for pod scheduling
tolerations: []

# -- Affinity rules for pod scheduling
affinity: {}
```

The `# --` convention is used by tools like `helm-docs` to auto-generate parameter documentation tables.

### NOTES.txt

```yaml
Thank you for installing {{ .Chart.Name }}!

Your release is named {{ .Release.Name }}.

To access the application:

{{- if .Values.ingress.enabled }}
  Visit: https://{{ index .Values.ingress.hosts 0 "host" }}
{{- else if eq .Values.service.type "LoadBalancer" }}
  Get the LoadBalancer IP:
  export SERVICE_IP=$(kubectl get svc {{ include "mychart.fullname" . }} \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if eq .Values.service.type "NodePort" }}
  Get the NodePort:
  export NODE_PORT=$(kubectl get svc {{ include "mychart.fullname" . }} \
    -o jsonpath='{.spec.ports[0].nodePort}')
  echo http://<NODE_IP>:$NODE_PORT
{{- else }}
  Port forward:
  kubectl port-forward svc/{{ include "mychart.fullname" . }} \
    8080:{{ .Values.service.port }}
  echo http://localhost:8080
{{- end }}

{{- if .Values.config.adminPassword }}
Administrator credentials:
  Username: admin
  Password: {{ .Values.config.adminPassword }}
{{- end }}

For more information, visit: https://github.com/myorg/mychart
```

### JSON Schema (values.schema.json)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Values",
  "type": "object",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "description": "Number of replicas"
    },
    "image": {
      "type": "object",
      "properties": {
        "repository": {
          "type": "string",
          "minLength": 1
        },
        "tag": {
          "type": "string",
          "minLength": 1,
          "default": "latest"
        }
      },
      "required": ["repository"]
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer"]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    }
  },
  "required": ["replicaCount", "image"]
}
```

**Note:** The `values.schema.json` file provides input validation when users run `helm install`, `helm upgrade`, or `helm lint`. It catches type errors and missing required values before templates are rendered.

---

## Chart Testing

### Test Pods

Test pods run as Kubernetes Pods (typically Jobs) annotated with `"helm.sh/hook": test`:

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
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  containers:
    - name: wget
      image: busybox:1.36
      command:
        - wget
        - -q
        - -O
        - /dev/null
        - "{{ include "mychart.fullname" . }}:{{ .Values.service.port }}"
  restartPolicy: Never
```

```bash
# Run all tests
helm test my-release

# Run tests with timeout
helm test my-release --timeout 5m

# Run tests and see logs on failure
helm test my-release --logs

# Filter specific tests
helm test my-release --filter "test-connection"
```

### helm-unittest Plugin — Unit Testing Templates

```bash
# Install the plugin
helm plugin install https://github.com/helm-unittest/helm-unittest

# Create test files in tests/ directory
```

```yaml
# tests/deployment_test.yaml
suite: Deployment
templates:
  - deployment.yaml
tests:
  - it: should have default name
    asserts:
      - equal:
          path: metadata.name
          value: RELEASE-NAME-mychart

  - it: should set replicas from values
    set:
      replicaCount: 5
    asserts:
      - equal:
          path: spec.replicas
          value: 5

  - it: should set correct image
    set:
      image:
        repository: myrepo/myapp
        tag: "2.0.0"
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: myrepo/myapp:2.0.0

  - it: should have liveness probe when enabled
    set:
      probes:
        liveness: true
    asserts:
      - exists:
          path: spec.template.spec.containers[0].livenessProbe

  - it: should fail when image.repository is empty
    set:
      image:
        repository: ""
    asserts:
      - failedTemplate:
          errorMessage: "image.repository is required"
```

```bash
# Run unit tests
helm unittest ./mychart

# Run specific test suite
helm unittest ./mychart --suite Deployment

# Run with color output
helm unittest --color ./mychart

# Update test snapshots
helm unittest --update-snapshot ./mychart
```

---

## Chart Distribution

### Packaging

```bash
# Package a chart into a .tgz archive
helm package ./mychart

# Output: mychart-1.0.0.tgz

# Package to a specific directory
helm package ./mychart --destination ./releases

# Sign the package with GPG
helm package ./mychart --sign --key 'mykey@example.com' --keyring ~/.gnupg/secring.gpg

# Verify a signed package
helm verify mychart-1.0.0.tgz
```

### Repository Hosting

```bash
# Option 1: Traditional chart repository (ChartMuseum, Harbor, etc.)
# Create index.yaml
helm repo index ./releases --url https://charts.example.com

# Upload to static hosting (S3, GCS, Nginx)
aws s3 sync ./releases s3://my-helm-charts/

# Add the repository
helm repo add myrepo https://charts.example.com

# Option 2: OCI registry (recommended for Helm 3.8+)
helm package ./mychart
helm push mychart-1.0.0.tgz oci://registry-1.docker.io/myorg

# Install from OCI
helm install my-release oci://registry-1.docker.io/myorg/mychart --version 1.0.0
```

### Artifact Hub Publishing

1. Add an `artifacthub.io` annotation section to `Chart.yaml`:

```yaml
annotations:
  artifacthub.io/license: Apache-2.0
  artifacthub.io/links: |
    - name: Documentation
      url: https://docs.myapp.com
    - name: Source Code
      url: https://github.com/myorg/myapp
  artifacthub.io/maintainers: |
    - name: Platform Team
      email: platform@example.com
  artifacthub.io/changes: |
    - Added horizontal pod autoscaling support
    - Fixed ingress path type default
  artifacthub.io/signKey: |
    fingerprint: ABC123...
    url: https://keys.example.com/pgp-key.asc
  artifacthub.io/prerelease: "false"
```

2. Push the chart to a public repository.
3. Artifact Hub automatically indexes charts from registered repositories.

---

## Chart Maintenance

### Deprecation

```yaml
# Chart.yaml
deprecated: true
```

When a chart is marked deprecated:
- `helm search repo` shows a deprecation warning.
- `helm install` still works (no enforcement).
- This is an informational signal to users.

### Dependency Updates

```bash
# Update all dependencies to latest compatible versions
helm dependency update ./mychart

# Check for available updates
helm dependency list ./mychart

# Force update and rebuild lock file
helm dependency update ./mychart --skip-refresh && helm dependency build ./mychart
```

### Breaking Changes — Migration Guides

When introducing breaking changes:

1. **Bump the major version** in `Chart.yaml`.
2. **Document the breaking changes** in `README.md` under an "Upgrading" section.
3. **Provide a migration path** — scripts, step-by-step instructions, or a values migration tool.
4. **Announce deprecation** in the previous major version with a notice in `NOTES.txt`.
5. **Test the migration** in CI against the previous chart version.

```markdown
## Upgrading from 1.x to 2.0.0

### Breaking Changes
- `service.port` renamed to `service.ports.http`
- `persistence.enabled` defaulted to `false` (was `true`)
- Ingress API requires `networking.k8s.io/v1` (Kubernetes 1.19+)

### Migration
1. Rename `service.port` to `service.ports.http` in your values file.
2. Set `persistence.enabled: true` if you need persistent storage.
3. Verify Kubernetes version supports `networking.k8s.io/v1`.
4. Run `helm upgrade my-release myrepo/mychart --version 2.0.0 --dry-run` first.
```

---

## Template Conventions

### Naming

```yaml
# _helpers.tpl — standard naming template:
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}
```

### Variable Naming

| Scope | Convention | Example |
|-------|------------|---------|
| Template variables (local) | `$camelCase` | `$podName`, `$servicePort` |
| Values keys | `camelCase` | `replicaCount`, `imagePullPolicy` |
| Kubernetes resource names | `lowercase-hyphenated` | `my-release-mychart` |
| Labels | `lowercase.with.dots` | `app.kubernetes.io/name` |

### _helpers.tpl Patterns

```yaml
{{/* Full resource name */}}
{{- define "mychart.fullname" -}}
...
{{- end -}}

{{/* Common labels */}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}

{{/* Selector labels (must match spec.selector.matchLabels) */}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{/* Chart name and version */}}
{{- define "mychart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end -}}

{{/* Service account name */}}
{{- define "mychart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "mychart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end -}}

{{/* Image pull secrets helper */}}
{{- define "mychart.imagePullSecrets" -}}
{{- with .Values.imagePullSecrets }}
imagePullSecrets:
{{- toYaml . | nindent 2 }}
{{- end }}
{{- with .Values.global.imagePullSecrets }}
imagePullSecrets:
{{- toYaml . | nindent 2 }}
{{- end }}
{{- end -}}
```

---

## Avoiding Template Bloat

### One Resource Per File Convention

```
CORRECT:
  templates/deployment.yaml
  templates/service.yaml
  templates/configmap.yaml
  templates/secret.yaml

WRONG (template bloat):
  templates/all-resources.yaml  # Contains Deployment, Service, ConfigMap, Secret
```

**Benefits:**
- Easier to find and modify specific resources.
- Template errors report the exact file.
- `--show-only` can target specific resources.
- Merge conflicts are scoped to individual resources.
- Code reviews are focused on relevant changes.

### When to Split Templates

| Indicator | Action |
|-----------|--------|
| Template file is > 200 lines | Split into logical groups: `deployment.yaml` + `deployment-volumes.yaml`. |
| Multiple resources in one file | Split each resource into its own file. |
| Repeated blocks across files | Move to `_helpers.tpl` as a named template. |
| Complex conditional logic (> 3 nesting levels) | Extract to a helper function. |
| Same resource type, different conditions | One file per configuration variant (e.g., `ingress-nginx.yaml`, `ingress-traefik.yaml`). |

### Using Named Templates Effectively

```yaml
# Instead of repeating this in every file:
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}
{{- with .Values.affinity }}
affinity:
  {{- toYaml . | nindent 2 }}
{{- end }}
{{- with .Values.tolerations }}
tolerations:
  {{- toYaml . | nindent 2 }}
{{- end }}

# Define once in _helpers.tpl:
{{- define "mychart.podScheduling" -}}
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}
{{- with .Values.affinity }}
affinity:
  {{- toYaml . | nindent 2 }}
{{- end }}
{{- with .Values.tolerations }}
tolerations:
  {{- toYaml . | nindent 2 }}
{{- end }}
{{- end -}}

# Use everywhere:
{{- include "mychart.podScheduling" . | nindent 6 }}
```

---

## Chart Compatibility — kubeVersion and Capabilities

### kubeVersion Field

```yaml
# Chart.yaml
kubeVersion: ">=1.25.0-0 <1.30.0-0"
```

This is a SemVer range that prevents installation on incompatible Kubernetes versions:

```bash
# Installing on old cluster:
$ helm install my-release ./mychart
Error: chart requires Kubernetes version >=1.25.0-0 <1.30.0-0
```

| Constraint | Usage |
|------------|-------|
| `>=1.25.0-0` | Minimum Kubernetes version (inclusive). The `-0` suffix accounts for pre-release versions. |
| `<1.30.0-0` | Maximum Kubernetes version (exclusive). |
| `>=1.25.0-0 <1.30.0-0` | Range: works on 1.25, 1.26, 1.27, 1.28, 1.29. |

### Capabilities Object

The `.Capabilities` built-in object provides cluster information at render time:

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `.Capabilities.KubeVersion` | object | Kubernetes version with `.Major`, `.Minor`, `.GitVersion` | `v1.29.2` |
| `.Capabilities.APIVersions` | list | All API versions available on the cluster | `["apps/v1", "networking.k8s.io/v1", ...]` |
| `.Capabilities.HelmVersion` | object | Helm version with `.Version`, `.GitCommit`, `.GitTreeState`, `.GoVersion` | `v3.14.0` |

```yaml
# Conditional rendering based on API versions:
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
apiVersion: networking.k8s.io/v1
{{- else if .Capabilities.APIVersions.Has "networking.k8s.io/v1beta1" }}
apiVersion: networking.k8s.io/v1beta1
{{- else }}
apiVersion: extensions/v1beta1
{{- end }}
kind: Ingress
```

```yaml
# Conditional based on Kubernetes minor version:
{{- if semverCompare ">=1.27-0" .Capabilities.KubeVersion.Version }}
  # Use features available in Kubernetes 1.27+
{{- end }}
```

**Production Note:** Always use `.Capabilities.APIVersions.Has()` to check API availability rather than version comparison. API availability depends on both Kubernetes version and enabled feature gates, and `.Has()` reflects the actual cluster API surface.

---

## Working with Third-Party Charts

### Evaluation Criteria

| Criterion | What to Check | Why |
|-----------|---------------|-----|
| Maintainer reputation | Is the maintainer known? How many contributors? | Likelihood of ongoing support. |
| Release frequency | When was the last release? How frequent are updates? | Security patches and Kubernetes compatibility. |
| GitHub stars/issues | Star count, open vs closed issues ratio. | Community health. |
| Chart version vs app version | Does the chart track the latest stable app version? | Application currency. |
| Template quality | Run `helm lint --strict`. Review templates. | Quality of generated manifests. |
| Security posture | Are secrets passed via values? Are containers run as root by default? Does it mount host paths? | Production safety. |
| Dependency depth | How many sub-dependencies? Are they pinned? | Supply chain risk. |
| License | What license? Does it align with organizational policy? | Legal compliance. |

### Security Review Checklist

- [ ] No credentials in `values.yaml` (use Kubernetes Secrets or external secret management).
- [ ] Containers run as non-root (`securityContext.runAsNonRoot: true`).
- [ ] Read-only root filesystem where possible (`securityContext.readOnlyRootFilesystem: true`).
- [ ] No hostNetwork, hostPID, or hostIPC unless explicitly required.
- [ ] Resource limits are set on all containers.
- [ ] NetworkPolicies restrict traffic appropriately.
- [ ] `automountServiceAccountToken: false` unless required.
- [ ] No use of `latest` image tag.

### Customization Strategies

| Strategy | Description | Best For |
|----------|-------------|----------|
| Values overrides | Use `--set` or custom `values.yaml`. | Minor configuration changes. |
| Wrapper chart | Create a parent chart with the third-party chart as a dependency. | Adding resources or modifying behavior without forking. |
| Post-renderer | Apply `kustomize` or a custom script after rendering. | Patch labels, annotations, or inject sidecars. |
| Fork | Create and maintain a custom version of the chart. | Major customizations or when upstream is unmaintained. |

---

## Chart Development Workflow

### Local Development

```bash
# 1. Create or modify the chart
vim templates/deployment.yaml

# 2. Lint
helm lint --strict ./mychart

# 3. Render locally (no cluster needed)
helm template test-release ./mychart --debug

# 4. Test individual template
helm template test-release ./mychart --show-only templates/deployment.yaml

# 5. Unit test templates
helm unittest ./mychart

# 6. Validate with dry-run
helm install test-release ./mychart --dry-run --debug
```

### Minikube/Kind Testing

```bash
# Start a local cluster
minikube start --kubernetes-version=v1.29.0
# or
kind create cluster --image kindest/node:v1.29.0

# Install the chart
helm install test-release ./mychart --wait --timeout 5m

# Run integration tests
helm test test-release --logs

# Verify resources
kubectl get all -l app.kubernetes.io/instance=test-release
kubectl logs deployment/test-release-mychart

# Exercise upgrade path
helm upgrade test-release ./mychart --set replicaCount=5 --wait

# Test rollback
helm rollback test-release --wait

# Clean up
helm uninstall test-release
```

### CI Validation Pipeline

```bash
#!/bin/bash
# ci-validate.sh — Run in CI for every chart change

set -euo pipefail

CHART_DIR="./mychart"
RELEASE_NAME="ci-test-${BUILD_ID}"

echo "=== Step 1: Lint ==="
helm lint --strict --with-subcharts "${CHART_DIR}"

echo "=== Step 2: Unit tests ==="
helm unittest "${CHART_DIR}"

echo "=== Step 3: Dry-run install ==="
helm template "${RELEASE_NAME}" "${CHART_DIR}" --debug > /dev/null

echo "=== Step 4: Install on test cluster ==="
helm install "${RELEASE_NAME}" "${CHART_DIR}" \
  --namespace "ci-${BUILD_ID}" --create-namespace \
  --wait --timeout 5m

echo "=== Step 5: Run tests ==="
helm test "${RELEASE_NAME}" --namespace "ci-${BUILD_ID}" --logs

echo "=== Step 6: Test upgrade ==="
helm upgrade "${RELEASE_NAME}" "${CHART_DIR}" \
  --namespace "ci-${BUILD_ID}" \
  --wait --timeout 5m

echo "=== Step 7: Test rollback ==="
helm rollback "${RELEASE_NAME}" --namespace "ci-${BUILD_ID}" --wait

echo "=== Step 8: Cleanup ==="
helm uninstall "${RELEASE_NAME}" --namespace "ci-${BUILD_ID}" --wait
kubectl delete namespace "ci-${BUILD_ID}" --ignore-not-found

echo "=== All validations passed ==="
```

---

## Complete Chart Development Lifecycle Diagram

```
+---------------------------------------------------------------------------------------------+
|                      HELM CHART DEVELOPMENT LIFECYCLE                                       |
+---------------------------------------------------------------------------------------------+
|                                                                                              |
|  +-------------+     +-------------+     +-------------+     +-------------+                 |
|  | 1. PLAN     |     | 2. CREATE    |     | 3. DEVELOP   |     | 4. TEST      |              |
|  |             |     |             |     |             |     |             |                 |
|  | - Identify  |---->| - helm      |---->| - Write      |---->| - helm lint  |              |
|  |   resources |     |   create    |     |   templates  |     | - unittest   |              |
|  | - Design    |     | - Update    |     | - Define     |     | - template   |              |
|  |   values    |     |   Chart.yaml|     |   values     |     | - dry-run    |              |
|  | - Plan      |     | - Scaffold  |     | - Add        |     | - minikube   |              |
|  |   deps      |     |   templates |     |   helpers    |     |   deploy     |              |
|  +------+------+     +------+------+     +------+------+     +------+------+               |
|         |                   |                   |                   |                        |
|         |                   |                   |                   |                        |
|         +-------------------+-------------------+-------------------+                        |
|                                           |                                                 |
|                                           v                                                 |
|                              +-------------+                                             |
|                              | 5. DOCUMENT |                                             |
|                              |             |                                             |
|                              | - README.md |                                             |
|                              | - values    |                                             |
|                              |   comments  |                                             |
|                              | - NOTES.txt |                                             |
|                              | - schema    |                                             |
|                              +------+------+                                             |
|                                     |                                                    |
|                                     v                                                    |
|  +-------------+     +-------------+     +-------------+     +-------------+             |
|  | 6. PACKAGE  |     | 7. DISTRIBUTE|     | 8. MAINTAIN  |     | 9. DEPRECATE|            |
|  |             |     |             |     |             |     |             |             |
|  | - helm       |---->| - Push to   |---->| - Monitor   |---->| - Mark      |            |
|  |   package   |     |   OCI/repo  |     |   issues    |     |   deprecated|            |
|  | - Sign      |     | - Artifact  |     | - Update    |     | - Announce  |             |
|  |   package   |     |   Hub       |     |   deps      |     |   EOL       |              |
|  | - Version   |     | - Announce  |     | - Release   |     | - Archive   |             |
|  |   bump      |     |   release   |     |   patches   |     |   repo      |             |
|  +-------------+     +-------------+     +-------------+     +-------------+             |
|                                                                                            |
+---------------------------------------------------------------------------------------------+
```

---

## Best Practices Summary Checklist

### Chart Structure
- [ ] `Chart.yaml` has `apiVersion: v2`, `name`, `version` (required fields).
- [ ] `type` is `application` or `library` as appropriate.
- [ ] `appVersion` reflects the packaged application version.
- [ ] `kubeVersion` constrains compatible Kubernetes versions.
- [ ] One Kubernetes resource per template file.
- [ ] `_helpers.tpl` contains all shared named templates.
- [ ] `NOTES.txt` is informative and actionable.
- [ ] `tests/` directory contains at least one meaningful test.
- [ ] `.helmignore` excludes development artifacts.

### Values and Configuration
- [ ] `values.yaml` uses comments (`# --`) for every parameter.
- [ ] Sensible defaults are provided for every value.
- [ ] `values.schema.json` validates required values and types.
- [ ] No secrets in `values.yaml` — use Kubernetes Secrets or external secret management.
- [ ] Subchart values are namespaced under their chart name/alias.
- [ ] Global values are used sparingly and only for true cross-cutting concerns.

### Templates
- [ ] All resources have standard labels (`app.kubernetes.io/*`).
- [ ] All selectors use `matchLabels` from a helper, not inline values.
- [ ] `required` function is used for mandatory values.
- [ ] Resource quotas (limits/requests) are templated and configurable.
- [ ] `.Capabilities.APIVersions.Has()` is used for API compatibility.
- [ ] Security contexts are set on pods and containers.
- [ ] Image tags are not `latest` (use immutable tags).
- [ ] ServiceAccount creation is optional and configurable.

### Dependencies
- [ ] Dependency versions are pinned to specific ranges (not `*`).
- [ ] `Chart.lock` is committed to version control.
- [ ] Tags and conditions control optional dependencies.
- [ ] Subchart values are appropriately configured.
- [ ] Library chart dependencies are documented.

### Testing and Validation
- [ ] `helm lint --strict --with-subcharts` passes.
- [ ] `helm unittest` covers all templates.
- [ ] `helm template` produces valid YAML.
- [ ] `helm install --dry-run --debug` succeeds against a test cluster.
- [ ] `helm test` validates end-to-end functionality.
- [ ] Upgrade from the previous chart version is tested.
- [ ] Rollback is tested.

### Documentation
- [ ] `README.md` describes the chart, prerequisites, parameters, and upgrade notes.
- [ ] `values.yaml` has inline documentation comments.
- [ ] `NOTES.txt` provides post-install guidance.
- [ ] `CHANGELOG.md` or release notes track version changes.
- [ ] Breaking changes have migration guides.

### Distribution and Maintenance
- [ ] Chart package is signed (if using GPG).
- [ ] OCI or chart repository is publicly accessible (if open-source).
- [ ] Artifact Hub annotations are present (if publishing publicly).
- [ ] Deprecation notice is added before removing a chart.
- [ ] Dependency updates are tested before release.
- [ ] Security vulnerabilities in dependencies are monitored.