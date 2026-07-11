# Chapter 8: Templating

## 8.1 Go Template Syntax

Helm templates are written using Go's `text/template` and `html/template` packages, enhanced with the **Sprig** function library (70+ additional functions).

### 8.1.1 Delimiters

Helm uses two delimiters:

| Delimiter | Purpose |
|---|---|
| `{{ }}` | Evaluate an expression, action, or variable and emit its result into the rendered output |
| `{{- -}}` | Same as `{{ }}`, but **strip adjacent whitespace** (left, right, or both) |

**Basic example:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
```

Renders to:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-release-my-chart
```

### 8.1.2 Whitespace Control

The `-` character inside delimiters trims whitespace:

| Syntax | Behavior |
|---|---|
| `{{- "hello" }}` | Trim whitespace to the **left** of the expression |
| `{{ "hello" -}}` | Trim whitespace to the **right** of the expression |
| `{{- "hello" -}}` | Trim whitespace on **both sides** |

**Example:**

```yaml
# WITHOUT whitespace control
env:
  {{ if .Values.debug }}
  - name: DEBUG
    value: "true"
  {{ end }}

# Renders (with blank lines):
env:

  - name: DEBUG
    value: "true"

```

```yaml
# WITH whitespace control
env:
  {{- if .Values.debug }}
  - name: DEBUG
    value: "true"
  {{- end }}

# Renders (clean):
env:
  - name: DEBUG
    value: "true"
```

**Rule of thumb:** Use `{{-` at the start of a block and `-}}` before a newline within a block to produce clean YAML. In YAML, extra whitespace before content can break indentation.

---

## 8.2 Built-in Objects

Helm exposes several built-in objects available in every template. These are the `.` (dot) context objects:

### 8.2.1 `.Release` — Release Information

| Field | Description | Example Value |
|---|---|---|
| `.Release.Name` | Release name | `"my-release"` |
| `.Release.Namespace` | Namespace to be released into | `"production"` |
| `.Release.IsUpgrade` | `true` if this is an upgrade operation | `true` / `false` |
| `.Release.IsInstall` | `true` if this is an install operation | `true` / `false` |
| `.Release.Revision` | The revision number (starts at 1) | `3` |
| `.Release.Service` | The service rendering the template | `"Helm"` |

**Usage pattern:**

```yaml
metadata:
  name: {{ .Release.Name }}-deployment
  namespace: {{ .Release.Namespace }}
  annotations:
    helm.sh/revision: "{{ .Release.Revision }}"
```

```yaml
# Conditionally run a migration Job only on install
{{- if .Release.IsInstall }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migration
# ...
{{- end }}
```

### 8.2.2 `.Values` — User-Supplied Configuration

The merged values from `values.yaml`, `-f` files, and `--set` flags. This is a map (dictionary).

```yaml
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

**Accessing nested values:**

```yaml
# Dot notation
{{ .Values.resources.limits.cpu }}

# Index notation (for keys with special characters or dots)
{{ index .Values "prometheus.io" "scrape" }}
```

**Checking if a value is defined:**

```yaml
{{- if hasKey .Values "customField" }}
  customField: {{ .Values.customField }}
{{- end }}
```

### 8.2.3 `.Chart` — Chart Metadata

Read from `Chart.yaml`:

| Field | Source | Example |
|---|---|---|
| `.Chart.Name` | `Chart.yaml: name` | `"nginx"` |
| `.Chart.Version` | `Chart.yaml: version` | `"1.2.3"` |
| `.Chart.AppVersion` | `Chart.yaml: appVersion` | `"1.25.0"` |
| `.Chart.Description` | `Chart.yaml: description` | `"A simple Nginx chart"` |
| `.Chart.Type` | `Chart.yaml: type` | `"application"` or `"library"` |
| `.Chart.Keywords` | `Chart.yaml: keywords` | `["web", "nginx"]` |
| `.Chart.Home` | `Chart.yaml: home` | `"https://nginx.org"` |
| `.Chart.Sources` | `Chart.yaml: sources` | `["https://github.com/..."]` |
| `.Chart.Maintainers` | `Chart.yaml: maintainers` | `[{name:"John", email:"..."}]` |
| `.Chart.Icon` | `Chart.yaml: icon` | `"https://nginx.org/icon.png"` |
| `.Chart.ApiVersion` | `Chart.yaml: apiVersion` | `"v2"` |
| `.Chart.Dependencies` | `Chart.yaml: dependencies` | List of dependency objects |
| `.Chart.Deprecated` | `Chart.yaml: deprecated` | `true` / `false` |
| `.Chart.Annotions` | `Chart.yaml: annotations` | `{category: "Infrastructure"}` |

```yaml
# Common usage
metadata:
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
```

### 8.2.4 `.Files` — Chart File Access

Access to non-template files within the chart. See Section 8.18 for full details.

### 8.2.5 `.Capabilities` — Cluster Capabilities

Information about the Kubernetes cluster:

| Path | Description | Example |
|---|---|---|
| `.Capabilities.APIVersions` | List of all API versions on the cluster | `["v1", "apps/v1", "networking.k8s.io/v1", ...]` |
| `.Capabilities.KubeVersion` | Kubernetes version object | `{Major: "1", Minor: "29", GitVersion: "v1.29.0", ...}` |
| `.Capabilities.KubeVersion.Version` | Full version string | `"v1.29.0"` |
| `.Capabilities.KubeVersion.Major` | Major version | `"1"` |
| `.Capabilities.KubeVersion.Minor` | Minor version | `"29"` |
| `.Capabilities.HelmVersion` | Helm version object | `{Version: "v3.14.0", ...}` |

```yaml
# Check Kubernetes version for API compatibility
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
# Check minimum Kubernetes version
{{- if semverCompare ">=1.25-0" .Capabilities.KubeVersion.Version }}
  # Use features available only in K8s 1.25+
{{- end }}
```

**Exam Tip:** `semverCompare` is a Sprig function that compares semantic versions. It is essential for writing charts that support multiple Kubernetes versions.

### 8.2.6 `.Template` — Template Metadata

| Field | Description | Example |
|---|---|---|
| `.Template.Name` | The template file path relative to the chart root | `"my-chart/templates/deployment.yaml"` |
| `.Template.BasePath` | The templates directory path | `"my-chart/templates"` |

```yaml
# Useful for labeling resources with their source template
metadata:
  labels:
    app.kubernetes.io/component: {{ .Template.Name | replace "templates/" "" | replace ".yaml" "" }}
