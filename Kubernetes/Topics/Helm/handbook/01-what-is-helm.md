# Chapter 1: What is Helm?

---

## 1.1 The Problem: Why Helm Exists

Imagine deploying a production application to Kubernetes. Your application needs:

- A **Deployment** with 3 replicas, resource limits, probes, and affinity rules
- A **Service** to expose it internally
- An **Ingress** with TLS termination
- A **ConfigMap** for environment-specific configuration
- A **Secret** for database credentials
- A **PersistentVolumeClaim** for storage
- A **HorizontalPodAutoscaler** for scaling
- A **PodDisruptionBudget** for availability
- A **ServiceAccount** with RBAC bindings
- A **NetworkPolicy** for network segmentation

That is **10 YAML files** for a single microservice. Across 50 microservices and 3 environments (dev, staging, prod), you are now managing **1,500 YAML manifests** — each with environment-specific values that differ by namespace, replica count, resource allocation, ingress hostname, and database connection strings.

The result without Helm:

- Copy-pasted YAML with manual find-and-replace
- No versioning of deployment configurations
- No easy way to share reusable application definitions
- Manual rollback by re-applying old manifests (if you remembered to save them)
- No consistent lifecycle management (install, upgrade, rollback, uninstall)
- Duplication across environments with drift over time

Helm was created to solve exactly these problems.

---

## 1.2 Problems Helm Solves

### 1.2.1 Templating

Instead of hardcoding values into YAML, Helm uses Go templates to parameterize manifests. A single Deployment template works across all environments.

**Example — Parameterized Deployment:**

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
```

With a values file per environment:

```yaml
# values-prod.yaml
replicaCount: 5
image:
  repository: myregistry.io/myapp
  tag: v2.1.3
resources:
  limits:
    cpu: "2000m"
    memory: "4Gi"
```

```yaml
# values-dev.yaml
replicaCount: 1
image:
  repository: myregistry.io/myapp
  tag: latest
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
```

**Production Note:** One template, infinite environments. No copy-paste, no drift.

### 1.2.2 Versioning

Helm charts carry semantic versions (`Chart.yaml` → `version`). Every deployment of a chart — a **release** — receives an incrementing revision number. This means:

- You know **exactly** which version of your application configuration is running in every environment
- You can **roll back** to any previous revision instantly
- You can **audit** the history of changes with `helm history`

```bash
$ helm history my-release
REVISION  UPDATED                   STATUS      CHART         APP VERSION  DESCRIPTION
1         Mon Jan 15 10:02:14 2026  superseded  myapp-1.1.0   2.1.0        Install complete
2         Mon Jan 15 10:05:22 2026  superseded  myapp-1.2.0   2.2.0        Upgrade complete
3         Mon Jan 15 14:30:01 2026  deployed    myapp-1.3.0   2.3.0        Upgrade complete
```

**Exam Tip:** The CKA exam frequently tests `helm history`, `helm rollback`, and understanding revision numbers. Know that revisions start at **1** and increment monotonically.

### 1.2.3 Sharing

Helm charts are designed to be shared. The ecosystem includes:

- **ArtifactHub** — the central Helm chart repository (https://artifacthub.io) with thousands of community charts
- **OCI Registries** — any OCI-compliant container registry can store Helm charts (Docker Hub, ECR, GCR, ACR, Harbor)
- **Git** — charts are source code; they belong in version control alongside application code
- **Private Repositories** — organizations run internal Helm repositories for proprietary charts

```bash
# Adding a repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Searching across all repositories
helm search repo wordpress

# Pulling a chart from OCI
helm pull oci://registry-1.docker.io/bitnamicharts/nginx
```

**Production Note:** Use OCI registries for production. Traditional Helm repositories (simple HTTP servers with an `index.yaml`) do not support content-addressable storage, have weak integrity guarantees, and are being phased out in the ecosystem.

### 1.2.4 Configuration Management

Helm separates **what** you deploy (the chart) from **how** it is configured (the values). This separation enables:

- **Environment-specific overrides** via multiple values files
- **Layered configuration** — default values → environment values → CLI overrides
- **Schema validation** — `values.schema.json` enforces type and constraint checking
- **Secret management** — integration with external secret stores (Vault, AWS Secrets Manager, SOPS)

```
Values Precedence (lowest to highest):
  chart/values.yaml           ← Defaults from the chart author
  → values-env.yaml            ← Environment-specific overrides
  → --values custom.yaml       ← Additional custom values files
  → --set key=value            ← CLI inline overrides
  → --set-string key=value     ← CLI string overrides (forces string type)
