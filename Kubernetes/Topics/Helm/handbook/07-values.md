# Chapter 7: Values

## 7.1 What Are Values?

Values are the **configuration layer** of Helm. They are the bridge between a generic, reusable chart (templates) and a specific, concrete deployment. Every Helm template references `.Values` to inject configuration:

```yaml
# templates/deployment.yaml
replicas: {{ .Values.replicaCount }}
```

Without values, a chart is just a blueprint. With values, it becomes a running application tailored to a specific environment, team, or use case.

Values flow through the system as follows:

```
values.yaml (chart default)
    ↓
values.yaml (parent chart, if subchart)
    ↓
-f values/production.yaml (user-supplied file)
    ↓
--set key=value (CLI overrides)
    ↓
Merged into .Values → Templates render → Kubernetes manifests
```

---

## 7.2 `values.yaml`

### 7.2.1 Location and Purpose

Every Helm chart must contain a `values.yaml` in its root directory. This file defines **default values** for the chart—values that make the chart run out of the box with sensible defaults.

### 7.2.2 Structure and Naming Conventions

```yaml
# Chart-level metadata
nameOverride: ""
fullnameOverride: ""

# Image configuration
image:
  repository: nginx
  tag: "1.25.0"
  pullPolicy: IfNotPresent

# Resource requests and limits
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# Scaling
replicaCount: 1
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# Networking
service:
  type: ClusterIP
  port: 80

# Ingress
ingress:
  enabled: false
  className: ""
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific

# Advanced: node affinity, tolerations, pod security
nodeSelector: {}
tolerations: []
affinity: {}
podAnnotations: {}
podSecurityContext: {}
securityContext: {}

# Service account
serviceAccount:
  create: true
  annotations: {}
  name: ""
```

### 7.2.3 Naming Conventions

| Convention | Example | Rationale |
|---|---|---|
| camelCase | `replicaCount`, `pullPolicy` | Matches Helm's built-in template patterns |
| Nested objects | `image.repository`, `service.port` | Logical grouping, avoids flat namespace |
| Boolean flags | `enabled: false`, `tls.create: true` | Clear on/off semantics |
| Empty defaults | `nodeSelector: {}`, `tolerations: []` | Indicate optional, user-overridable |
| Resource quantities | `cpu: 500m`, `memory: 512Mi` | Use Kubernetes standard formats |

**Note:** While chart authors are free to use any key names they choose, following the conventions from `helm create` makes your chart familiar to users.

---

## 7.3 Value Override Precedence

Helm merges values from multiple sources. When the same key appears in more than one source, the **last source wins**. The precedence order from lowest to highest is:

### 7.3.1 Complete Precedence Order

| Priority | Source | Mechanism |
|---|---|---|
| 1 (lowest) | Chart's built-in `values.yaml` | Included in the chart package |
| 2 | Parent chart's `values.yaml` (for subcharts) | `dependencies` in `Chart.yaml` |
| 3 | User-supplied values file 1 | `-f values/common.yaml` |
| 4 | User-supplied values file 2 (overrides file 1) | `-f values/production.yaml` |
| 5 | `--set` overrides | `--set key=value` |
| 6 | `--set-string` overrides | `--set-string key=value` |
| 7 | `--set-file` overrides | `--set-file key=path/to/file` |
| 8 (highest) | `--set-json` overrides | `--set-json 'key={"k":"v"}'` |

### 7.3.2 Precedence Diagram

```
┌─────────────────────────────────────────────────────────┐
│                 VALUE OVERRIDE PRECEDENCE                │
│                    (Highest Priority)                    │
├─────────────────────────────────────────────────────────┤
│  8. --set-json  'resources={"cpu":"500m"}'              │
│                         ↑                               │
│  7. --set-file   config.nginx=./nginx.conf              │
│                         ↑                               │
│  6. --set-string  image.tag="12345"                     │
│                         ↑                               │
│  5. --set         replicaCount=3                        │
│                         ↑                               │
│  4. -f values/prod.yaml           (2nd values file)     │
│                         ↑                               │
│  3. -f values/common.yaml         (1st values file)     │
│                         ↑                               │
│  2. Parent chart values.yaml      (if subchart)         │
│                         ↑                               │
│  1. Chart's built-in values.yaml  (lowest priority)     │
├─────────────────────────────────────────────────────────┤
│                    (Lowest Priority)                     │
└─────────────────────────────────────────────────────────┘
```