```

---

## 8.3 Actions (Control Structures)

### 8.3.1 `if` / `else if` / `else`

```yaml
{{- if .Values.autoscaling.enabled }}
replicas: {{ .Values.autoscaling.minReplicas }}
{{- else if .Values.replicaCount }}
replicas: {{ .Values.replicaCount }}
{{- else }}
replicas: 1
{{- end }}
```

### 8.3.2 `with`

`with` changes the current scope (dot) for its block. Useful for reducing repetition:

```yaml
# WITHOUT with (verbose)
env:
  - name: DB_HOST
    value: {{ .Values.database.host }}
  - name: DB_PORT
    value: {{ .Values.database.port | quote }}

# WITH with (concise)
env:
  {{- with .Values.database }}
  - name: DB_HOST
    value: {{ .host }}
  - name: DB_PORT
    value: {{ .port | quote }}
  {{- end }}
```

Inside `with`, `.` refers to `.Values.database`. If `.Values.database` is empty/falsy, the entire block is skipped (just like `if`).

**Important:** `with` rebinds `.` (dot). If you need to access the root context inside `with`, use `$`:

```yaml
{{- with .Values.database }}
  - name: RELEASE_NAME
    value: {{ $.Release.Name }}   # $ reaches back to the root
  - name: DB_HOST
    value: {{ .host }}
{{- end }}
```

### 8.3.3 `range`

Iterates over slices (lists) and maps (dictionaries):

```yaml
# Iterate over a list
ports:
  {{- range .Values.servicePorts }}
  - name: {{ .name }}
    port: {{ .port }}
    protocol: {{ .protocol }}
  {{- end }}