```

### 1.2.5 Lifecycle Management

Helm provides a complete lifecycle API for Kubernetes resources:

| Operation | Command | What Happens |
|-----------|---------|--------------|
| Install | `helm install` | Creates a new release — all resources are created in the cluster |
| Upgrade | `helm upgrade` | Applies changes to an existing release — creates, updates, or deletes resources as needed |
| Rollback | `helm rollback` | Reverts to a previous revision by re-applying its manifest |
| Uninstall | `helm uninstall` | Deletes all resources associated with a release |
| History | `helm history` | Lists all revisions of a release |
| Status | `helm status` | Shows the current state of a release |
| Test | `helm test` | Runs chart tests against a deployed release |
| Get | `helm get` | Retrieves release information (values, manifest, hooks, notes) |

---

## 1.3 Why Kubernetes Needs Helm

Kubernetes is a **declarative** system — you describe desired state, and controllers reconcile reality to match it. But Kubernetes does not prescribe:

- How to **author** and **organize** resource definitions
- How to **parameterize** resources for different environments
- How to **package** and **distribute** groups of resources
- How to **version** and **roll back** groups of resources atomically
- How to manage the **lifecycle** of complex, multi-resource applications

Helm fills this gap. It is to Kubernetes what `apt` is to Debian — the package manager that sits one layer above the core platform.

---

## 1.4 Package Manager Comparison

| Platform | Package Manager | Package Format | Repository | Install Command | Registry Type |
|----------|----------------|---------------|------------|----------------|---------------|
| Ubuntu/Debian | `apt` | `.deb` | apt repositories | `apt install nginx` | HTTP(S) repos |
| macOS | `brew` (Homebrew) | Formula (Ruby) | Taps (Git repos) | `brew install nginx` | Git repos |
| Node.js | `npm` | `.tgz` (tarball) | npm registry | `npm install express` | Central registry |
| Python | `pip` | `.whl` / `.tar.gz` | PyPI | `pip install flask` | Central registry |
| Java | Maven/Gradle | `.jar` / `.war` | Maven Central | `mvn install` | Central registry |
| **Kubernetes** | **Helm** | **`.tgz` (chart)** | **Helm repos / OCI** | **`helm install`** | **HTTP(S) / OCI** |

**Note:** The Helm `.tgz` package is simply a gzipped tarball of the chart directory. You can inspect it with `tar`:

```bash
$ tar -xzf mychart-1.0.0.tgz
$ ls mychart/
Chart.yaml  values.yaml  charts/  templates/  README.md  .helmignore
```

---

## 1.5 All Key Helm Concepts — Explained in Detail

### 1.5.1 Chart

A **chart** is a Helm package. It contains all the resource definitions necessary to run an application, tool, or service on Kubernetes.

**Directory structure:**

```
mychart/
├── Chart.yaml              # Chart metadata (name, version, description, dependencies)
├── values.yaml             # Default configuration values
├── values.schema.json      # (Optional) JSON Schema for values validation
├── charts/                 # Directory for chart dependencies (subcharts)
│   └── postgresql-12.1.0.tgz
├── templates/              # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl        # Named template definitions (partials)
│   ├── NOTES.txt           # Post-install help text displayed to user
│   └── tests/              # Test pod definitions
│       └── test-connection.yaml
├── crds/                   # Custom Resource Definitions (installed before templates)
│   └── my-crd.yaml
├── README.md               # Human-readable chart documentation
├── LICENSE                 # Chart license
└── .helmignore             # Files to exclude from the package (like .gitignore)
```

**Exam Tip:** Know that `Chart.yaml` is required. `templates/` is where your Kubernetes YAML templates live. `values.yaml` provides defaults. `charts/` holds dependencies. `crds/` contains CRD YAML files that are installed before any template rendering and are **never** updated or deleted by upgrade/rollback.

### 1.5.2 Chart.yaml — Required Fields

```yaml
apiVersion: v2              # Helm 3 API version (Helm 2 used v1)
name: myapp                 # Chart name — must match directory name
version: 1.2.3              # Chart version (SemVer) — this is the PACKAGE version
appVersion: "2.1.0"         # Application version — the version of the app INSIDE the chart
description: A Helm chart for deploying MyApp
type: application           # "application" or "library" (library charts provide templates only)
maintainers:
  - name: Platform Team
    email: platform@example.com
keywords:
  - web
  - api
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    tags:
      - database
annotations:
  category: WebApplication
  licenses: MIT
```

### 1.5.3 Chart Type: Application vs Library

| Type | Purpose | Contains Templates? | Can Be Installed? |
|------|---------|---------------------|-------------------|
| `application` | Deployable application or service | Yes | Yes |
| `library` | Reusable template fragments and helper functions | Yes (only partials) | No |

**Library charts** do not produce any Kubernetes resources directly. They are included as dependencies and provide named templates that parent charts can invoke.

```yaml
# Library chart's Chart.yaml
apiVersion: v2
name: mylib
version: 1.0.0
type: library
```

```yaml
# Parent chart using the library
dependencies:
  - name: mylib
    version: 1.0.0
    repository: "file://../mylib"
```

Then in the parent's templates:

```yaml
{{ include "mylib.labels" . }}
{{ include "mylib.selectorLabels" . }}
```

**Exam Tip:** Know the difference between `application` and `library` chart types. Library charts cannot be installed standalone (`helm install` will fail). They are used exclusively for sharing template code across multiple charts.

### 1.5.4 Repository

A **repository** is a collection of charts. There are two kinds:

**Traditional Helm Repository (HTTP):**

A simple HTTP server serving an `index.yaml` file and `.tgz` chart packages.

```
https://charts.example.com/
├── index.yaml              # Catalogue of all charts and versions
├── myapp-1.0.0.tgz
├── myapp-1.1.0.tgz
└── myapp-1.2.0.tgz
```

The `index.yaml` contains metadata for every chart version:

```yaml
apiVersion: v1
entries:
  myapp:
    - apiVersion: v2
      name: myapp
      version: 1.2.0
      urls:
        - https://charts.example.com/myapp-1.2.0.tgz
      created: "2026-07-11T10:00:00Z"
      digest: sha256:abc123...
    - apiVersion: v2
      name: myapp
      version: 1.1.0
      urls:
        - https://charts.example.com/myapp-1.1.0.tgz