### 7.3.3 Precedence Example

Given this scenario:

```yaml
# values.yaml (chart default)
replicaCount: 1
image:
  repository: nginx
  tag: "1.25.0"

# values/production.yaml (via -f)
replicaCount: 3
image:
  tag: "1.26.0"
```

And the install command:

```bash
helm install my-release ./chart \
  -f values/production.yaml \
  --set image.tag=1.27.0
```

The final merged values are:

```yaml
replicaCount: 3          # from values/production.yaml
image:
  repository: nginx       # from values.yaml (chart default — not overridden)
  tag: "1.27.0"           # from --set (highest priority)
```

---

## 7.4 Multiple Values Files

### 7.4.1 Order Matters

When multiple `-f` files are specified, they are processed **left to right**, with the **rightmost** file winning for overlapping keys:

```bash
helm install my-release ./chart \
  -f values/common.yaml \
  -f values/us-east.yaml \
  -f values/production.yaml
```

```
common.yaml  →  us-east.yaml  →  production.yaml
                                                  ↑
                                         wins on overlap
```

### 7.4.2 Layered Values Pattern

A common pattern is to layer values files:

```
values/
├── common.yaml          # Base configuration (all environments)
├── development.yaml     # Dev-specific overrides
├── staging.yaml         # Staging-specific overrides
└── production.yaml      # Production-specific overrides
```

```bash
# Dev environment
helm install my-app ./chart -f values/common.yaml -f values/development.yaml

# Production environment
helm install my-app ./chart -f values/common.yaml -f values/production.yaml

# Staging with additional inline override
helm install my-app ./chart \
  -f values/common.yaml \
  -f values/staging.yaml \
  --set replicaCount=5
```

### 7.4.3 Merging Behavior Between Files

```yaml
# values/base.yaml
database:
  host: db.example.com
  port: 5432
  pool:
    max: 10
    min: 2

# values/overrides.yaml
database:
  port: 6432          # Overwrites port
  pool:
    min: 5            # Deep merge: only overrides pool.min, preserves pool.max
```

The result is a **deep merge**:

```yaml
database:
  host: db.example.com   # from base.yaml (unchanged)
  port: 6432             # from overrides.yaml
  pool:
    max: 10              # from base.yaml (unchanged)
    min: 5               # from overrides.yaml
```

**Note:** Helm merges maps (dictionaries) deeply. It does NOT deep-merge lists/arrays. When a list key appears in a higher-precedence source, it **replaces** the entire list.

---

## 7.5 `--set` in Depth

### 7.5.1 Syntax

```
--set key1=value1,key2=value2
```

Values are separated by commas. Nested keys use dot-notation:

```bash
--set image.repository=my-registry/nginx,image.tag=v2.0.0,replicaCount=5
```

### 7.5.2 Dot-Notation for Nested Keys

```bash
# Set nested value
--set resources.limits.cpu=1000m

# Set multiple properties of the same nested object
--set service.port=8080,service.type=NodePort

# Set values with commas in them (escape the comma)
--set annotations."prometheus\.io/port"=9090
```

### 7.5.3 Escaping and Quoting

| Character | How to escape | Example |
|---|---|---|
| Comma in key or value | `\,` | `--set key="value\,with\,commas"` |
| Period in key | `\.` in key name, or use `\` | `--set "prometheus\.io/scrape"="true"` |
| Spaces in values | Use quotes | `--set name="my release"` |
| Dollar signs | Use single quotes or escape | `--set password='$3cr3t'` |

### 7.5.4 Limitations of `--set`

- **Not suitable for large configurations**: Dense values should go in a `-f` YAML file.
- **No array/object syntax**: Arrays are set index-by-index (see Section 7.9).
- **Escaping complexity**: Deeply nested keys with special characters become unwieldy.
- **No validation**: `--set` bypasses `values.schema.json` validation (the value is parsed as a string, not JSON-validated).

**Best Practice:** Use `--set` ONLY for small overrides in development or CI/CD where a dedicated values file is impractical. For anything beyond a handful of keys, use `-f` with a dedicated values file.

---

## 7.6 `--set-string`

### 7.6.1 Purpose

`--set-string` forces a value to be treated as a **string**, even if it looks like a number, boolean, or null.

### 7.6.2 When to Use

```bash
# Without --set-string: 8080 becomes an integer in .Values
helm install my-release ./chart --set service.port=8080
# .Values.service.port = 8080 (integer)