# Iterate over a map
labels:
  {{- range $key, $value := .Values.customLabels }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

Inside `range`, `.` is set to each element. Use `$` to reference the root when needed.

**Iterating over a map with key-value assignment:**

```yaml
data:
  {{- range $filename, $content := .Values.configFiles }}
  {{ $filename }}: |
    {{ $content | nindent 4 }}
  {{- end }}
```

### 8.3.4 `define`, `template`, `include`, `block`

See Section 8.15 for a full discussion of named templates.

---

## 8.4 Pipelines

Pipelines chain operations with the `|` (pipe) operator. The left-hand value is passed as the **last argument** to the function on the right:

```yaml
# Pipeline: upper all strings, then quote the result
{{ .Values.name | upper | quote }}

# Equivalent to (without pipes):
{{ quote (upper .Values.name) }}
```

**Common pipeline patterns:**

```yaml
{{ .Values.replicaCount | default 1 }}
{{ .Values.image.tag | required "image.tag is required" }}
{{ .Values.configData | toYaml | nindent 8 }}
{{ .Files.Get "config.toml" | b64enc }}
{{ .Values.annotations | toYaml | indent 4 }}
```

**The last argument rule:** In Go templates, the piped value becomes the **last argument** to the function:

```yaml
# These are equivalent:
{{ .Values.count | add 1 }}
{{ add 1 .Values.count }}

# The piped value goes to the LAST position:
{{ .Values.text | replace "old" "new" }}
{{ replace "old" "new" .Values.text }}
```

---

## 8.5 Variables

### 8.5.1 The `$` Root Variable

`$` always refers to the **root context** (the dot passed into the template):

```yaml
{{- range .Values.ports }}
  - name: {{ $.Release.Name }}-{{ .name }}
    port: {{ .port }}
{{- end }}
```

Without `$`, referring to `.Release.Name` inside the `range` would fail because `.` is bound to the current port element.

### 8.5.2 Local Variables with `:=`

```yaml
{{- $fullname := include "mychart.fullname" . }}
{{- $labels := include "mychart.labels" . }}

metadata:
  name: {{ $fullname }}
  labels:
    {{- $labels | nindent 4 }}
```

```yaml
# Accumulator pattern inside range
{{- $total := 0 }}
{{- range .Values.items }}
  {{- $total = add $total .count }}
{{- end }}
total: {{ $total }}
```

### 8.5.3 Scope Rules

| Context | What `.` refers to |
|---|---|
| Top of template | Root context (Release, Values, Chart, Files, Capabilities, Template) |
| Inside `with` | The object passed to `with` |
| Inside `range` (list) | Each list element |
| Inside `range` (map) | Each value (or use `$key, $value :=`) |
| Inside `define`/`template` | Whatever `.` is passed when invoked |

---

## 8.6 Conditionals

### 8.6.1 Boolean Evaluation

In Go templates, the following are **falsy**:

| Value | Considered |
|---|---|
| `false` | falsy |
| `0` (integer) | falsy |
| `0.0` (float) | falsy |
| `""` (empty string) | falsy |
| `nil` (null/undefined) | falsy |
| Empty slice (`[]`) | falsy |
| Empty map (`{}`) | falsy |

Everything else is **truthy** (including the string `"false"` and non-empty slices).

### 8.6.2 Comparison Operators

| Operator | Meaning |
|---|---|
| `eq` | Equal to |
| `ne` | Not equal to |
| `lt` | Less than |
| `gt` | Greater than |
| `le` | Less than or equal to |
| `ge` | Greater than or equal to |

```yaml
{{- if eq .Values.environment "production" }}
replicas: 5
{{- else if le .Values.replicaCount 0 }}
replicas: 1
{{- else }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

**Note:** `eq` is a function, not an infix operator. Write `eq A B`, not `A == B`.

### 8.6.3 Logical Operators

```yaml
{{- if and .Values.autoscaling.enabled (gt .Values.replicaCount 1) }}
  # Scaling logic
{{- end }}

{{- if or (eq .Values.env "staging") (eq .Values.env "production") }}
  # Production-like environment
{{- end }}

{{- if not .Values.debug }}
  # Debug is disabled
{{- end }}
```

### 8.6.4 Common Idioms

```yaml
# Toggle a feature
{{- if .Values.featureX.enabled }}
  # Feature X resources
{{- end }}

# Check nested existence safely
{{- if and .Values.database .Values.database.host }}
  - name: DB_HOST
    value: {{ .Values.database.host }}
{{- end }}

# Default with condition
{{- if .Values.replicaCount }}
replicas: {{ .Values.replicaCount }}
{{- else }}
replicas: 1
{{- end }}
```

---

## 8.7 Loops

### 8.7.1 `range` Over Slices

```yaml
env:
  {{- range .Values.envVars }}
  - name: {{ .name }}
    value: {{ .value | quote }}
  {{- end }}
```

### 8.7.2 `range` Over Maps

```yaml
annotations:
  {{- range $key, $value := .Values.podAnnotations }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

### 8.7.3 Index Access Within `range`

```yaml
# Accessing array by index
{{- range $index, $element := .Values.hosts }}
  - index: {{ $index }}
    host: {{ $element }}
{{- end }}
```

### 8.7.4 Nested `range`

```yaml
ingress:
  hosts:
    {{- range .Values.ingressHosts }}
    - host: {{ .host }}
      paths:
        {{- range .paths }}
        - path: {{ .path }}
          backend:
            serviceName: {{ .serviceName }}
            servicePort: {{ .servicePort }}
        {{- end }}
    {{- end }}
```

### 8.7.5 `break` and `continue` (Helm 3.10+)

```yaml
{{- range .Values.items }}
  {{- if eq .status "done" }}
    {{- break }}
  {{- end }}
  {{- if eq .skip true }}
    {{- continue }}
  {{- end }}
  - name: {{ .name }}
{{- end }}
```

---

## 8.8 Built-in Go Template Functions

The Go `text/template` package provides these functions (plus `call`, `html`, `index`, `js`, `len`, `not`, `or`, `urlquery`):

| Function | Description | Example |
|---|---|---|
| `print` | Print arguments with spaces | `{{ print "Hello" .Name }}` |
| `printf` | Formatted print (C-style) | `{{ printf "%s:%d" .Host .Port }}` |
| `println` | Print with newline | `{{ println .Name }}` |
| `call` | Call a function by name | `{{ call .FuncName .Arg1 .Arg2 }}` |
| `index` | Index array/map by key | `{{ index .Values "my.key" }}` |
| `len` | Length of a string/slice/map | `{{ len .Values.hosts }}` |
| `and` / `not` / `or` | Boolean logic | `{{ and (gt .X 0) (lt .X 10) }}` |
| `eq` / `ne` / `lt` / `le` / `gt` / `ge` | Comparison | `{{ eq .Env "prod" }}` |

---

## 8.9 Sprig Library — Complete Function Reference

Helm includes the [Sprig library](https://masterminds.github.io/sprig/), which adds **70+ functions** beyond Go's built-ins. Below is a categorized reference with examples.

### 8.9.1 String Functions

| Function | Description | Example | Output |
|---|---|---|---|
| `quote` | Wrap in double quotes, escape special chars | `{{ "hello" \| quote }}` | `"hello"` |
| `squote` | Wrap in single quotes | `{{ "hello" \| squote }}` | `'hello'` |
| `upper` | Convert to uppercase | `{{ "hello" \| upper }}` | `HELLO` |
| `lower` | Convert to lowercase | `{{ "HELLO" \| lower }}` | `hello` |
| `title` | Title case | `{{ "hello world" \| title }}` | `Hello World` |
| `trim` | Remove whitespace from both ends | `{{ "  hi  " \| trim }}` | `hi` |
| `trimAll` | Remove all occurrences of a character set | `{{ "aabbhelloabba" \| trimAll "ab" }}` | `hello` |
| `trimPrefix` | Remove prefix if present | `{{ "https://example.com" \| trimPrefix "https://" }}` | `example.com` |
| `trimSuffix` | Remove suffix if present | `{{ "example.com/" \| trimSuffix "/" }}` | `example.com` |
| `replace` | Replace all occurrences | `{{ "1.2.3" \| replace "." "-" }}` | `1-2-3` |
| `repeat` | Repeat a string N times | `{{ repeat 3 "ha" }}` | `hahaha` |
| `contains` | Test if substring exists (returns bool) | `{{ contains "ell" "hello" }}` | `true` |
| `hasPrefix` | Test prefix | `{{ hasPrefix "http" "https://x.com" }}` | `true` |
| `hasSuffix` | Test suffix | `{{ hasSuffix ".com" "example.com" }}` | `true` |
| `nospace` | Remove all whitespace | `{{ "a b c" \| nospace }}` | `abc` |
| `abbrev` | Abbreviate with ellipsis | `{{ abbrev 5 "hello world" }}` | `he...` |
| `abbrevboth` | Abbreviate both ends | `{{ abbrevboth 5 "1234567890" }}` | `...89...` |
| `initials` | Take initials | `{{ "First Last" \| initials }}` | `FL` |
| `cat` | Concatenate strings | `{{ cat "a" "b" "c" }}` | `a b c` |
| `indent` | Indent every line with N spaces | `{{ "a\nb\nc" \| indent 2 }}` | `  a\n  b\n  c` |
| `nindent` | Indent with leading newline | `{{ "a\nb\nc" \| nindent 2 }}` | `\n  a\n  b\n  c` |
| `wrap` | Word-wrap at N columns | `{{ wrap 10 "long string here" }}` | `long\nstring\nhere` |
| `wrapWith` | Word-wrap with custom wrapper pos | `{{ wrapWith 5 "\t" "hello world" }}` | `hello\tworld` |
| `substr` | Substring by start and length | `{{ substr 0 5 "hello world" }}` | `hello` |
| `trunc` | Truncate to N chars | `{{ trunc 5 "hello world" }}` | `hello` |
| `sha256sum` | SHA-256 hash | `{{ "foo" \| sha256sum }}` | `2c26b46b...` |
| `sha1sum` | SHA-1 hash | `{{ "foo" \| sha1sum }}` | `0beec7b...` |
| `adler32sum` | Adler-32 checksum | `{{ "foo" \| adler32sum }}` | `...` |
| `camelcase` | Convert to camelCase | `{{ "hello world" \| camelcase }}` | `HelloWorld` |
| `kebabcase` | Convert to kebab-case | `{{ "Hello World" \| kebabcase }}` | `hello-world` |
| `snakecase` | Convert to snake_case | `{{ "Hello World" \| snakecase }}` | `hello_world` |
| `swapcase` | Swap character casing | `{{ "Hello" \| swapcase }}` | `hELLO` |

**Common patterns:**

```yaml
# Quoting values for YAML safety
imageTag: {{ .Values.image.tag | quote }}

# Clean up chart version for labels
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}

# Indent multi-line config
data:
  config.yaml: |
    {{ .Values.configData | nindent 4 }}

# Generate a checksum annotation to trigger pod restart
checksum/config: {{ include (print .Template.BasePath "/configmap.yaml") . | sha256sum }}
```

### 8.9.2 Math Functions

| Function | Description | Example | Output |
|---|---|---|---|
| `add` | Sum (variadic) | `{{ add 1 2 3 }}` | `6` |
| `add1` | Increment by 1 | `{{ 5 \| add1 }}` | `6` |
| `sub` | Subtract second arg from first | `{{ sub 10 3 }}` | `7` |
| `mul` | Multiply (variadic) | `{{ mul 2 3 4 }}` | `24` |
| `div` | Integer division | `{{ div 10 3 }}` | `3` |
| `mod` | Modulo | `{{ mod 10 3 }}` | `1` |
| `max` | Maximum (variadic) | `{{ max 1 5 3 9 2 }}` | `9` |
| `min` | Minimum (variadic) | `{{ min 1 5 3 9 2 }}` | `1` |
| `floor` | Round down to nearest integer | `{{ 3.7 \| floor }}` | `3` |
| `ceil` | Round up to nearest integer | `{{ 3.1 \| ceil }}` | `4` |
| `round` | Round to nearest integer | `{{ 3.4 \| round }}` | `3` |
| `sub1` | Decrement by 1 | `{{ 5 \| sub1 }}` | `4` |

```yaml
# Calculate resource percentages
memoryLimit: {{ mul .Values.resources.requests.memory 2 }}

# Port math
targetPort: {{ add .Values.service.basePort 10 }}

# Replica math
minReadyReplicas: {{ div .Values.replicaCount 2 | ceil }}
```

### 8.9.3 Date Functions

| Function | Description | Example |
|---|---|---|
| `now` | Current time (time.Time) | `{{ now \| date "2006-01-02" }}` |
| `date` | Format a time | `{{ now \| date "2006-01-02 15:04:05" }}` |
| `dateInZone` | Format in timezone | `{{ now \| dateInZone "2006-01-02" "UTC" }}` |
| `dateModify` | Modify a time | `{{ now \| dateModify "+1h" \| date "15:04" }}` |
| `htmlDate` | HTML/RFC3339 format | `{{ now \| htmlDate }}` |
| `htmlDateInZone` | HTML format in zone | `{{ now \| htmlDateInZone "UTC" }}` |
| `duration` | Format a number of seconds as duration | `{{ 300 \| duration }}` → `5m0s` |
| `durationRound` | Round duration | `{{ 305 \| durationRound "1m" }}` → `5m5s` |
| `unixEpoch` | Unix epoch seconds | `{{ now \| unixEpoch }}` |
| `ago` | Duration between time and now | `{{ ago (dateModify (now) "-2h") }}` |

```yaml
metadata:
  annotations:
    deployed-at: {{ now | date "2006-01-02T15:04:05Z07:00" }}
```

**Exam Tip:** Go's time format uses the specific reference date `Mon Jan 2 15:04:05 MST 2006` (which is `01/02 03:04:05PM '06 -0700`). Memorize the reference values: `2006` = year, `01` = month, `02` = day, `15` = hour (24h), `04` = minute, `05` = second.

### 8.9.4 List Functions

| Function | Description | Example | Output |
|---|---|---|---|
| `list` | Create a list | `{{ list 1 2 3 }}` | `[1 2 3]` |
| `first` | First element | `{{ list 1 2 3 \| first }}` | `1` |
| `last` | Last element | `{{ list 1 2 3 \| last }}` | `3` |
| `initial` | All but last | `{{ list 1 2 3 \| initial }}` | `[1 2]` |
| `rest` | All but first | `{{ list 1 2 3 \| rest }}` | `[2 3]` |
| `append` | Append to list | `{{ list 1 2 \| append 3 }}` | `[1 2 3]` |
| `prepend` | Prepend to list | `{{ list 2 3 \| prepend 1 }}` | `[1 2 3]` |
| `concat` | Concatenate lists | `{{ concat (list 1 2) (list 3 4) }}` | `[1 2 3 4]` |
| `reverse` | Reverse order | `{{ list 1 2 3 \| reverse }}` | `[3 2 1]` |
| `uniq` | Remove duplicates | `{{ list 1 2 1 3 \| uniq }}` | `[1 2 3]` |
| `without` | Remove items | `{{ list 1 2 3 4 \| without 2 3 }}` | `[1 4]` |
| `has` | Test membership | `{{ has 2 (list 1 2 3) }}` | `true` |
| `slice` | Slice a list | `{{ slice (list 1 2 3 4 5) 1 3 }}` | `[2 3]` |
| `join` | Join elements with separator | `{{ list "a" "b" "c" \| join "," }}` | `a,b,c` |
| `sortAlpha` | Sort alphabetically | `{{ list "c" "a" "b" \| sortAlpha }}` | `[a b c]` |
| `split` | Split string into list | `{{ "a,b,c" \| split "," }}` | `[a b c]` |
| `splitn` | Split with max N parts | `{{ "a,b,c" \| splitn "," 2 }}` | `[a b,c]` |
| `empty` | Check if empty | `{{ empty .Values.nested }}` | `true` / `false` |
| `compact` | Remove empty values | `{{ list 0 "" nil "a" \| compact }}` | `[0 a]` |

```yaml
# Build image pull secrets list
imagePullSecrets:
  {{- $secrets := list }}
  {{- range .Values.imagePullSecrets }}
    {{- $secrets = append $secrets .name }}
  {{- end }}
  {{- if not (empty $secrets) }}
  - name: {{ $secrets | join "," }}
  {{- end }}

# CSV to YAML list
hosts:
  {{- range (.Values.commaSeparatedHosts | split ",") }}
  - {{ . | trim | quote }}
  {{- end }}
```

### 8.9.5 Dictionary Functions

| Function | Description | Example |
|---|---|---|
| `dict` | Create a dictionary | `{{ dict "key" "value" "k2" "v2" }}` |
| `set` | Add/set a key in a dict | `{{ $_ := set .Values "newKey" "val" }}` |
| `unset` | Remove a key | `{{ $_ := unset .Values "oldKey" }}` |
| `hasKey` | Test if key exists | `{{ hasKey .Values "database" }}` |
| `pluck` | Extract a key from list of dicts | `{{ pluck "name" .Values.services }}` |
| `keys` | List all keys | `{{ keys .Values.labels }}` |
| `values` | List all values | `{{ values .Values.labels }}` |
| `merge` | Shallow merge, right wins | `{{ merge $dict1 $dict2 }}` |
| `mergeOverwrite` | Deep merge, right wins on conflict | `{{ mergeOverwrite $dict1 $dict2 }}` |
| `deepCopy` | Deep copy a dict | `{{ $copy := deepCopy .Values }}` |
| `pick` | Select specific keys | `{{ pick .Values "name" "port" }}` |
| `omit` | Remove specific keys | `{{ omit .Values "secretData" "password" }}` |

```yaml
# Conditionally build a dictionary
{{- $args := dict "host" .Values.db.host }}
{{- if .Values.db.port }}
  {{- $_ := set $args "port" .Values.db.port }}
{{- end }}
```

### 8.9.6 Type Conversion Functions

| Function | Description | Example | Output |
|---|---|---|---|
| `toString` | Convert to string | `{{ 123 \| toString }}` | `"123"` |
| `toJson` | Convert to JSON | `{{ .Values.config \| toJson }}` | Compact JSON |
| `toPrettyJson` | Pretty-printed JSON | `{{ .Values.config \| toPrettyJson }}` | Indented JSON |
| `toRawJson` | Unescaped raw JSON | `{{ .Values.config \| toRawJson }}` | Raw JSON string |
| `fromJson` | Parse JSON string → object | `{{ '{"a":1}' \| fromJson }}` | Map `{a: 1}` |
| `toYaml` | Convert to YAML | `{{ .Values.config \| toYaml }}` | YAML string |
| `fromYaml` | Parse YAML string → object | `{{ "a: 1\nb: 2" \| fromYaml }}` | Map `{a: 1, b: 2}` |
| `typeOf` | Return type as string | `{{ typeOf .Values.port }}` | `int64`, `string`, etc. |
| `kindOf` | Return reflect.Kind | `{{ kindOf .Values.port }}` | `int64`, `string`, etc. |
| `kindIs` | Test if kind matches | `{{ kindIs "int64" .Values.port }}` | `true` / `false` |
| `typeIs` | Test if type matches | `{{ typeIs "int64" .Values.port }}` | `true` / `false` |
| `toDecimal` / `toString` / `toInt` / `toInt64` / `toFloat64` | Explicit numeric conversion | `{{ "5" \| toInt \| add1 }}` | `6` |

```yaml
# Convert dict to YAML inside a ConfigMap
data:
  config.yaml: |
    {{ .Values.config \| toYaml \| nindent 4 }}
```

### 8.9.7 Encoding Functions

| Function | Description | Example |
|---|---|---|
| `b64enc` | Base64 encode (standard) | `{{ "hello" \| b64enc }}` → `aGVsbG8=` |
| `b64dec` | Base64 decode | `{{ "aGVsbG8=" \| b64dec }}` → `hello` |
| `b32enc` | Base32 encode | `{{ "hello" \| b32enc }}` |
| `b32dec` | Base32 decode | `{{ ... \| b32dec }}` |

```yaml
# Secret with base64-encoded value
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
data:
  config: {{ .Files.Get "config.toml" | b64enc }}
```

### 8.9.8 Network Functions

| Function | Description |
|---|---|
| `getHostByName` | Perform DNS lookup, return IP |

```yaml
{{ getHostByName "example.com" }}
```

### 8.9.9 File Path Functions

| Function | Description | Example |
|---|---|---|
| `base` | Filename from path | `{{ base "/path/to/file.txt" }}` → `file.txt` |
| `dir` | Directory from path | `{{ dir "/path/to/file.txt" }}` → `/path/to` |
| `ext` | File extension | `{{ ext "/path/to/file.txt" }}` → `.txt` |
| `clean` | Clean path | `{{ clean "/path//to/../" }}` → `/path` |
| `isAbs` | Check if absolute path | `{{ isAbs "/etc/passwd" }}` → `true` |
| `osBase` | OS-specific basename | `{{ osBase }}` |
| `osClean` | OS-specific path clean | `{{ osClean }}` |
| `osDir` | OS-specific dirname | `{{ osDir }}` |
| `osExt` | OS-specific extension | `{{ osExt }}` |
| `osIsAbs` | OS-specific absolute check | `{{ osIsAbs }}` |

### 8.9.10 Crypto Functions

| Function | Description |
|---|---|
| `sha256sum` | SHA-256 hash |
| `sha1sum` | SHA-1 hash |
| `adler32sum` | Adler-32 checksum |
| `bcrypt` | Bcrypt hash |
| `htpasswd` | Apache htpasswd entry |
| `derivePassword` | Derive password deterministically |
| `randAlphaNum` | Random alphanumeric string |
| `randAlpha` | Random alphabetic |
| `randNumeric` | Random numeric |
| `randAscii` | Random printable ASCII |
| `uuidv4` | Random UUID v4 |

```yaml
# Checksum for config change detection
checksum/config: {{ include (print .Template.BasePath "/configmap.yaml") . | sha256sum }}

# Generate a random password on install
{{- if .Release.IsInstall }}
password: {{ randAlphaNum 32 | b64enc }}
{{- end }}
```

**Warning:** `randAlphaNum` and similar random functions regenerate on every template execution. Do not use them for values that must be stable across upgrades. For persistent secrets, use `lookup` to check if a Secret exists first.

### 8.9.11 Semantic Version Functions

| Function | Description | Example |
|---|---|---|
| `semver` | Parse a semantic version | `{{ semver "1.2.3" }}` |
| `semverCompare` | Compare two semvers | `{{ semverCompare ">=1.0.0" .Capabilities.KubeVersion.Version }}` |

```yaml
# Conditionally use API based on K8s version
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.Version }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
```

### 8.9.12 Reflection Functions

| Function | Description |
|---|---|
| `typeOf` | Return the type as a string |
| `kindOf` | Return the reflect.Kind |
| `kindIs` | Test reflect.Kind |
| `typeIsLike` | Test if type is like another |

---

## 8.10 Named Templates

### 8.10.1 `define` and `template`

`define` creates a named template. `template` invokes it:

```yaml
# _helpers.tpl
{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

# templates/deployment.yaml (invocation)
metadata:
  labels:
    {{- template "mychart.labels" . }}
```

### 8.10.2 `include` vs `template`

| Feature | `template` | `include` |
|---|---|---|
| Returns a value? | No (writes directly to output) | **Yes** (returns a string) |
| Can be used in pipelines? | No | **Yes** |
| Can be assigned to variable? | No | **Yes** |

```yaml
# template: CANNOT be piped
{{ template "mychart.labels" . }}

# include: CAN be piped and processed further
{{ include "mychart.labels" . | nindent 4 }}

# include: CAN be assigned to a variable
{{- $labels := include "mychart.labels" . }}
```

**Rule:** Always use `include` instead of `template` unless you are certain you will never need to pipe or assign the result.

### 8.10.3 `block`

`block` is a combination of `define` and `template`. It defines a template and immediately renders it. If the template is later redefined by the caller, the redefinition wins:

```yaml
{{- define "mychart.defaultLabels" -}}
app: {{ .Chart.Name }}
{{- end -}}

{{- block "mychart.labels" . }}
  {{- include "mychart.defaultLabels" . | nindent 2 }}
{{- end -}}
```

`block` is less commonly used in Helm than `include`, but it is useful for providing default content that can be overridden.

---

## 8.11 The `tpl` Function

`tpl` evaluates a **string as a Go template**. This allows values themselves to contain template directives:

### 8.11.1 Use Case: User-Supplied Templates in Values

```yaml
# values.yaml
hostname: "{{ .Release.Name }}.{{ .Values.domain }}"
domain: "example.com"

# templates/deployment.yaml
hostname: {{ tpl .Values.hostname . }}

# Renders to:
hostname: my-release.example.com
```

### 8.11.2 Use Case: Dynamic Templates Stored in Values

```yaml
# values.yaml
customAnnotations: |
  {{- range .Values.tags }}
  tags.{{ .key }}: {{ .value }}
  {{- end }}

tags:
  - key: environment
    value: production
  - key: team
    value: platform

# templates/deployment.yaml
metadata:
  annotations:
    {{- tpl .Values.customAnnotations . | nindent 4 }}
```

### 8.11.3 `tpl` Usage Patterns

```yaml
# Expanding a single templated value
{{ tpl .Values.nginxConf . }}

# Expanding and indent
{{ tpl .Values.config . | indent 4 }}

# Expanding from a file
{{ tpl (.Files.Get "config.tpl") . }}
```

**Note:** Always pass `.` (the root context) as the second argument to `tpl`, so that the evaluated template has access to `.Release`, `.Values`, `.Chart`, etc.

---

## 8.12 `required`

`required` ensures a value is non-empty. If the value is empty, template rendering fails with the given error message:

```yaml
{{ required "image.repository is required" .Values.image.repository }}
```

If `.Values.image.repository` is `""` or `nil`, Helm halts with:

```
Error: execution error at (mychart/templates/deployment.yaml):
image.repository is required
```

### 8.12.1 Pattern: Required Validation Block

```yaml
# _helpers.tpl
{{- define "mychart.validate" -}}
{{ required "A valid .Values.image.tag is required" .Values.image.tag }}
{{ required "A valid .Values.service.port is required" .Values.service.port }}
{{- end -}}
```

### 8.12.2 `required` with `fail`

```yaml
{{- if not .Values.database.host }}
  {{- fail "database.host is required" }}
{{- end }}
```

Both `required` and `fail` stop rendering. Use `required` for single-value checks; use `fail` for complex conditional logic.

---

## 8.13 `default`

`default` provides a fallback value when the primary value is empty:

```yaml
{{ .Values.replicaCount | default 1 }}
{{ .Values.image.pullPolicy | default "IfNotPresent" }}
{{ .Values.nested.missing | default "fallback" }}
```

**Important:** `default` only triggers on Go's empty values (`""`, `0`, `false`, `nil`, empty slice, empty map). It does NOT distinguish between "not set" and "set to the zero value." Be cautious with `default` on numeric keys where `0` is a valid value.

---

## 8.14 `lookup`

`lookup` queries the **Kubernetes API** from within a template. It uses the same credentials and RBAC permissions as the Helm user.

### 8.14.1 Syntax

```
lookup "apiVersion" "kind" "namespace" "name"
```

All arguments are strings. Omit `namespace` and `name` to list all resources.

### 8.14.2 Examples

```yaml
# Look up a specific Secret
{{- $existing := lookup "v1" "Secret" .Release.Namespace "my-secret" }}
{{- if $existing }}
  # Secret exists; extract its data
  password: {{ index $existing.data "password" | b64dec }}
{{- else }}
  # Secret does not exist; create it
{{- end }}

# List all ConfigMaps in a namespace
{{- range (lookup "v1" "ConfigMap" .Release.Namespace "").items }}
  - {{ .metadata.name }}
{{- end }}

# Look up a Deployment to get current replica count
{{- $deploy := lookup "apps/v1" "Deployment" .Release.Namespace (include "mychart.fullname" .) }}
{{- if $deploy }}
replicas: {{ $deploy.spec.replicas }}
{{- else }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

### 8.14.3 Use Cases

| Use Case | Example |
|---|---|
| Idempotent secret generation | Check if Secret exists with a random password; only generate on install |
| External dependency check | Verify that a required CRD or Secret exists before deploying |
| Migration patterns | Read old resource state before migrating to a new chart |
| Dynamic configuration | Read a ConfigMap's values and inject them into a new resource |

### 8.14.4 Security Implications

**Warning:** `lookup` makes template rendering non-deterministic—the output depends on the state of the cluster at render time. This breaks the "chart as a pure function" paradigm:

- `helm template` (offline rendering) cannot evaluate `lookup` and returns an empty result.
- CI/CD pipelines that render templates outside the cluster will not see the same output.
- `lookup` requires RBAC permissions beyond what the chart's resources need.

**Best Practice:** Minimize `lookup` usage. Prefer defining interfaces (e.g., `existingSecret` pattern) where users explicitly provide references to pre-existing resources.

---

## 8.15 `fail`

`fail` unconditionally stops template rendering with a custom error message:

```yaml
{{- fail "This chart requires Kubernetes 1.25+" }}
```

```yaml
# Conditional fail
{{- if and .Values.database.enabled (not .Values.database.host) }}
  {{- fail "database.enabled is true but database.host is not provided" }}
{{- end }}
```

Use `fail` sparingly. Prefer `required` for simple value checks.

---

## 8.16 Indentation and Whitespace Control

### 8.16.1 `indent` and `nindent`

```yaml
# indent: adds N spaces to the beginning of every line
{{ .Values.text | indent 4 }}

# nindent: adds a newline, THEN N spaces to every line
{{ .Values.text | nindent 4 }}
```

**`nindent` is the YAML workhorse.** In YAML, when you include a multi-line value inside a block scalar (using `|`), the value needs to be indented. `nindent` handles both the newline and the indentation:

```yaml
# WITHOUT nindent (manual, error-prone)
data:
  config.yaml: |
    {{ .Values.config | indent 4 }}    # Missing leading newline!

# WITH nindent (correct)
data:
  config.yaml: |
    {{ .Values.config | nindent 4 }}
```

### 8.16.2 Trim Markers `{{-` and `-}}`

| Pattern | Effect |
|---|---|
| `{{- ... }}` | Eat whitespace **before** the expression |
| `{{ ... -}}` | Eat whitespace **after** the expression |
| `{{- ... -}}` | Eat whitespace on **both sides** |

```yaml
# Without pruning: extra blank line
env:
  {{ if .Values.debug }}
  - name: DEBUG
    value: "true"
  {{ end }}

# With pruning: clean output
env:
  {{- if .Values.debug }}
  - name: DEBUG
    value: "true"
  {{- end }}
```

### 8.16.3 Whitespace Control Patterns

```yaml
# Pattern 1: Block that may be empty
metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- if .Values.extraLabels }}
    {{- toYaml .Values.extraLabels | nindent 4 }}
  {{- end }}

# Pattern 2: Range with commas (YAML flow syntax)
ports: [{{- range $i, $p := .Values.ports }}{{ if $i }}, {{ end }}{{ $p.port }}{{- end }}]

# Pattern 3: Clean conditionals
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "mychart.serviceAccountName" . }}
{{- end }}
```

---

## 8.17 `.Files` Object

The `.Files` object provides access to non-template, non-excluded files within the chart (configs, scripts, certificates, etc.).

### 8.17.1 `.Files.Get`

Reads the **contents** of a file as a string:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  binaryData:
    logo.png: {{ .Files.Get "files/logo.png" | b64enc }}
```

### 8.17.2 `.Files.Glob`

Returns a list of file paths matching a glob pattern:

```yaml
data:
  {{- range $path, $_ := .Files.Glob "configs/*.yaml" }}
  {{ base $path }}: |
    {{ $.Files.Get $path | nindent 4 }}
  {{- end }}
```

### 8.17.3 `.Files.AsConfig`

Converts files into a ConfigMap-compatible map. Files are stored under their base filename, with content as the value:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-files
data:
  {{- (.Files.Glob "configs/*.properties").AsConfig | nindent 2 }}
```

Renders as:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-release-files
data:
  app.properties: |
    server.port=8080
    ...
  db.properties: |
    jdbc.url=jdbc:postgresql://...
    ...
```

### 8.17.4 `.Files.AsSecrets`

Same as `AsConfig`, but Base64-encodes all file content:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret-files
data:
  {{- (.Files.Glob "secrets/*.key").AsSecrets | nindent 2 }}
```

### 8.17.5 `.Files.Lines`

Returns the lines of a file as a slice:

```yaml
{{- range .Files.Lines "config/domains.txt" }}
  - {{ . | trim }}
{{- end }}
```

### 8.17.6 File Access Limitations

- Only files within the chart directory are accessible.
- Templates (`.yaml`, `.tpl` files in `templates/`) are excluded.
- Files excluded by `.helmignore` are not accessible.
- File size is unlimited in memory, but Secret and ConfigMap have 1 MB limits.

---

## 8.18 Debugging Templates

### 8.18.1 `helm template`

Renders templates locally without contacting the Kubernetes API:

```bash
# Render all templates to stdout
helm template my-release ./my-chart -f values/production.yaml

# Render specific templates
helm template my-release ./my-chart -s templates/deployment.yaml

# Validate against cluster API versions
helm template my-release ./my-chart --validate

# Include CRDs in output
helm template my-release ./my-chart --include-crds

# Debug mode (show rendered templates with source context)
helm template my-release ./my-chart --debug
```

### 8.18.2 `helm install --dry-run --debug`

The most thorough pre-deploy check. It validates templates against the cluster API, checks resource types, and shows the rendered manifests:

```bash
helm install my-release ./my-chart --dry-run --debug -f values/production.yaml
```

### 8.18.3 Common Debugging Techniques

| Problem | Debug Approach |
|---|---|
| Template not rendering | Add `{{ printf "DEBUG: %v" .Values.someKey }}` temporarily |
| Incorrect type | Use `{{ typeOf .Values.someKey }}` to check the type |
| Whitespace issues | Pipe output to `cat -A` to see hidden characters |
| Missing values | Use `{{ .Values \| toPrettyJson }}` to dump all values |
| Unexpected merge | Run `helm get values <release> --all` |

### 8.18.4 `helm lint`

Checks the chart for issues, including template rendering:

```bash
helm lint ./my-chart --with-subcharts --strict
```

---

## 8.19 Complete Real-World Template Examples

### 8.19.1 Deployment

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "mychart.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "mychart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          {{- with .Values.livenessProbe }}
          livenessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.readinessProbe }}
          readinessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- range .Values.env }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 8.19.2 Service

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      {{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
  selector:
    {{- include "mychart.selectorLabels" . | nindent 4 }}
```

### 8.19.3 ConfigMap

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}-config
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
data:
  {{- with .Values.configData }}
  config.yaml: |
    {{- toYaml . | nindent 4 }}
  {{- end }}
  {{- range $key, $value := .Values.configFiles }}
  {{ $key }}: |
    {{- $value | nindent 4 }}
  {{- end }}
```

### 8.19.4 Secret

```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "mychart.fullname" . }}-secret
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
type: Opaque
data:
  {{- if .Values.secret.data }}
  {{- range $key, $value := .Values.secret.data }}
  {{ $key }}: {{ $value | b64enc | quote }}
  {{- end }}
  {{- end }}
  {{- if .Values.secret.existingSecret }}{{/* Use external secret */}}{{ end }}
```

### 8.19.5 Ingress

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "mychart.fullname" . -}}
{{- $svcPort := .Values.service.port -}}
{{- if and .Values.ingress.className (not (semverCompare ">=1.18-0" .Capabilities.KubeVersion.Version)) }}
  {{- if not (hasKey .Values.ingress.annotations "kubernetes.io/ingress.class") }}
    {{- $_ := set .Values.ingress.annotations "kubernetes.io/ingress.class" .Values.ingress.className }}
  {{- end }}
{{- end }}
apiVersion: {{ if semverCompare ">=1.19-0" .Capabilities.KubeVersion.Version }}networking.k8s.io/v1{{ else }}networking.k8s.io/v1beta1{{ end }}
kind: Ingress
metadata:
  name: {{ $fullName }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if and .Values.ingress.className (semverCompare ">=1.18-0" .Capabilities.KubeVersion.Version) }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            {{- if semverCompare ">=1.19-0" $.Capabilities.KubeVersion.Version }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ $fullName }}
                port:
                  number: {{ $svcPort }}
            {{- else }}
            backend:
              serviceName: {{ $fullName }}
              servicePort: {{ $svcPort }}
            {{- end }}
          {{- end }}
    {{- end }}
{{- end }}
```

### 8.19.6 HorizontalPodAutoscaler

```yaml
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled -}}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "mychart.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "mychart.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### 8.19.7 PodDisruptionBudget