```

**OCI Repository (Container Registry):**

Helm charts stored as OCI artifacts in any OCI-compliant container registry (Docker Hub, ECR, GCR, ACR, Harbor, GitHub Container Registry).

```bash
# OCI commands
helm pull oci://registry-1.docker.io/bitnamicharts/nginx --version 18.2.0
helm push myapp-1.0.0.tgz oci://registry-1.docker.io/myorg/
```

**Production Note:** Prefer OCI registries for production. They provide:
- Content-addressable storage (SHA256 digests)
- Signing and verification (cosign, notation)
- Fine-grained access control (same as container images)
- No need to maintain a separate HTTP server with `index.yaml`

**ArtifactHub** (https://artifacthub.io) is the CNCF's central index for Helm charts. It aggregates charts from hundreds of repositories but does not host charts directly. Think of it as "Google for Helm charts."

### 1.5.5 Release

A **release** is an instance of a chart running in a Kubernetes cluster. It is the combination of:

- A specific **chart** (name + version)
- A specific **configuration** (values)
- A specific **namespace**
- A **release name** (user-provided or auto-generated)

```
Release = Chart + Values + Namespace + Release Name
```

**Key properties:**

- **Immutable state:** Each release revision's manifest is stored and never modified. This enables deterministic rollback.
- **Mutable state:** The "current" revision of a release changes on every `helm install` and `helm upgrade`.
- **Namespaced:** By default, releases are scoped to a namespace. A chart installed in `prod` and `dev` creates two independent releases (unless `--create-namespace` is used, which is the recommended pattern).

```bash
# These are two independent releases
helm install myapp-prod ./mychart --namespace prod
helm install myapp-dev  ./mychart --namespace dev
```

**Production Note:** Use meaningful release names that encode environment and application. Avoid names like `test` or `foo`. Use `myapp-prod`, `myapp-staging`, `myapp-dev`.

**Warning:** Do **not** reuse a release name that has been uninstalled unless you include the `--history-max` flag. Helm keeps release history by default (up to 10 revisions). Reusing a name before the history is purged can cause conflicts. Always set `--history-max` explicitly.

### 1.5.6 Revision

A **revision** is a specific versioned snapshot of a release. Revisions are:

- **Monotonically incrementing integers** starting at 1
- **Immutable** — once created, a revision's manifest never changes
- **Stored** as Kubernetes Secrets (by default) in the release namespace

Every `helm install`, `helm upgrade`, and `helm rollback` creates a new revision.

| Action | Revision Created? | New Revision Number |
|--------|-------------------|---------------------|
| `helm install` | Yes | 1 |
| `helm upgrade` | Yes | Previous + 1 |
| `helm rollback` | Yes | Previous + 1 |
| `helm uninstall` | No (deletes all) | N/A |

```bash
$ helm history my-release
REVISION  UPDATED                   STATUS          CHART         APP VERSION  DESCRIPTION
1         Thu Jul 10 09:00:00 2026  superseded      myapp-1.0.0   1.0          Install complete
2         Thu Jul 10 10:00:00 2026  superseded      myapp-1.1.0   1.1          Upgrade complete
3         Thu Jul 10 11:00:00 2026  superseded      myapp-1.2.0   1.2          Upgrade complete
4         Thu Jul 10 14:30:00 2026  deployed        myapp-1.2.0   1.2          Rollback to 2
```

**Exam Tip:** Revision 4 in the example above is a rollback to revision 2's manifest, but it is recorded as a **new** revision. Revision numbers only go up — they are never reused or decreased. A rollback to revision 2 creates revision 5, which has the same manifest as revision 2 but is a distinct revision.

### 1.5.7 Package

A **package** (or chart archive) is a `.tgz` file containing the chart directory. It is the unit of distribution in Helm.

```bash
# Create a package
$ helm package ./mychart
Successfully packaged chart and saved it to: ./mychart-1.0.0.tgz

# Verify its contents
$ tar -tzf mychart-1.0.0.tgz
mychart/Chart.yaml
mychart/values.yaml
mychart/templates/deployment.yaml
mychart/templates/service.yaml
mychart/templates/_helpers.tpl
mychart/templates/NOTES.txt
mychart/.helmignore
mychart/charts/
```

**The package includes:**

- All templates and static files
- Chart metadata (`Chart.yaml`)
- Default values (`values.yaml`)
- Dependencies (`.tgz` files in `charts/`) — if packaged with `--dependency-update` or if dependencies were already fetched
- The `.helmignore` file determines which local files are excluded from the package

```bash
# Example .helmignore
.git
.gitignore
*.swp
*.bak
*.orig
.vscode/
.idea/
*.md         # Exclude README from package (NOT recommended)
```

**Note:** Helm does NOT include `.git/` directories or files matching `.helmignore` patterns in the package.

### 1.5.8 OCI Registry

OCI (Open Container Initiative) registries store Helm charts as OCI artifacts. This is the modern, recommended approach for Helm chart storage.

**How it works:**

An OCI artifact is a generic blob + manifest stored alongside container images. Helm charts are pushed as OCI artifacts with a specific media type.

```
registry-1.docker.io/myorg/mychart
├── v1.0.0 (OCI artifact)
│   ├── manifest (JSON, mediaType: application/vnd.cncf.helm.config.v1+json)
│   ├── layer: mychart-1.0.0.tgz (gzipped chart)
│   └── layer: provenance.json (optional, for signed charts)
├── v1.1.0 (OCI artifact)
└── v1.2.0 (OCI artifact)
```

```bash
# OCI chart operations
helm pull oci://registry-1.docker.io/myorg/mychart --version 1.0.0
helm push mychart-1.0.0.tgz oci://registry-1.docker.io/myorg/
helm show all oci://registry-1.docker.io/myorg/mychart --version 1.0.0
helm install my-release oci://registry-1.docker.io/myorg/mychart --version 1.0.0
```

**Production Note:** OCI registries inherit the same authentication mechanism as container images. Use `helm registry login` to authenticate:

```bash
# Docker Hub
echo "$DOCKER_PASSWORD" | helm registry login registry-1.docker.io \
  --username "$DOCKER_USERNAME" --password-stdin