# With --set-string: 8080 stays a string
helm install my-release ./chart --set-string service.port=8080
# .Values.service.port = "8080" (string)
```

**Common use cases:**

| Scenario | Command |
|---|---|
| Version numbers that look like numbers | `--set-string image.tag="12345"` |
| Port numbers that must remain strings | `--set-string app.port="3306"` |
| Boolean-looking strings | `--set-string feature.flag="true"` |
| IDs that are all digits | `--set-string aws.accountId="123456789012"` |

**Note:** If your template uses `{{ .Values.port | int }}`, the type in `values.yaml` does not matter—the template converts it. But if your template uses `{{ .Values.port }}` directly in a context expecting a string (e.g., an env var value), the type MUST be a string.

---

## 7.7 `--set-json`

### 7.7.1 Purpose

`--set-json` allows setting complex values—objects, arrays, nested structures—using JSON notation. The value is **parsed as JSON** before being merged.

### 7.7.2 Examples

```bash
# Set a nested object
--set-json 'resources.limits={"cpu":"500m","memory":"512Mi"}'

# Set an array
--set-json 'imagePullSecrets=[{"name":"regcred"},{"name":"aws-secret"}]'

# Set a list of strings
--set-json 'hosts=["host1.example.com","host2.example.com"]'

# Set a complex nested structure
--set-json 'ingress.annotations={"kubernetes.io/ingress.class":"nginx","cert-manager.io/cluster-issuer":"letsencrypt"}'

# Set boolean and null values explicitly
--set-json 'feature.enabled=true'
--set-json 'advanced.config=null'
```

### 7.7.3 `--set-json` vs `--set` Comparison

| Feature | `--set` | `--set-json` |
|---|---|---|
| Nested objects | Dot-notation only | Full JSON objects |
| Arrays | Index-by-index: `ports[0].port=80` | Native JSON arrays: `ports=[{"port":80}]` |
| Type inference | Automatic (strings, numbers, bools) | Explicit via JSON types |
| Complexity | Simple keys | Arbitrary JSON structures |

**Recommendation:** Use `--set-json` whenever you need to set an array, a deeply nested object, or when you need explicit type control (boolean, null).

---

## 7.8 `--set-file`

### 7.8.1 Purpose

`--set-file` reads the **contents of a file** and injects them as a value. The entire file content becomes the value of the specified key.

### 7.8.2 Syntax

```bash
--set-file <key>=<path-to-file>
```

### 7.8.3 Use Cases

```bash
# Inject a certificate
--set-file tls.cert=/path/to/cert.pem

# Inject a custom Nginx config
--set-file nginx.conf=/etc/nginx/nginx.conf

# Inject a script
--set-file initScript=/opt/scripts/init.sh

# Inject multiple files
--set-file config.app=./configs/app.yaml \
  --set-file config.db=./configs/database.yaml
```

### 7.8.4 In Templates

In the template, the value behaves as a multi-line string:

```yaml
# values.yaml
config: ""
```

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}
data:
  nginx.conf: |
    {{ .Values.config | nindent 4 }}
```

```bash
helm install my-release ./chart --set-file config=./nginx.conf
```

---

## 7.9 Reusing and Resetting Values

### 7.9.1 `--reuse-values`

When upgrading a release, `--reuse-values` retains **all previously deployed values** and merges any new `-f` or `--set` values on top:

```bash
helm upgrade my-release ./chart --reuse-values --set replicaCount=5
```

**How it works:**