```yaml
# templates/pdb.yaml
{{- if .Values.podDisruptionBudget.enabled -}}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "mychart.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  {{- if .Values.podDisruptionBudget.minAvailable }}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- else if .Values.podDisruptionBudget.maxUnavailable }}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
{{- end }}
```

### 8.19.8 ServiceAccount

```yaml
# templates/serviceaccount.yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "mychart.serviceAccountName" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

### 8.19.9 `_helpers.tpl` (Supporting Template)

```yaml
# templates/_helpers.tpl
{{/*
Expand the name of the chart.
*/}}
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this.
*/}}
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

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "mychart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "mychart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "mychart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

---

## 8.20 Template Best Practices

### 8.20.1 Use Named Templates in `_helpers.tpl`

- Centralize reusable template logic in `_helpers.tpl`.
- Every named template should have a comment block describing its purpose.
- Use the chart's name as a namespace prefix: `mychart.fullname`, `mychart.labels`, etc.

### 8.20.2 Consistent Naming

```yaml
# GOOD: consistent camelCase, clear prefixes
replicaCount: 3
autoscaling:
  enabled: true
  minReplicas: 2
podAnnotations:
  prometheus.io/scrape: "true"

# BAD: inconsistent styles
replica_count: 3
AutoScaling:
  Enable: true
  MinReplicas: 2
Pod-Annotations:
  prom/scrape: "true"
```