# AWS ECR
aws ecr get-login-password --region us-east-1 | \
  helm registry login 123456789012.dkr.ecr.us-east-1.amazonaws.com \
  --username AWS --password-stdin

# Azure ACR
helm registry login myregistry.azurecr.io \
  --username "$ACR_USERNAME" --password "$ACR_PASSWORD"

# Google Artifact Registry
gcloud auth print-access-token | \
  helm registry login us-central1-docker.pkg.dev \
  --username oauth2accesstoken --password-stdin
```

### 1.5.9 Templates

Templates are Go `text/template` files stored in the `templates/` directory of a chart. They produce Kubernetes YAML manifests when rendered with values.

**Template lifecycle:**

```
values.yaml ──────────────┐
                          ├──→ Go Template Engine ──→ Rendered YAML ──→ Kubernetes API
values override ──────────┤     (text/template + Sprig)
built-in objects (.Release,│
  .Chart, .Capabilities) ─┘
```

**Key template characteristics:**

- Files in `templates/` with any extension are processed (convention: `.yaml`, `.tpl`)
- Files prefixed with `_` are partials — not rendered standalone, but available via `{{ define }}` / `{{ include }}` / `{{ template }}`
- The `templates/NOTES.txt` file is special — its rendered content is displayed after `helm install` and `helm upgrade`
- Templates can contain any valid Kubernetes resource YAML, plus Go template directives

**Example template:**

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
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
```

**The `{{-` and `-}}` whitespace controls:**

```
{{-  → Strip whitespace to the left (before this directive)
-}}  → Strip whitespace to the right (after this directive)
```

```yaml
# Without whitespace control
label:
  {{ .Values.label }}

# Renders as:
label:
  production

# With whitespace control
label:
  {{- .Values.label }}

# Renders as:
label:production    # Wrong — missing space

# With proper control
label: {{ .Values.label }}
# or
label: {{- " " }}{{ .Values.label }}
```

**Exam Tip:** Whitespace control (`{{-` and `-}}`) is tested in the CKA exam. Understand when to chomp whitespace for clean output and when NOT to (e.g., inside quoted strings or inline with other text where a space is needed).

### 1.5.10 Values

Values are the configuration data passed into templates. They come from multiple sources with a well-defined precedence order.

**Values sources (lowest to highest precedence):**

```
1. chart/values.yaml                  ← Defaults (chart author)
2. Parent chart's values (if subchart) ← Overridden via global or specific keys
3. --values / -f file.yaml            ← User-provided values files
4. --set key=value                    ← CLI inline overrides
5. --set-string key=value             ← CLI string overrides
```

**Example — values.yaml:**

```yaml
# Default values for mychart
replicaCount: 1

image:
  repository: nginx
  tag: ""
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific

resources: {}
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

nodeSelector: {}
tolerations: []
affinity: {}
```

**Accessing values in templates:**

```yaml
{{ .Values.replicaCount }}
{{ .Values.image.repository }}
{{ .Values.image.tag | default .Chart.AppVersion }}
{{ .Values.service.port }}
```

**Values schema validation (values.schema.json):**

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1
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

```bash
# Helm validates values against the schema on install/upgrade
$ helm install my-release ./mychart --set service.port=99999
Error: values don't meet the specifications of the schema(s) in the following chart(s):
mychart:
- service.port: Must be less than or equal to 65535
```

### 1.5.11 Dependencies (Subcharts)

Dependencies allow a chart to include other charts. They are declared in `Chart.yaml` and stored (as `.tgz` files) in the `charts/` directory.

**Declaring dependencies in Chart.yaml:**

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    tags:
      - database

  - name: redis
    version: "18.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
    tags:
      - cache

  - name: common
    version: "2.0.0"
    repository: "file://../common"
```

**Commands:**

```bash
# Download dependencies into charts/
helm dependency update ./mychart

# List dependencies
helm dependency list ./mychart

# Build a dependencies lock file (Chart.lock)
helm dependency build ./mychart
```

**The `charts/` directory after `helm dependency update`:**

```
mychart/
└── charts/
    ├── postgresql-12.1.0.tgz
    ├── redis-18.0.0.tgz
    └── common-2.0.0.tgz
```

**Conditional dependencies:**

Dependencies can be conditionally enabled/disabled using the `condition` and `tags` fields:

```yaml
# values.yaml
postgresql:
  enabled: true    # condition: postgresql.enabled evaluates to true → postgresql is installed

redis:
  enabled: false   # condition: redis.enabled evaluates to false → redis is skipped

tags:
  database: true   # postgresql has tags: [database] → enabled
  cache: false     # redis has tags: [cache] → disabled
```

**Note:** For a dependency to be enabled, **all** conditions and **all** tags must evaluate to true (AND logic).

**Values precedence with subcharts:**

```
Subchart values.yaml             (lowest)
→ Parent's values for subchart key
→ Parent's global values           (highest)
```

```yaml
# Parent's values.yaml
postgresql:                    # Passed to the postgresql subchart
  auth:
    username: myuser
    password: mypassword
  primary:
    persistence:
      size: 20Gi

global:                        # Inherited by ALL subcharts
  imageRegistry: myregistry.io
  imagePullSecrets:
    - name: my-secret