```
Previous deployed values (revision N)
    ↓ (kept entirely)
    ↓
+ New --set / -f values (merged on top)
    ↓
= Final deployed values (revision N+1)
```

**Warning:** `--reuse-values` does NOT pick up new defaults from the updated chart's `values.yaml`. If the chart author added a new key default in `values.yaml`, it will be absent unless you explicitly set it.

### 7.9.2 `--reset-values`

Discards **all previously deployed values**. Only the chart's built-in `values.yaml` and any new `-f`/`--set` values are used:

```bash
helm upgrade my-release ./chart --reset-values -f values/new-settings.yaml
```

**Use case:** When you want to start fresh—perhaps you are migrating from one values structure to a completely different one.

### 7.9.3 `--reset-then-reuse-values` (Helm 3.11+)

This flag combines the two: first reset to chart defaults, then re-apply the previous release values, and then apply new `-f`/`--set` overrides on top:

```bash
helm upgrade my-release ./chart --reset-then-reuse-values -f values/production.yaml
```

**Execution order:**

```
1. Reset to chart defaults (values.yaml)
2. Re-apply previous release's deployed values
3. Apply new -f values (values/production.yaml)
4. Apply new --set overrides
```

**Use case:** This is the "safest" upgrade approach: it picks up new chart defaults for keys you never explicitly set, while preserving your explicit customizations, and still allowing new overrides.

### 7.9.4 Comparison Matrix

| Flag | Old deployed values | Old chart defaults | New chart defaults | New `-f`/`--set` |
|---|---|---|---|---|
| (none) | Discarded | Discarded | Used | Used |
| `--reuse-values` | **Kept** | Discarded | **Not picked up** | Merged on top |
| `--reset-values` | **Discarded** | Discarded | Used | Used |
| `--reset-then-reuse-values` | **Kept** (re-applied) | Discarded | **Picked up** (for unset keys) | Merged on top |

---

## 7.10 Merge Behavior

### 7.10.1 Deep Merge for Maps (Dictionaries)

Helm performs a **deep merge** on nested maps across all value sources:

```yaml
# Chart values.yaml
app:
  database:
    host: localhost
    port: 5432
    ssl: false

# -f overrides.yaml
app:
  database:
    port: 5433
    ssl: true
```

**Result (deep merge):**

```yaml
app:
  database:
    host: localhost     # unchanged
    port: 5433          # overridden
    ssl: true           # overridden
```

### 7.10.2 Shallow Replace for Lists (Arrays)

Lists are NOT deep-merged. A list from a higher-precedence source **replaces** the entire list:

```yaml
# Chart values.yaml
hosts:
  - host1.example.com
  - host2.example.com

# -f overrides.yaml
hosts:
  - host3.example.com
```

**Result (replacement, not merge):**

```yaml
hosts:
  - host3.example.com
```

The original `host1.example.com` and `host2.example.com` are lost.

**Workaround:** If you need to add to a list rather than replace it, use index-based `--set` (see Section 7.11) or design your values to use maps instead of lists where feasible.

---

## 7.11 Array Handling

### 7.11.1 Index-Based Overrides with `--set`

```yaml
# Chart values.yaml
ports:
  - name: http
    port: 80
    protocol: TCP
  - name: https
    port: 443
    protocol: TCP
```

```bash
# Override the first port's value
--set ports[0].port=8080

# Override the second port's name and port
--set ports[1].name=tls,ports[1].port=8443

# Add a third port
--set ports[2].name=metrics,ports[2].port=9090,ports[2].protocol=TCP
```

### 7.11.2 Setting Arrays with `--set-json`

```bash
# Replace the entire ports array
--set-json 'ports=[{"name":"http","port":80},{"name":"metrics","port":9090}]'
```

### 7.11.3 COALESCE Function

The Helm `coalesce` function (in `_helpers.tpl`) is used to build arrays from values:

```yaml
{{/*
Return the appropriate list of image pull secrets
*/}}
{{- define "imagePullSecrets" -}}
{{- $pullSecrets := list }}
{{- range .Values.imagePullSecrets }}
{{- $pullSecrets = append $pullSecrets . }}
{{- end }}
{{- if .Values.global.imagePullSecrets }}
{{- range .Values.global.imagePullSecrets }}
{{- $pullSecrets = append $pullSecrets . }}
{{- end }}
{{- end }}
{{- if not (empty $pullSecrets) }}
imagePullSecrets:
{{- range $pullSecrets }}
  - name: {{ . }}
{{- end }}
{{- end }}
{{- end -}}
```

### 7.11.4 Limitations of Array Handling

| Limitation | Impact | Mitigation |
|---|---|---|
| No splice/insert | Cannot insert at arbitrary index | Use maps instead of arrays where possible |
| Index gaps break | `ports[0]`, `ports[2]` with no `ports[1]` → error | Always set contiguous indices |
| No "-" to append | Cannot natively append to an existing array | Use `--set-json` to replace the entire array |

---

## 7.12 Required Values

### 7.12.1 The `required` Function

In templates, use `required` to enforce that a value is provided:

```yaml
# templates/deployment.yaml
replicas: {{ required "replicaCount must be set" .Values.replicaCount }}
```

If `.Values.replicaCount` is empty or nil, Helm fails with:

```
Error: execution error at (my-chart/templates/deployment.yaml:10:14):
replicaCount must be set
```

### 7.12.2 Best Practice: Group Required Validation

```yaml
# _helpers.tpl
{{/*
Validate required values
*/}}
{{- define "mychart.validateValues" -}}
{{- if not .Values.database.host }}
{{- fail "database.host is required" }}
{{- end }}
{{- if not .Values.database.password }}
{{- fail "database.password is required (use a Secret reference, not a plaintext value)" }}
{{- end }}
{{- end -}}
```

Then call this at the top of every template that needs validation:

```yaml
{{ include "mychart.validateValues" . }}
```

---

## 7.13 `values.schema.json`

Helm supports JSON Schema validation for `values.yaml`. Create a file named `values.schema.json` in the chart root:

### 7.13.1 Schema Example

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "title": "Values",
  "type": "object",
  "required": ["replicaCount", "image"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "description": "Number of pod replicas"
    },
    "image": {
      "type": "object",
      "required": ["repository", "tag"],
      "properties": {
        "repository": {
          "type": "string",
          "description": "Container image repository"
        },
        "tag": {
          "type": "string",
          "default": "latest",
          "description": "Container image tag"
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "Never", "IfNotPresent"],
          "default": "IfNotPresent"
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer", "ExternalName"]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    },
    "autoscaling": {
      "type": "object",
      "properties": {
        "enabled": {
          "type": "boolean"
        },
        "minReplicas": {
          "type": "integer",
          "minimum": 1
        },
        "maxReplicas": {
          "type": "integer",
          "minimum": 1
        }
      },
      "if": {
        "properties": { "enabled": { "const": true } }
      },
      "then": {
        "required": ["minReplicas", "maxReplicas"]
      }
    }
  }
}
```

### 7.13.2 Schema Validation Flow

```
values.yaml + -f files + --set overrides
            ↓
    [values.schema.json]
            ↓
    Validation pass/fail
            ↓
  fail → Error message, install/upgrade aborted
  pass → Proceed to template rendering
```

### 7.13.3 Schema Benefits

- **Fail fast**: Catch invalid configurations before they reach the Kubernetes API.
- **Self-documenting**: Schema acts as living documentation for all accepted values.
- **IDE support**: Editors like VS Code can provide autocomplete for values if the schema is present.
- **Conditional validation**: Use JSON Schema `if/then/else` for context-dependent rules.

**Exam Tip:** The schema file must be named exactly `values.schema.json` and placed in the chart root directory. Helm validates values **before** rendering templates.

---

## 7.14 Environment-Specific Values

### 7.14.1 Directory Structure Pattern

```
my-chart/
├── Chart.yaml
├── values.yaml              # Production-safe defaults
├── values.schema.json
├── templates/
└── env-values/
    ├── development.yaml     # Dev overrides
    ├── staging.yaml         # Staging overrides
    └── production.yaml      # Production overrides