### 8.20.3 Avoid Business Logic in Templates

Templates should be declarative, not procedural. Avoid:

```yaml
# BAD: complex logic in template
{{- if and (eq .Values.env "production") (gt .Values.replicaCount 3) (not .Values.useHPA) }}
  {{- /* ... */}}
{{- end }}
```

Instead, restructure values to make templates simpler:

```yaml
# GOOD: push decisions to values
replicas: {{ .Values.deployConfig.replicas }}
```

### 8.20.4 Use `include`, Not `template`

`include` returns a string that can be piped, stored in a variable, or nindent'd. `template` cannot. Always prefer `include`.

### 8.20.5 Quote Paths with Special Characters

```yaml
# Use index for keys with dots or special chars
{{ index .Values "prometheus.io/scrape" }}

# Or assert the key in values.schema.json
```

### 8.20.6 Provide Sensible Defaults

Every value should have a reasonable default in `values.yaml`. The chart should run out of the box with:

```bash
helm install my-release ./my-chart
```

### 8.20.7 Use `semverCompare` for API Version Compatibility

Always condition the `apiVersion` field on the cluster's Kubernetes version for resources that changed between versions (Ingress, PDB, CronJob, etc.).

### 8.20.8 Limit `lookup` Usage

`lookup` makes charts non-portable and non-deterministic. Reserve it for when there is truly no other way (e.g., idempotent random secret generation).