```

**Production Note:** Be careful with subchart values. Subchart defaults are merged with parent overrides, but list-type values are **replaced** (not merged). Always consult the subchart's documentation for available values.

### 1.5.12 Hooks

Hooks are Kubernetes resources annotated to run at specific points in the release lifecycle. They are defined as normal templates but with special annotations.

**Available hook events:**

| Hook | When It Runs | Use Case |
|------|-------------|----------|
| `pre-install` | After templates are rendered, before resources are created | Schema migrations, pre-flight checks |
| `post-install` | After all resources are created | Data seeding, notifications |
| `pre-delete` | Before resource deletion | Backup before cleanup |
| `post-delete` | After all resources are deleted | Cleanup, notifications |
| `pre-upgrade` | After templates rendered, before upgrade applied | Schema migrations |
| `post-upgrade` | After upgrade applied | Cache warming, notifications |
| `pre-rollback` | Before rollback | Pre-rollback validation |
| `post-rollback` | After rollback | Cleanup, notifications |
| `test` | When `helm test` is run | Integration tests |

**Hook annotations:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migration
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install       # Run before upgrade AND before install
    "helm.sh/hook-weight": "5"                     # Execution order (lower = first)
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: myapp-migrations:{{ .Values.image.tag }}
          command: ["./run-migrations.sh"]
```

**Hook weights:**

- Default weight is `0`
- Hooks with lower weights run first
- Hooks of the same type with the same weight run in parallel
- Negative weights run before positive weights

**Hook delete policies:**

| Policy | Behavior |
|--------|----------|
| `before-hook-creation` | Delete the previous hook resource before creating a new one (default for Jobs) |
| `hook-succeeded` | Delete the hook resource after it succeeds |
| `hook-failed` | Delete the hook resource if it fails |

**Exam Tip:** Hooks create Kubernetes resources that Helm tracks separately from main resources. Hook resources are NOT deleted by `helm uninstall` unless you set a delete policy. They have their own lifecycle independent of the release.

### 1.5.13 Chart Version vs Application Version

| Field | Location | Meaning | Example |
|-------|----------|---------|---------|
| `version` | `Chart.yaml` | SemVer of the **chart package** itself | `1.2.3` |
| `appVersion` | `Chart.yaml` | Version of the **application** deployed by the chart | `2.1.0` |

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.5.0        # Chart version — bumped when chart template/config changes
appVersion: "3.2.1"   # App version — bumped when the container image version changes
```

**Why two versions?**

- You can update the chart (fix a template bug, add a probe, change resource limits) without changing the application version.
- You can update the application (new image tag) without modifying the chart.

In templates, you can reference both:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

This means if `.Values.image.tag` is not set, Helm falls back to `.Chart.AppVersion`.

**Exam Tip:** In `helm history`, both `CHART` and `APP VERSION` columns are shown. Know the difference. The CKA may ask you to inspect a release and identify the chart version vs the application version.

### 1.5.14 Manifest

A **manifest** is the complete set of rendered Kubernetes YAML documents produced by processing a chart's templates with a set of values. It is a multi-document YAML file (separated by `---`).

```bash
# View the rendered manifest without installing
$ helm template my-release ./mychart --values values-prod.yaml

# View the manifest of an installed release
$ helm get manifest my-release
```

The manifest contains **all** resources the chart produces:

```yaml
---
# Source: mychart/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-release-mychart
  labels:
    helm.sh/chart: mychart-1.0.0
    app.kubernetes.io/name: mychart
    app.kubernetes.io/instance: my-release
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/managed-by: Helm
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-release-mychart-config
data:
  environment: "production"
  log_level: "info"
---
# Source: mychart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-release-mychart
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/name: mychart
    app.kubernetes.io/instance: my-release
---
# Source: mychart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-release-mychart
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: mychart
      app.kubernetes.io/instance: my-release
  template:
    spec:
      containers:
        - name: mychart
          image: myregistry.io/myapp:2.1.0
```

### 1.5.15 Release Metadata

Helm stores release metadata as Kubernetes Secrets (default) or ConfigMaps (configurable) in the release namespace.

**Default Secret naming pattern:**

```
sh.helm.release.v1.<release-name>.v<revision>
```

Example:

```
sh.helm.release.v1.my-release.v1
sh.helm.release.v1.my-release.v2
sh.helm.release.v1.my-release.v3
```

**What's inside a release Secret:**

```bash
$ kubectl get secret sh.helm.release.v1.my-release.v1 -o yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sh.helm.release.v1.my-release.v1
  namespace: prod
  labels:
    owner: helm
    status: superseded                # deployed, superseded, pending-install, etc.
    name: my-release
    version: "1"
    modifiedAt: "1705284134"
    chart: mychart-1.0.0
    app.kubernetes.io/managed-by: Helm
type: helm.sh/release.v1
data:
  release: <base64-encoded-gzipped-JSON>
```

The `release` field in the Secret data contains a **base64-encoded, gzip-compressed JSON** payload with the full release information:

```json
{
  "name": "my-release",
  "info": {
    "status": "deployed",
    "first_deployed": "2026-07-11T10:00:00Z",
    "last_deployed": "2026-07-11T10:00:00Z",
    "description": "Install complete"
  },
  "chart": {
    "metadata": {
      "name": "mychart",
      "version": "1.0.0",
      "appVersion": "2.1.0"
    },
    "values": { /* All resolved values */ },
    "templates": [ /* All template files */ ]
  },
  "config": { /* Merged values used for this release */ },
  "manifest": "---\n# All rendered YAML\n---\n...",
  "version": 1,
  "namespace": "prod"
}
```

**Decoding a release Secret:**

```bash
# Extract and decode
kubectl get secret sh.helm.release.v1.my-release.v1 \
  -o jsonpath='{.data.release}' | \
  base64 -d | gunzip | jq .