```

### 7.14.2 Layered Deployment

```bash
# Development: minimal resources, debug logging
helm upgrade --install my-app ./my-chart \
  -f env-values/development.yaml \
  --namespace dev --create-namespace

# Staging: moderate resources, integration endpoints
helm upgrade --install my-app ./my-chart \
  -f env-values/staging.yaml \
  --namespace staging --create-namespace

# Production: high resources, monitoring, strict security
helm upgrade --install my-app ./my-chart \
  -f env-values/production.yaml \
  --namespace prod --create-namespace \
  --atomic --wait
```

### 7.14.3 Environment Values Example

```yaml
# env-values/development.yaml
replicaCount: 1
resources:
  limits:
    cpu: 250m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
logging:
  level: debug
autoscaling:
  enabled: false

---
# env-values/production.yaml
replicaCount: 5
resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi
logging:
  level: warn
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
monitoring:
  enabled: true
```

---

## 7.15 Global Values

### 7.15.1 Sharing Values with Subcharts

The `global` key in `values.yaml` is a special namespace that is automatically available to **all subcharts**:

```yaml
# Parent chart values.yaml
global:
  imageRegistry: my-registry.example.com
  imagePullSecrets:
    - regcred
  storageClass: premium-ssd
  environment: production
```

**In a subchart template:**

```yaml
# subchart/templates/deployment.yaml
image: "{{ .Values.global.imageRegistry }}/{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### 7.15.2 Global Value Behavior

- Global values are **shared** across the parent chart and all subcharts.
- Subcharts can reference `.Values.global.*` directly.
- The parent chart sets global values; subcharts consume them.
- If a subchart defines its own `global` values in its `values.yaml`, they are overridden by the parent's global values.

### 7.15.3 Common Use Cases

| Use Case | Example Key |
|---|---|
| Registry URL | `global.imageRegistry` |
| Pull secrets | `global.imagePullSecrets` |
| Storage class | `global.storageClass` |
| Environment name | `global.environment` |
| Common labels | `global.labels` |
| DNS domain suffix | `global.domain` |

---

## 7.16 Importing Values from Subcharts

### 7.16.1 `import-values` in `Chart.yaml`

A parent chart can **import** specific values from a child (subchart) for use in its own templates:

```yaml
# parent/Chart.yaml
dependencies:
  - name: redis
    version: "18.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    import-values:
      - child: master.service.ports.redis   # Source in subchart
        parent: redisPort                    # Destination in parent

      - child: auth.password                 # Source in subchart
        parent: redisPassword               # Destination in parent
```

After import, the parent chart can use `{{ .Values.redisPort }}` and `{{ .Values.redisPassword }}` in its own templates.

### 7.16.2 Bulk Import

```yaml
# Import all exports from a subchart
dependencies:
  - name: redis
    version: "18.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    import-values:
      - child: ""       # Empty string imports all export keys
        parent: ""      # Map to same keys in parent
```

---

## 7.17 Common Mistakes with Values

### 7.17.1 Typo in `--set` Paths

```bash
# WRONG — no error, but nothing happens
--set replicaCount=5    # chart uses "replicaCount" (camelCase)

# RIGHT
--set replicaCount=5
```

Helm silently ignores unrecognized keys unless `--set` is validating against `values.schema.json`. There is no warning for typos.

### 7.17.2 YAML Parsing Errors

```yaml
# WRONG — YAML interprets values
port: 8080          # integer
password: false     # boolean, not a string!

# RIGHT — quote everything ambiguous
port: "8080"
password: "false"
```

Common YAML gotchas in values files:

| Input | YAML Interprets As | Fix |
|---|---|---|
| `version: 1.10` | `1.1` (float!) | `version: "1.10"` |
| `countries: [NO, UK]` | `[false, "UK"]` (NO = false in YAML 1.1) | `countries: ["NO", "UK"]` |
| `port: 080` | Octal number (error) | `port: "080"` |
| `timeout: 60` | Integer, fine | N/A |
| `password: yes` | Boolean `true` | `password: "yes"` |

### 7.17.3 Merge Surprises

```yaml
# values.yaml
replicas: 2

# Second file via -f
# Does NOT mention replicas at all
```