---

## 8.21 Common Template Mistakes and Fixes

### 8.21.1 Missing Space After `{{`

```yaml
# WRONG
{{.Values.replicaCount}}

# RIGHT
{{ .Values.replicaCount }}
```

The space after `{{` is required by the Go template parser.

### 8.21.2 Indentation Broken by `include`

```yaml
# WRONG — include output starts at column 0
metadata:
  labels:
{{ include "mychart.labels" . }}

# RIGHT — use nindent to align under 'labels:'
metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```

### 8.21.3 Using `eq` Incorrectly

```yaml
# WRONG — eq is a function, not an operator
{{ if .Values.enabled eq true }}

# RIGHT
{{ if eq .Values.enabled true }}

# Also right (boolean shorthand)
{{ if .Values.enabled }}
```

### 8.21.4 Forgetting `$` Inside `range` or `with`

```yaml
# WRONG — .Release is not accessible inside range
{{- range .Values.ports }}
  - name: {{ .Release.Name }}-{{ .name }}
{{- end }}

# RIGHT — use $ to reference root
{{- range .Values.ports }}
  - name: {{ $.Release.Name }}-{{ .name }}
{{- end }}
```

### 8.21.5 YAML Type Coercion in `--set`

```yaml
# values.yaml
image:
  tag: "latest"

# Template
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

# --set sends integer, template inserts "12345" without quotes
# If tag is numeric, YAML may parse it as an integer
```