```

**Exam Tip:** The CKA exam may test your ability to inspect Helm release secrets directly. Know the naming convention (`sh.helm.release.v1.<name>.v<revision>`) and understand that the data is base64-encoded gzipped JSON.

### 1.5.16 Storage Backend

Helm stores all release information in Kubernetes itself. This is called the **storage backend**.

| Backend | Helm Version | Default? | Pros | Cons |
|---------|-------------|----------|------|------|
| Secrets | Helm 3 | **Yes** | Encrypted at rest (if KMS is configured), RBAC controllable | Size limit (1MB per Secret) |
| ConfigMaps | Helm 3 | No (opt-in) | No size limit for values | Not encrypted at rest, visible in plain text |
| SQL (ConfigMaps) | Helm 2 only | N/A | Legacy | Removed in Helm 3 |

**Configuring the storage backend:**

```bash
# Use ConfigMaps instead of Secrets (NOT recommended for production)
helm install my-release ./mychart --set helm.sh/storage=configmaps

# Or set via environment variable
export HELM_DRIVER=configmap
helm install my-release ./mychart
```

**History limits:**

By default, Helm keeps a maximum of **10** revisions per release. This is configurable:

```bash
# Set history limit at install time
helm install my-release ./mychart --history-max 20

# Clean up old revisions
helm history my-release --max 5

# Delete a specific revision secret
kubectl delete secret sh.helm.release.v1.my-release.v5 -n prod
```

**Production Note:** Monitor the number of release Secrets in your cluster. In CI/CD environments with frequent deployments, 10 revisions can accumulate quickly. Set `--history-max` to a value appropriate for your rollback window. Consider a cleanup CronJob:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: helm-history-cleanup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: alpine/k8s:1.30.0
              command:
                - sh
                - -c
                - |
                  kubectl get secrets -n prod -l owner=helm -l status=superseded \
                    --sort-by=.metadata.labels.version \
                    -o name | head -n -10 | xargs -r kubectl delete -n prod
          restartPolicy: OnFailure
```

**Warning:** Deleting release Secrets manually removes Helm's ability to roll back past that revision. Always use `helm uninstall` or `helm history` with proper cleanup. Do not manually delete the currently `deployed` revision's Secret — Helm will lose track of the release.

### 1.5.17 Rendering Engine