Result: `replicas` is `2` (chart default). This is expected.

```yaml
# values.yaml
resources: {}
```

```bash
# User provides
-f override.yaml
```

```yaml
# override.yaml
resources:
  limits:
    cpu: 500m
```

Result: `resources: {limits: {cpu: 500m}}` — the empty default `{}` is overwritten, not merged into.

### 7.17.4 Array Override Surprises

```yaml
# Chart default values.yaml
tolerations:
  - key: "node-role"
    operator: "Exists"
    effect: "NoSchedule"

# User provides -f
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "high-cpu"
    effect: "NoSchedule"
```

Result: Only the user's toleration survives. The chart's default toleration is **replaced**, not appended.

### 7.17.5 `--reuse-values` Forgetting New Defaults

```bash
# Chart v1 default: logLevel: info, enableMetrics: false
helm install my-release ./chart --set enableMetrics=true

# Chart v2 adds: enableTracing: false (new default)
helm upgrade my-release ./chart --reuse-values
# Result: enableTracing is UNDEFINED!
# --reuse-values skips new chart defaults.
```

**Fix:** Use `--reset-then-reuse-values` (Helm 3.11+) to pick up new chart defaults while keeping explicit overrides.

---

## 7.18 Best Practices

### 7.18.1 Never Put Sensitive Data in `values.yaml`

Values files are plain text and committed to version control. Secrets belong in:

- External secret stores (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager)
- Kubernetes Secrets managed separately
- Sealed Secrets or External Secrets Operator
- Reference them in values as pointers, not as raw data:

```yaml
# GOOD
database:
  passwordSecretName: db-password
  passwordSecretKey: password

# BAD
database:
  password: "super-secret-password"
```

### 7.18.2 Use Schema Validation

Always include a `values.schema.json` for any chart shared across teams or used in production. It catches misconfigurations before they hit the cluster.

### 7.18.3 Document Every Value

Add comments to `values.yaml`:

```yaml
# replicaCount — Number of application pod replicas.
# Minimum: 1, Maximum: 100 (enforced by values.schema.json)
replicaCount: 1
```

Or generate a `README.md` with a values table using tools like `helm-docs`.

### 7.18.4 Keep `values.yaml` Production-Safe

The default `values.yaml` should represent a **safe, functional, minimal deployment**. It should:
- Run with minimal resources.
- Not require external dependencies unless they have sensible defaults.
- Not expose services publicly by default.
- Not enable debug modes.

### 7.18.5 Layer Values, Don't Duplicate

```
values/
├── base.yaml       # Shared across all environments
├── dev.yaml        # Only dev-specific overrides
├── staging.yaml    # Only staging-specific overrides
└── prod.yaml       # Only prod-specific overrides
```

Each environment file contains ONLY differences from `base.yaml`. This avoids copy-paste drift.

### 7.18.6 Prefer `-f` Over `--set` for Production

`--set` is convenient but opaque in audit logs and hard to review. Use dedicated values files for production deployments. They are:

- Version-controlled.
- Peer-reviewable in pull requests.
- Self-documenting.
- Reusable across releases.

### 7.18.7 Test Values Before Deploying

```bash
# Validate the schema
helm lint my-chart

# Dry-run with values to catch template errors
helm template my-release my-chart -f values/production.yaml > rendered.yaml

# Check rendered output with kubectl dry-run
kubectl apply --dry-run=server -f rendered.yaml
```

---

## 7.19 Summary

- Values are the **configuration surface** of Helm charts, turning generic templates into specific deployments.
- Precedence flows: `values.yaml` (lowest) → parent chart → `-f` files → `--set` → `--set-string` → `--set-file` → `--set-json` (highest).
- Maps are **deep-merged**; arrays are **replaced** wholesale.
- `--reuse-values` preserves previous settings but skips new chart defaults; `--reset-then-reuse-values` (3.11+) is safer.
- `values.schema.json` validates values before deployment—always include it for production charts.
- Never store secrets in `values.yaml`. Reference them via Kubernetes Secret names or external secret stores.
- Layer values files by environment and use `-f` (not `--set`) for production deployments.