**Fix:** Use `{{ .Values.image.tag | quote }}` to always emit a quoted string.

### 8.21.6 Null Pointer When Accessing Nested Undefined Values

```yaml
# WRONG — panics if .Values.resources is nil or undefined
cpu: {{ .Values.resources.limits.cpu }}

# RIGHT — use with to check nesting safely
{{- with .Values.resources }}
  {{- with .limits }}
cpu: {{ .cpu }}
  {{- end }}
{{- end }}

# RIGHT — use default with deep nesting
cpu: {{ ((.Values.resources).limits).cpu | default "500m" }}
```

### 8.21.7 `required` With Nested Paths

```yaml
# WRONG — required fails on the entire chain, not just the leaf
{{ required "tag required" .Values.image.tag }}
# Panics if .Values.image is nil

# RIGHT — use with for safety
{{- with .Values.image }}
  {{ required "image.tag is required" .tag }}
{{- end }}
```

### 8.21.8 `tpl` Forgetting the Context

```yaml
# WRONG — no context, .Release etc. are unavailable
{{ tpl .Values.templateStr }}

# RIGHT — pass . as the second argument
{{ tpl .Values.templateStr . }}
```

### 8.21.9 Misplaced `-end` Causing YAML Errors

```yaml
# WRONG — the - in {{- end }} eats the newline, breaking YAML
spec:
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  {{- end }}   # This is extra/unmatched!

# RIGHT — each {{ end }} must match an opening {{ if }}/{{ with }}/{{ range }}
```

---

## 8.22 Summary

- Helm templates use Go's `text/template` language enhanced with the **Sprig** function library (70+ functions).
- Six built-in objects are available: `.Release`, `.Values`, `.Chart`, `.Files`, `.Capabilities`, `.Template`.
- `include` is preferred over `template` because it returns a string that can be piped.
- Use `nindent` for injecting multi-line content into YAML blocks.
- The `tpl` function evaluates a string as a template—useful for user-supplied template strings in values.
- `lookup` queries the Kubernetes API but breaks determinism; use it sparingly.
- Debug with `helm template --debug`, `helm install --dry-run --debug`, and `helm lint`.
- Centralize reusable template logic in `_helpers.tpl` with the `define` action.
- Always use `require` or `fail` to validate mandatory values before template rendering.
- Write templates that are declarative, deterministic, and independent of cluster state.