The rendering engine is the internal Helm component that combines chart templates with values to produce Kubernetes YAML manifests.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HELM RENDERING ENGINE                        │
│                                                                     │
│  ┌──────────┐    ┌───────────────┐    ┌─────────────────────────┐   │
│  │ Values   │    │  Templates    │    │  Built-in Objects        │   │
│  │          │    │               │    │  .Release .Chart         │   │
│  │ values.  │    │ templates/    │    │  .Values .Capabilities   │   │
│  │ yaml     │    │ *.yaml *.tpl  │    │  .Template .Files        │   │
│  └────┬─────┘    └──────┬────────┘    └───────────┬─────────────┘   │
│       │                 │                         │                 │
│       └─────────┬───────┴─────────────┬───────────┘                 │
│                 │                     │                             │
│                 ▼                     ▼                             │
│          ┌──────────────────────────────────┐                       │
│          │   Go text/template + Sprig       │                       │
│          │   Template Execution Engine      │                       │
│          └──────────────┬───────────────────┘                       │
│                         │                                           │
│                         ▼                                           │
│          ┌──────────────────────────────────┐                       │
│          │   Multi-document YAML Output     │                       │
│          │   (manifest)                     │                       │
│          └──────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
```

**Rendering steps (internal):**

1. **Load Chart:** Parse `Chart.yaml`, load `values.yaml`, read all files in `templates/`
2. **Merge Values:** Apply values precedence — defaults → overrides → CLI flags
3. **Process Dependencies:** Recursively process subcharts (their templates are separate, scoped to their values)
4. **Execute Templates:** For each template file (excluding `_partials` and `NOTES.txt`), execute the Go template with merged values and built-in objects
5. **Validate:** Check that output is valid YAML and valid Kubernetes resource kinds
6. **Concatenate:** Join all rendered templates into a single multi-document YAML string
7. **Post-process:** Add release labels, add `helm.sh/chart` annotation, apply any transforms
8. **Output:** Return the final manifest

### 1.5.18 Template Engine (Go text/template + Sprig)

Helm's template engine is a combination of:

- **Go's `text/template`** package — provides the core template syntax (`{{ }}`, pipelines, conditionals, loops, etc.)
- **Sprig library** — provides 70+ template functions (string manipulation, math, date, encoding, etc.)
- **Helm-specific built-in objects** — `.Release`, `.Chart`, `.Values`, `.Capabilities`, `.Template`, `.Files`

**Go text/template basics:**

```
{{ .Values.replicaCount }}                    ← Print a value
{{ if .Values.ingress.enabled }}...{{ end }}   ← Conditional
{{ range .Values.env }}...{{ end }}            ← Loop
{{ with .Values.resources }}...{{ end }}       ← Scoped block
{{ include "mychart.labels" . }}               ← Named template inclusion
{{ define "mychart.labels" }}...{{ end }}      ← Named template definition
{{ $var := .Values.foo }}                      ← Variable assignment
{{ .Values.color | upper }}                    ← Pipeline — pipe left to right
{{ .Values.foo | default "bar" }}              ← Default value
{{ .Values.x | required "x is required" }}     ← Required value
```

**Sprig functions (commonly used):**

| Category | Functions |
|----------|-----------|
| String | `upper`, `lower`, `trim`, `trimAll`, `trimPrefix`, `trimSuffix`, `replace`, `repeat`, `quote`, `squote`, `nospace`, `hasPrefix`, `hasSuffix`, `contains`, `indent`, `nindent`, `camelcase`, `snakecase`, `kebabcase`, `abbrev`, `abbrevboth`, `trunc`, `shuffle`, `wrap`, `wrapWith`, `substr`, `initial`, `plural`, `toString` |
| Math | `add`, `sub`, `mul`, `div`, `max`, `min`, `maxf`, `minf`, `ceil`, `floor`, `round`, `add1`, `sub1` |
| Type | `toString`, `toJson`, `toPrettyJson`, `toRawJson`, `fromJson`, `toYaml`, `fromYaml`, `typeOf`, `kindOf`, `kindIs`, `typeIs`, `empty`, `ternary` |
| Date | `now`, `date`, `dateInZone`, `duration`, `dateModify`, `htmlDate`, `htmlDateInZone` |
| List | `list`, `first`, `last`, `rest`, `initial`, `append`, `prepend`, `concat`, `reverse`, `uniq`, `without`, `has`, `slice`, `join`, `sortAlpha`, `pick`, `omit`, `compact`, `dict`, `keys`, `values`, `merge`, `mergeOverwrite` |
| Encoding | `b64enc`, `b64dec`, `sha256sum`, `sha1sum`, `adler32sum`, `camelcase`, `snakecase`, `kebabcase`, `regexMatch`, `regexFindAll`, `regexReplaceAll` |
| Flow | `fail`, `required`, `coalesce`, `ternary`, `default`, `empty` |
| Kubernetes | `lookup` (query live K8s API from template), `include` (named template), `tpl` (re-render string as template) |

---

## 1.6 The Helm Ecosystem — ASCII Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           THE HELM ECOSYSTEM                                 │
│                                                                              │
│  ┌─────────────────┐                              ┌──────────────────────┐   │
│  │  ArtifactHub     │                              │  OCI Registries       │   │
│  │  (Chart Index)   │                              │  (Docker Hub, ECR,    │   │
│  │                  │                              │   GCR, ACR, Harbor)   │   │
│  └────────┬────────┘                              └───────────┬──────────┘   │
│           │                                                   │              │
│           │  indexes / discovers                              │  pushes /    │
│           │                                                   │  pulls       │
│           ▼                                                   ▼              │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                         HELM CLI (helm)                              │     │
│  │                                                                      │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐     │     │
│  │  │ create   │  │ install  │  │ upgrade  │  │ rollback         │     │     │
│  │  │ package  │  │ uninstall│  │ template │  │ history / status │     │     │
│  │  │ repo     │  │ plugin   │  │ test     │  │ get / lint       │     │     │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘     │     │
│  └────────┬──────────────────┬──────────────────┬──────────────────────┘     │
│           │                  │                  │                             │
│           ▼                  ▼                  ▼                             │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────────┐            │
│  │  Local       │  │  Kubernetes      │  │  External            │            │
│  │  Filesystem  │  │  API Server      │  │  Secret Stores       │            │
│  │              │  │                  │  │  (Vault, SOPS, etc.) │            │
│  │  charts/     │  │  ┌────────────┐  │  │                      │            │
│  │  values.yaml │  │  │ Secrets    │  │  │  helm-secrets plugin │            │
│  │  templates/  │  │  │ (release)  │  │  │  vault-plugin        │            │
│  │  .tgz files  │  │  └────────────┘  │  └──────────────────────┘            │
│  └──────────────┘  │  ┌────────────┐  │                                      │
│                    │  │ Resources  │  │                                      │
│                    │  │ (Deployment│  │                                      │
│                    │  │  Service,  │  │                                      │
│                    │  │  Ingress,  │  │                                      │
│                    │  │  etc.)     │  │                                      │
│                    │  └────────────┘  │                                      │
│                    └──────────────────┘                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1.7 How Helm Differs from Kustomize

Both Helm and Kustomize solve configuration management for Kubernetes, but they take fundamentally different approaches.

| Aspect | Helm | Kustomize |
|--------|------|-----------|
| **Approach** | Templating engine (Go templates + values) | Overlay/patch system (base + patches) |
| **How it works** | Templates produce YAML by substituting values at render time | Base YAML is selectively modified via patches, overlays, and transformers |
| **Package format** | `.tgz` chart archive | Plain directories of YAML |
| **Distribution** | Chart repositories + OCI registries | Git repositories (no packaging) |
| **Versioning** | Built-in (chart version + app version + revisions) | None (relies on Git) |
| **Lifecycle** | Full: install, upgrade, rollback, uninstall, history, status, test | None — `kubectl apply -k` only, no state tracking |
| **State management** | Tracks releases in Secrets, knows what it installed | Stateless — applies YAML, no record of what was applied |
| **Rollback** | `helm rollback` (re-apply previous manifest from stored history) | `kubectl rollout undo` (only for Deployments, not for all resources) |
| **Dependencies** | Subcharts, `dependencies` in Chart.yaml | `resources:` field or external reference |
| **Learning curve** | Moderate (Go templates have a learning curve) | Lower (mostly YAML) |
| **Best for** | Complex applications with many resources and environments | Simpler configs, environment-specific patches, GitOps workflows |
| **Kubernetes integration** | External tool | Built into `kubectl` as `kubectl kustomize` |

**Exam Tip:** Understand the Helm vs Kustomize trade-offs. The CKA exam does not test Kustomize directly, but you should know that Helm is a **package manager with lifecycle management** while Kustomize is a **template-free configuration customization tool**.

**Can they work together?** Yes — many teams use Helm to render templates and Kustomize for final environment-specific patching (post-render). Argo CD natively supports both Helm and Kustomize in the same Application.

---

## 1.8 Helm 2 vs Helm 3

Helm 3 was a major redesign released in November 2019. Understanding the differences is useful context.

### 1.8.1 Tiller Removal — The Biggest Change

```
Helm 2 Architecture:

  User ──→ helm CLI ──→ Tiller (in-cluster pod) ──→ Kubernetes API ──→ Resources
              │                    │
              │                    └── Stored releases in ConfigMaps
              │                        (in kube-system namespace)
              └── Communicated with Tiller via gRPC

Helm 3 Architecture:

  User ──→ helm CLI ──→ Kubernetes API ──→ Resources
              │
              └── Release history stored as Secrets in the release namespace
                  (uses same kubeconfig, same permissions as the user)
```

**Why Tiller was removed:**

| Problem with Tiller | Helm 3 Solution |
|---------------------|-----------------|
| Tiller ran as a cluster-wide superuser (cluster-admin by default) — massive security risk | Helm CLI uses the user's kubeconfig and permissions directly |
| Tiller was a single point of failure — one Tiller outage blocked all deployments | No server-side component to fail |
| Tiller's release storage was in `kube-system` — all tenants shared the same namespace | Releases stored in the release's own namespace |
| RBAC for Tiller was complex and poorly understood | Helm permissions are just the user's Kubernetes RBAC |
| Tiller needed its own TLS certificates for secure gRPC | No gRPC — direct HTTPS to the API server |

### 1.8.2 Other Key Differences

| Feature | Helm 2 | Helm 3 |
|---------|--------|--------|
| Server-side component | Tiller (required) | None (client-only) |
| Release storage | ConfigMaps in `kube-system` | Secrets in release namespace (default) |
| Release naming | Required user-provided name | Optional — auto-generates if not provided |
| Chart dependencies | `requirements.yaml` (separate file) | `dependencies:` in `Chart.yaml` |
| Chart API version | `apiVersion: v1` | `apiVersion: v2` |
| 3-way strategic merge | No (2-way) | Yes (3-way) |
| CRDs | `crds/` directory — managed like templates (updated on upgrade) | `crds/` directory — installed once, never updated or deleted |
| Lua hooks | Supported | Removed (use Kubernetes Jobs) |
| `helm serve` (local repo) | Built-in | Removed (use `helm repo index` + any HTTP server or `chartmuseum`) |
| Namespace management | Tiller handled namespaces | User's kubeconfig context determines namespace (override with `--namespace`) |
| Client-only mode | `helm template` | Same + `helm install --dry-run` |

### 1.8.3 3-Way Strategic Merge Patch (Helm 3)

Helm 3 introduced a 3-way merge for upgrades, resolving a long-standing pain point:

| Merge Type | Compares | Used By | Problem |
|------------|----------|---------|---------|
| 2-way merge | Old manifest vs New manifest | Helm 2 | Cannot detect resources created/edited outside Helm (drift) |
| **3-way merge** | **Old manifest vs New manifest vs Live state** | **Helm 3** | **Detects manual changes and can preserve or overwrite them** |

```
3-Way Merge:

  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
  │  Old Manifest │       │  New Manifest │       │  Live State   │
  │  (prev rev)   │       │  (desired)    │       │  (in cluster) │
  └──────┬───────┘       └──────┬───────┘       └──────┬───────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                │
                                ▼
                   ┌────────────────────────┐
                   │  Compute Strategic      │
                   │  Merge Patch (3-way)    │
                   │                        │
                   │  Patch = (Old→New)      │
                   │  + (Old→Live diff)      │
                   │  resolved by rules      │
                   └───────────┬────────────┘
                               │
                               ▼
                   ┌────────────────────────┐
                   │  Apply to API Server    │
                   └────────────────────────┘
```

**What this means in practice:**

- Fields **removed** from the new manifest → removed from the live resource
- Fields **added** to the new manifest → added to the live resource
- Fields **changed** in the new manifest → updated in the live resource
- Fields that are **same** in old and new manifest but **changed** in live state → the live change is **kept** (manual edits are preserved if Helm doesn't claim to manage them)

**Example scenario:**

```
Old Manifest:  replicas: 3, resources: {cpu: 500m}
New Manifest:  replicas: 5, resources: {cpu: 500m}
Live State:    replicas: 3, resources: {cpu: 1000m}  (manually scaled CPU)

Result:        replicas: 5, resources: {cpu: 1000m}
               ^ Helm updates replicas because it changed in new manifest
               ^ Helm preserves the manual CPU change because it didn't change
                 between old and new manifest — Helm assumes manual intent
```

---

## 1.9 Chapter Summary

| Concept | One-Line Definition |
|---------|---------------------|
| Chart | A package of Kubernetes resource templates, values, and metadata |
| Repository | A collection of charts, served via HTTP or OCI |
| Release | An instance of a chart running in a cluster — Chart + Values + Name |
| Revision | A numbered snapshot of a release (1, 2, 3...) — immutable |
| Package | A `.tgz` file containing a chart — the distribution unit |
| OCI Registry | Container-registry-style storage for Helm charts |
| Templates | Go template files that produce Kubernetes YAML when rendered |
| Values | Configuration data fed into templates — hierarchical, overridable |
| Dependencies | Subcharts included by a parent chart |
| Hooks | Kubernetes resources that run at specific lifecycle events |
| Manifest | The complete rendered YAML output from a chart |
| Release Metadata | Release info stored in base64-encoded gzipped JSON inside Secrets |
| Storage Backend | Kubernetes Secrets (or ConfigMaps) storing release history |
| Rendering Engine | Internal Helm component: templates + values → YAML |
| Template Engine | Go `text/template` + Sprig function library |

---

## 1.10 What's Next

Chapter 2 covers Helm's internal architecture: how the CLI talks to Kubernetes, how releases are stored, how the 3-way merge works, and what happens internally during every Helm operation.

→ Continue to **Chapter 2: Helm Architecture** (`02-architecture.md`)
