# Chapter 23: Interview Questions

Helm interview questions organized by difficulty — from fundamentals to deep architectural reasoning. Each entry includes what the interviewer is looking for, a model answer, and the key points a candidate must mention.

---

## EASY — Fundamentals (30 Questions)

---

### Q1. What is Helm and why is it needed?

**What the interviewer is looking for:** Understanding of Helm's role at the 10,000-foot level — the problem it solves versus raw Kubernetes manifests.

**Model answer:**

Helm is the package manager for Kubernetes. Kubernetes provides primitive resources (Deployments, Services, Ingresses) but does not prescribe how to author, version, package, distribute, or manage the lifecycle of groups of resources that form an application. Helm fills this gap by providing:

- **Templating** — one parameterized chart works across dev, staging, and prod without copy-paste.
- **Versioning** — charts carry SemVer; every deployment receives a monotonically increasing revision number.
- **Lifecycle management** — `install`, `upgrade`, `rollback`, `uninstall`, `history`, `status`, `test`.
- **Distribution** — charts are shared via HTTP repositories or OCI registries.

Without Helm, a team running 50 microservices across 3 environments manages 150 sets of copy-pasted YAML with manual find-and-replace, no rollback, and inevitable configuration drift.

**Key points to mention:**
- Kubernetes is declarative but doesn't package applications.
- Helm = the `apt`/`brew`/`npm` equivalent for Kubernetes.
- Solves templating, versioning, sharing, and lifecycle management.

---

### Q2. What is a Helm chart?

**What the interviewer is looking for:** You can describe the unit of distribution in Helm and enumerate its components.

**Model answer:**

A Helm chart is a collection of files that describe a related set of Kubernetes resources. It is the packaging format for Helm. A chart is structured as a directory containing:

- `Chart.yaml` — metadata (name, version, app version, dependencies).
- `values.yaml` — default configuration values.
- `templates/` — Go-templated Kubernetes YAML manifests.
- `templates/NOTES.txt` — post-install help text displayed to the user.
- `templates/_helpers.tpl` — reusable named template fragments.
- `charts/` — dependent subcharts (fetched by `helm dependency update`).
- `crds/` — Custom Resource Definitions (installed before templates, never updated by upgrades).
- `.helmignore` — file patterns to exclude when packaging.
- Optionally: `values.schema.json`, `README.md`, `LICENSE`.

When packaged, a chart becomes a `.tgz` archive. It is the unit that gets pushed to repositories, pulled by users, and instantiated as releases in clusters.

**Key points to mention:**
- Describe the directory structure.
- `Chart.yaml` is mandatory; `templates/` holds parameterized manifests.
- `crds/` has special behavior — installed first, never upgraded.
- Library charts (`type: library`) provide reusable template code but cannot be installed standalone.

---

### Q3. What is a Helm release?

**What the interviewer is looking for:** Understanding of the instantiation concept — chart + values = release.

**Model answer:**

A release is an instance of a chart running in a Kubernetes cluster. It is the combination of a specific chart, a specific set of values, a namespace, and a release name.

```
Release = Chart + Values + Namespace + Release Name
```

Each time you run `helm install`, `helm upgrade`, or `helm rollback`, a new revision of the release is created. Helm stores release metadata as Kubernetes Secrets (default) in the release's namespace, tracking the manifest, values, and chart that were used. This enables deterministic rollback — you can return to any previous revision because its full state is preserved immutably.

The same chart can be installed multiple times in different namespaces or with different release names, producing independent releases (e.g., `myapp-prod` and `myapp-dev`).

**Key points to mention:**
- Release = chart instantiation with specific values in a specific namespace.
- Each install/upgrade/rollback creates a new revision.
- Release state is persisted in Secrets (`sh.helm.release.v1.<name>.v<rev>`).
- Rollback is deterministic because every revision's manifest is stored immutably.

---

### Q4. What is a Helm repository?

**What the interviewer is looking for:** You understand the two types of chart distribution mechanisms.

**Model answer:**

A Helm repository is a collection of packaged Helm charts that can be discovered and downloaded. There are two types:

**Traditional HTTP repositories** — a static HTTP server serving an `index.yaml` catalogue file alongside `.tgz` chart packages. The `index.yaml` lists every chart, every version, and the URL to each package. Commands: `helm repo add`, `helm repo update`, `helm search repo`.

**OCI registries** — Helm charts stored as OCI artifacts in any OCI-compliant container registry (Docker Hub, ECR, GCR, ACR, Harbor). This is the modern, recommended approach. Commands: `helm registry login`, `helm push`, `helm pull oci://...`.

OCI registries are preferred for production because they offer content-addressable storage (SHA256 digests), signing and verification (cosign/notation), fine-grained access control, and reuse existing container registry infrastructure.

**Key points to mention:**
- Two types: traditional HTTP repos and OCI registries.
- Traditional: `index.yaml` + `.tgz` files on an HTTP server.
- OCI: stored as artifacts in container registries.
- OCI is the modern recommendation.

---

### Q5. Difference between chart version and appVersion

**What the interviewer is looking for:** Understanding of the dual-versioning model and when each is bumped.

**Model answer:**

Both are fields in `Chart.yaml`:

| Field | Meaning | Example | Bumped when... |
|-------|---------|---------|----------------|
| `version` | SemVer of the **chart package** itself | `1.2.3` | Template logic, defaults, dependencies, or chart structure change |
| `appVersion` | Version of the **application** running inside the containers | `3.2.1` | The container image version changes |

They are decoupled because you can fix a template bug or add a health probe (chart change) without releasing a new application version, and vice versa. In templates, `.Chart.AppVersion` is often used as the default image tag:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

**Key points to mention:**
- `version` = chart package version (template/config changes).
- `appVersion` = application/container image version.
- Decoupled: chart fixes don't require new app releases, and app updates may not require chart changes.
- `helm history` displays both columns.

---

### Q6. What is values.yaml?

**What the interviewer is looking for:** Understanding of the configuration layer that makes charts reusable.

**Model answer:**

`values.yaml` is the default configuration file in a Helm chart. It contains the default values that are substituted into templates during rendering. It is the chart author's way of providing sensible defaults so that a user can install the chart with minimal configuration (`helm install my-release ./mychart`).

Values are accessed in templates via the `.Values` object:

```yaml
replicas: {{ .Values.replicaCount }}
```

Users override these defaults through multiple mechanisms (highest precedence first):
1. `--set` / `--set-string` CLI flags
2. `--values` / `-f` custom values files
3. Parent chart's values (for subcharts)
4. `values.yaml` (lowest precedence)

**Key points to mention:**
- Default configuration for the chart.
- Accessed in templates via `.Values`.
- Can be overridden at install/upgrade time.
- Should contain only the most common/default settings.

---

### Q7. What are Helm templates?

**What the interviewer is looking for:** Understanding of the rendering engine and Go template syntax.

**Model answer:**

Helm templates are files in the `templates/` directory of a chart that produce Kubernetes YAML manifests when rendered. They use Go's `text/template` syntax extended with the Sprig function library (70+ functions) and Helm-specific built-in objects (`.Release`, `.Chart`, `.Values`, `.Capabilities`, `.Template`, `.Files`).

Templates enable parameterization — a single Deployment template works across all environments by substituting values at render time:

```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Files starting with `_` (e.g., `_helpers.tpl`) are partials — they define named templates but are not rendered standalone. `NOTES.txt` is a special template whose output is displayed to the user after install/upgrade.

**Key points to mention:**
- Go `text/template` + Sprig + Helm built-in objects.
- Parameterization enables one template for all environments.
- `_` prefix = partial templates, not rendered standalone.
- `NOTES.txt` = special template for user-facing output.

---

### Q8. What is the difference between helm install and helm upgrade?

**What the interviewer is looking for:** Understanding of the release lifecycle — creation vs. mutation.

**Model answer:**

`helm install` creates a **new** release. It renders the chart with the provided values, applies the manifest to the cluster, and stores the release as revision 1. If a release with the same name already exists, `helm install` fails.

`helm upgrade` modifies an **existing** release. It compares the new manifest with the previously deployed manifest, computes a three-way strategic merge patch, applies only the changes, and creates a new revision (incrementing the revision number). If the release does not exist, `helm upgrade` fails unless `--install` is specified.

`helm upgrade --install` combines both — install if the release doesn't exist, upgrade if it does. This is the preferred command for CI/CD pipelines (`helm upgrade --install` is idempotent).

**Key points to mention:**
- `install` = create new release; fails if name already taken.
- `upgrade` = mutate existing release; fails if release doesn't exist (unless `--install`).
- `upgrade --install` = idempotent; preferred for CI/CD.
- Each creates a new revision.

---

### Q9. How do you list all releases?

**What the interviewer is looking for:** Basic operational CLI fluency.

**Model answer:**

```bash
helm list                 # releases in the current namespace
helm list --all           # all releases regardless of status
helm list --all-namespaces # releases across all namespaces
helm list -A              # shorthand for --all-namespaces
helm list --namespace prod # releases in a specific namespace
helm list --failed        # only failed releases
helm list --deployed      # only deployed releases
helm list --superseded    # only superseded releases
helm list --uninstalled   # releases that were uninstalled but have history
```

Output includes name, namespace, revision, status, chart version, and app version.

**Key points to mention:**
- `helm list` is scoped to the current namespace by default.
- `-A` / `--all-namespaces` for cluster-wide view.
- Status filters: `--deployed`, `--failed`, `--superseded`, `--uninstalled`.
- Know both `-A` and `--all-namespaces`.

---

### Q10. How do you uninstall a release?

**What the interviewer is looking for:** Understanding of release cleanup and the `--keep-history` flag.

**Model answer:**

```bash
helm uninstall my-release              # uninstall from default namespace
helm uninstall my-release -n prod      # uninstall from specific namespace
helm uninstall my-release --keep-history  # keep release history for auditing
helm uninstall my-release --dry-run    # preview what would be deleted
```

`helm uninstall` deletes all Kubernetes resources that were created by the release and removes the release's history Secrets (revisions). Use `--keep-history` to retain history records after resource deletion, which is useful for auditing or if you intend to reinstall later.

**Key points to mention:**
- Deletes all resources created by the release.
- By default, also deletes release history.
- `--keep-history` retains history for auditing.
- `--dry-run` previews without executing.

---

### Q11. What is helm repo add?

**What the interviewer is looking for:** Understanding of traditional repository registration.

**Model answer:**

`helm repo add` registers a traditional Helm chart repository (HTTP server with `index.yaml`) on your local machine. It stores the repository name and URL in the local Helm configuration, enabling subsequent `helm search repo` and `helm install` from that repository.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable
```

After adding a repository, you must run `helm repo update` to fetch the latest `index.yaml` before searching or installing. The repository list is stored in the local Helm cache, typically at `~/.cache/helm/repository/`.

This is distinct from `helm registry login`, which authenticates with an OCI registry.

**Key points to mention:**
- Registers a traditional HTTP chart repository locally.
- Follow with `helm repo update` to fetch the index.
- Distinct from OCI registry login.
- Configuration stored in local Helm cache.

---

### Q12. What is helm repo update?

**What the interviewer is looking for:** Understanding of the index synchronization mechanism.

**Model answer:**

`helm repo update` fetches the latest `index.yaml` from every added repository and updates the local cache. Without this, `helm search repo` and `helm install` from repositories may return outdated results or fail because the local index doesn't reflect charts published after the last update.

```bash
helm repo update                     # update all repositories
helm repo update bitnami             # update a specific repository
```

The local repository cache is stored at `~/.cache/helm/repository/`. Each repository gets a cached copy of its `index.yaml`. When you run `helm search repo`, Helm queries this local cache, not the remote server directly.

**Key points to mention:**
- Fetches latest `index.yaml` from all registered repos.
- Updates local cache; searches query the cache, not remote.
- Run before `helm install` from a repository to ensure latest versions.
- Analogy: `apt update` before `apt install`.

---

### Q13. How do you search for charts?

**What the interviewer is looking for:** Chart discovery fluency.

**Model answer:**

```bash
helm search hub wordpress       # search ArtifactHub (global Helm chart index)
helm search repo nginx          # search locally cached repositories
helm search repo bitnami/nginx  # search a specific repository
helm search repo --versions     # show all available versions
helm search repo --regex "^nginx" # search with regex
```

`helm search hub` queries ArtifactHub (https://artifacthub.io), the CNCF's central index, which aggregates charts from hundreds of repositories. No local repository configuration is needed.

`helm search repo` searches the local repository cache (repositories you've added with `helm repo add` and updated with `helm repo update`).

**Key points to mention:**
- `search hub` = ArtifactHub (global, no local config needed).
- `search repo` = local cache of added repos.
- `--versions` shows all chart versions.
- `--regex` for pattern-based search.

---

### Q14. What is helm rollback?

**What the interviewer is looking for:** Understanding of Helm's revision-based rollback mechanism.

**Model answer:**

`helm rollback` reverts a release to a previous revision by re-applying the manifest that was stored at that revision. It creates a **new** revision (it does not delete or overwrite intermediate revisions).

```bash
helm rollback my-release 3         # rollback to revision 3
helm rollback my-release 3 --dry-run  # preview without applying
helm rollback my-release 0         # rollback to the previous revision
```

Key behavior:
- Revision numbers only increase — a rollback to revision 3 when you're at revision 5 creates revision 6, whose manifest is identical to revision 3's manifest.
- The rollback is deterministic because every revision's full manifest is stored immutably.
- `helm rollback my-release 0` is a shortcut meaning "roll back one revision" (to the last superseded revision).

**Key points to mention:**
- Reverts to a previous revision's manifest.
- Rollback creates a **new** revision (numbers always increment).
- `0` = rollback to previous revision.
- Deterministic because all revision manifests are stored immutably.
- Dependencies at the target revision are used.

---

### Q15. How do you check release history?

**What the interviewer is looking for:** Operational awareness of revision tracking.

**Model answer:**

```bash
helm history my-release               # full history (default: up to history-max)
helm history my-release --max 5       # last 5 revisions
helm history my-release -n prod       # from specific namespace
```

Output:

```
REVISION  UPDATED                   STATUS      CHART         APP VERSION  DESCRIPTION
1         Mon Jan 15 10:02:14 2026  superseded  myapp-1.0.0   1.0          Install complete
2         Mon Jan 15 10:05:22 2026  superseded  myapp-1.1.0   1.1          Upgrade complete
3         Mon Jan 15 14:30:01 2026  deployed    myapp-1.2.0   1.2          Upgrade complete
```

Each row is a revision. The `STATUS` column shows the current state: `deployed` (the current live revision), `superseded` (previous revisions), `failed`, `pending-install`, etc. `helm history` reads the release Secrets stored in the namespace (`sh.helm.release.v1.<name>.v<rev>`).

**Key points to mention:**
- Shows all revisions with timestamps, status, chart version, app version.
- `--max` limits output.
- Reads from release Secrets in the namespace.
- Essential for auditing and deciding which revision to roll back to.

---

### Q16. What is helm lint?

**What the interviewer is looking for:** Chart quality and validation awareness.

**Model answer:**

`helm lint` examines a chart directory for issues that would prevent it from installing correctly. It validates:

- `Chart.yaml` for required fields and correct formatting.
- Template syntax — catches Go template errors (missing `end`, invalid pipelines, undefined values).
- YAML structure — ensures rendered output is valid YAML.
- Kubernetes resource validation — checks that resource `kind` and `apiVersion` are valid.
- Best practices — missing `app.kubernetes.io/` labels, missing `readinessProbe`, etc.

```bash
helm lint ./mychart                       # lint chart in directory
helm lint ./mychart --strict              # treat warnings as errors
helm lint ./mychart --values values-prod.yaml  # lint with specific values
```

A clean lint is a prerequisite for CI/CD pipelines. It should be part of every pull request that modifies chart templates.

**Key points to mention:**
- Validates Chart.yaml, template syntax, YAML structure, Kubernetes resource validity.
- `--strict` escalates warnings to errors.
- Should be run in CI/CD before packaging or deploying.
- Does not check runtime behavior — only static analysis.

---

### Q17. What is helm template?

**What the interviewer is looking for:** Understanding of the rendering-only command for debugging and GitOps.

**Model answer:**

`helm template` renders a chart's templates locally and outputs the resulting Kubernetes YAML manifest to stdout — without contacting the Kubernetes API server and without creating or modifying any release. It is purely a local rendering operation.

```bash
helm template my-release ./mychart                    # render with default values
helm template my-release ./mychart -f values-prod.yaml # render with overrides
helm template my-release ./mychart --set key=value     # render with --set
helm template my-release ./mychart --debug             # verbose output
```

Unlike `helm install`, `helm template` does not:
- Create a release record.
- Validate against the Kubernetes API server.
- Execute hooks.
- Store values or manifests as Secrets.

It is commonly used in GitOps workflows (e.g., with ArgoCD or Flux that render Helm charts themselves) and for debugging template output before actual deployment.

**Key points to mention:**
- Local-only rendering — no API server interaction.
- Does not create releases or revision history.
- Used for GitOps (pre-rendered manifests in Git) and debugging.
- Different from `helm get manifest`, which retrieves the manifest of an installed release.

---

### Q18. What is NOTES.txt in a chart?

**What the interviewer is looking for:** Awareness of the user-facing post-install documentation mechanism.

**Model answer:**

`NOTES.txt` is a special template file in the `templates/` directory. Its rendered content is displayed to the user after `helm install` and `helm upgrade`. It is the post-installation message — it typically provides:

- Instructions for accessing the application (service URL, ingress hostname).
- Credentials for default accounts.
- Next steps (how to connect, how to configure).
- Links to documentation.

```yaml
# templates/NOTES.txt
Thank you for installing {{ .Chart.Name }}!

Your application is now running at:
  http://{{ .Values.ingress.host }}

To get the admin password:
  kubectl get secret --namespace {{ .Release.Namespace }} {{ .Release.Name }}-admin -o jsonpath="{.data.password}" | base64 -d

For more information, visit: https://docs.example.com
```

`NOTES.txt` uses the same Go template syntax as other templates and has access to all built-in objects (`.Release`, `.Chart`, `.Values`, etc.).

**Key points to mention:**
- Special template displayed after install/upgrade.
- Provides user guidance: how to access, credentials, next steps.
- Full Go template syntax available.
- Also shown by `helm get notes <release>`.

---

### Q19. What is _helpers.tpl?

**What the interviewer is looking for:** Understanding of template reuse and the partials convention.

**Model answer:**

`_helpers.tpl` is a conventional file (not mandatory, but universally adopted) in the `templates/` directory that contains reusable named template definitions — called partials or helpers. The `_` prefix prevents Helm from rendering it as a standalone Kubernetes manifest.

It typically defines:

```yaml
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name (.Chart.Name) | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

These are invoked in other templates using `{{ include "mychart.labels" . }}` or `{{ template "mychart.name" . }}`.

**Key points to mention:**
- Conventional file for reusable named template definitions.
- `_` prefix prevents standalone rendering.
- Contains `define`/`end` blocks for labels, names, selectors.
- Used via `include` (preferred, supports pipelines) or `template`.
- DRY principle — define once, use everywhere in the chart.

---

### Q20. What are Helm hooks?

**What the interviewer is looking for:** Understanding of lifecycle events and the annotation-based hook mechanism.

**Model answer:**

Helm hooks are Kubernetes resources (typically Jobs) annotated to execute at specific points in the release lifecycle. They are defined as regular templates but carry special annotations that tell Helm when to run them and how to clean up.

**Available hook events:**

| Hook | When It Runs | Use Case |
|------|-------------|----------|
| `pre-install` | Before resources are created | Schema migrations, pre-flight checks |
| `post-install` | After resources are created | Data seeding, notifications |
| `pre-delete` | Before resource deletion | Backup before cleanup |
| `post-delete` | After resource deletion | Cleanup notifications |
| `pre-upgrade` | Before upgrade is applied | Schema migrations |
| `post-upgrade` | After upgrade is applied | Cache warming, notifications |
| `pre-rollback` | Before rollback | Validation |
| `post-rollback` | After rollback | Cleanup |
| `test` | When `helm test` is run | Integration tests |

**Hook annotations:**

```yaml
annotations:
  "helm.sh/hook": pre-upgrade,pre-install
  "helm.sh/hook-weight": "5"                # execution order (lower = first)
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

Hook weights determine execution order within the same event. Delete policies control when hook resources are cleaned up (`before-hook-creation`, `hook-succeeded`, `hook-failed`).

**Key points to mention:**
- Annotation-based mechanism; hooks are regular Kubernetes resources.
- 9 hook events covering install, upgrade, delete, rollback, and test.
- Hook weights control execution order (lower = first, default 0).
- Delete policies determine cleanup behavior.
- Hook resources are NOT deleted by `helm uninstall` unless delete policy is set.

---

### Q21. What is helm test?

**What the interviewer is looking for:** Understanding of the built-in testing mechanism for releases.

**Model answer:**

`helm test` runs tests defined in a chart against a deployed release. Tests are Kubernetes resources (typically Pods or Jobs) annotated with `"helm.sh/hook": test`. These test resources run after the release is deployed and are meant to validate that the application is functioning correctly.

```bash
helm test my-release              # run tests for a release
helm test my-release --logs       # show test pod logs
helm test my-release --timeout 5m # set a timeout
```

A test Pod definition looks like:

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ .Release.Name }}-service:{{ .Values.service.port }}']
  restartPolicy: Never
```

Helm waits for the test Pod to reach the `Succeeded` phase. If it fails, `helm test` exits with a non-zero code. Tests run sequentially, not in parallel.

**Key points to mention:**
- Runs test resources annotated with `"helm.sh/hook": test`.
- Tests are typically Pods or Jobs that validate connectivity or functionality.
- `--logs` streams test pod logs.
- Sequential execution; success requires Pods to reach `Succeeded`.
- Useful for basic smoke-testing in CI/CD.

---

### Q22. What is .helmignore?

**What the interviewer is looking for:** Understanding of packaging controls.

**Model answer:**

`.helmignore` is a file at the root of a chart directory that specifies files and directories to exclude when packaging the chart with `helm package`. It uses the same syntax as `.gitignore`.

```
# .helmignore
.git
.gitignore
.idea/
.vscode/
*.swp
*.bak
*.orig
.DS_Store
```

When `helm package ./mychart` runs, it creates a `.tgz` file containing the chart directory, but files matching patterns in `.helmignore` are omitted. This keeps the package lean and prevents accidental inclusion of editor files, OS metadata, or sensitive local configuration.

**Key points to mention:**
- Same syntax as `.gitignore`.
- Specifies files/directories to exclude from packaged `.tgz`.
- Keeps packages clean — excludes editor files, build artifacts, secrets.
- Located at chart root.

---

### Q23. How do you pass custom values during install?

**What the interviewer is looking for:** Practical knowledge of value override mechanisms.

**Model answer:**

There are three ways to pass custom values during `helm install` (or `helm upgrade`):

1. **Values file(s):** `-f` or `--values`

```bash
helm install my-release ./mychart -f values-prod.yaml
helm install my-release ./mychart -f values-base.yaml -f values-prod.yaml  # multiple
```

Multiple files are merged left to right (last file wins).

2. **CLI inline:** `--set`

```bash
helm install my-release ./mychart --set replicaCount=3
helm install my-release ./mychart --set image.tag=v2.1.0,service.type=LoadBalancer
```

3. **CLI string:** `--set-string` (forces string type, prevents YAML type inference)

```bash
helm install my-release ./mychart --set-string image.tag=latest  # stays string, not coerced
```

Precedence (highest to lowest):
```
--set-string > --set > -f (last file) > -f (first file) > values.yaml
```

**Key points to mention:**
- `-f` for values files (can use multiple; merged left to right).
- `--set` for inline overrides (comma-separated key=value pairs).
- `--set-string` to force string type.
- Precedence order matters.

---

### Q24. What is Chart.yaml?

**What the interviewer is looking for:** Knowledge of the chart metadata specification.

**Model answer:**

`Chart.yaml` is the mandatory metadata file at the root of every Helm chart. It describes the chart's identity, version, dependencies, and type.

**Required fields:**
- `apiVersion: v2` — the Helm API version (v2 for Helm 3).
- `name` — chart name (lowercase, alphanumeric with hyphens).
- `version` — SemVer of the chart package.

**Key optional fields:**
- `appVersion` — version of the application inside the chart.
- `description` — short description.
- `type: application` or `type: library` (library charts cannot be installed standalone).
- `dependencies` — list of subchart dependencies with name, version, repository, condition, and tags.
- `maintainers` — list of maintainers with name and email.
- `keywords` — search keywords for ArtifactHub.
- `annotations` — arbitrary metadata.
- `kubeVersion` — SemVer constraint for compatible Kubernetes versions.
- `icon` — URL to an icon image.
- `deprecated` — boolean to mark chart as deprecated.

**Key points to mention:**
- Mandatory: `apiVersion`, `name`, `version`.
- Required for Helm 3: `apiVersion: v2`.
- `type: application` vs `type: library`.
- `dependencies` declared here; fetched by `helm dependency update`.

---

### Q25. What is Chart.lock?

**What the interviewer is looking for:** Understanding of dependency locking for reproducible builds.

**Model answer:**

`Chart.lock` is an auto-generated lock file produced by `helm dependency update` (or `helm dependency build`). It records the exact versions of all dependencies that were fetched, including their SHA256 digests. It is the Helm equivalent of `package-lock.json` (npm) or `Gemfile.lock` (Bundler).

```yaml
# Chart.lock
dependencies:
  - name: postgresql
    version: 12.1.0
    repository: https://charts.bitnami.com/bitnami
    digest: sha256:abc123def456...
  - name: redis
    version: 18.0.0
    repository: https://charts.bitnami.com/bitnami
    digest: sha256:789ghi012jkl...
```

Chart.lock should be committed to version control. It ensures that every developer, CI pipeline, and deployment environment uses the exact same dependency versions, producing reproducible builds. Without it, `helm dependency update` could fetch newer patch versions, leading to inconsistency.

**Key points to mention:**
- Auto-generated dependency lock file.
- Records exact versions and SHA256 digests of dependencies.
- Should be committed to Git for reproducible builds.
- Analogous to `package-lock.json` / `Gemfile.lock`.

---

### Q26. What is helm dependency update?

**What the interviewer is looking for:** Understanding of dependency resolution and fetching.

**Model answer:**

`helm dependency update` downloads chart dependencies declared in `Chart.yaml` into the `charts/` directory and generates/updates `Chart.lock`. It resolves dependencies based on the version constraints in `Chart.yaml` and fetches the `.tgz` archives from the specified repositories.

```bash
helm dependency update ./mychart       # download deps and update Chart.lock
helm dependency build ./mychart        # rebuild from Chart.lock (no remote fetch)
helm dependency list ./mychart         # list dependencies with status
```

Difference between `update` and `build`:
- `helm dependency update` fetches from remote repositories and updates `Chart.lock`.
- `helm dependency build` rebuilds the `charts/` directory using the versions locked in `Chart.lock` (no remote fetch). If `Chart.lock` doesn't exist, it behaves like `update`.

After running, the `charts/` directory contains `.tgz` files for each dependency, which are then used when installing or packaging the parent chart.

**Key points to mention:**
- Downloads dependencies into `charts/` directory.
- Generates/updates `Chart.lock` with exact versions.
- `helm dependency update` fetches from remotes.
- `helm dependency build` rebuilds from lock file.
- Run before `helm install` or `helm package` from a chart with dependencies.

---

### Q27. What is --dry-run?

**What the interviewer is looking for:** Understanding of the preview/simulation flag.

**Model answer:**

`--dry-run` simulates an operation without actually applying changes to the cluster. It renders templates, validates against the Kubernetes API, and displays what would be created, updated, or deleted — but nothing is persisted.

```bash
helm install my-release ./mychart --dry-run       # show what would be installed
helm upgrade my-release ./mychart --dry-run       # show what would change
helm uninstall my-release --dry-run               # show what would be deleted
```

When combined with `--debug`, `--dry-run` also shows computed values and rendered templates. This is invaluable for:
- Verifying template rendering before deployment.
- Catching YAML syntax errors.
- Seeing the computed values (especially with multiple `-f` files and `--set` flags).
- Auditing changes before production deployments.

**Key points to mention:**
- Simulates operation without mutating the cluster.
- Shows rendered manifests and computed values (with `--debug`).
- Essential for pre-deployment verification.
- Server-side validation against the Kubernetes API (unlike `helm template`, which is local-only).

---

### Q28. What does --atomic do?

**What the interviewer is looking for:** Understanding of automatic rollback on failure.

**Model answer:**

`--atomic` makes a `helm install` or `helm upgrade` operation atomic — if the operation fails, Helm automatically rolls back to the previous successful revision. This eliminates the risk of a partial deployment leaving the application in a broken state.

```bash
helm upgrade my-release ./mychart --atomic
helm upgrade my-release ./mychart --atomic --timeout 5m
```

Behavior:
- Helm monitors the deployment until it succeeds or the `--timeout` expires.
- If the operation fails (timeout or error), Helm automatically performs a rollback to the last successful revision.
- The failed revision is still recorded in history (marked as `failed`) and a rollback revision is created.
- `--atomic` implies `--wait`.

**When to use:** In CI/CD pipelines for automated deployments where a failed upgrade must not leave the production system in a broken state.

**Key points to mention:**
- Automatically rolls back on failure.
- Implies `--wait`.
- Failed revision recorded in history; rollback creates new revision.
- Essential for CI/CD automation in production.

---

### Q29. What does --wait do?

**What the interviewer is looking for:** Understanding of synchronous deployment verification.

**Model answer:**

`--wait` makes `helm install` or `helm upgrade` wait until all Kubernetes resources are in a ready state before completing the command. It monitors Pods, Deployments, StatefulSets, DaemonSets, and Jobs created by the release, and blocks until they are ready or the `--timeout` expires (default 5 minutes).

```bash
helm install my-release ./mychart --wait
helm install my-release ./mychart --wait --timeout 10m
```

Without `--wait`, `helm install` returns as soon as the resources are accepted by the Kubernetes API server, not when they are actually running. With `--wait`, the Helm command blocks until all Pods are running, all readiness probes pass, and all Deployments reach their desired replica counts.

**Key points to mention:**
- Blocks until created resources are actually ready.
- Monitors Pods, Deployments, StatefulSets, DaemonSets, Jobs.
- Default timeout: 5 minutes (configurable with `--timeout`).
- Without `--wait`, Helm returns after API server acceptance (not actual readiness).

---

### Q30. How do you specify a namespace in Helm?

**What the interviewer is looking for:** Namespace management fluency.

**Model answer:**

There are three ways to specify a namespace:

1. **`--namespace` / `-n` flag:**

```bash
helm install my-release ./mychart --namespace prod
helm list --namespace prod
```

2. **`--create-namespace` flag:** Creates the namespace if it doesn't exist (no need for a separate `kubectl create ns`).

```bash
helm install my-release ./mychart --namespace prod --create-namespace
```

3. **Kubeconfig context default:** If no namespace is specified, Helm uses the namespace from the current kubeconfig context. You can check this with `kubectl config view --minify`.

```bash
kubectl config set-context --current --namespace=staging
helm install my-release ./mychart   # uses "staging" namespace
```

Important: In Helm, the namespace is embedded in the release metadata. Two releases with the same name in different namespaces are independent. The release Secrets (`sh.helm.release.v1.<name>.v<rev>`) are stored in the release namespace.

**Key points to mention:**
- `-n` / `--namespace` sets the target namespace.
- `--create-namespace` auto-creates the namespace.
- Default: namespace from current kubeconfig context.
- Release Secrets are stored in the release namespace.

---

## MEDIUM — Architecture & Operations (40 Questions)

---

### Q31. How does Helm store release information internally?

**What the interviewer is looking for:** Deep understanding of the storage backend — Secrets, ConfigMaps, their structure, and the encoded payload.

**Model answer:**

Helm stores release information as Kubernetes resources in the release's namespace. By default, it uses **Secrets** (can be configured to use ConfigMaps via `HELM_DRIVER=configmap`). Each revision of a release is stored as a separate Secret.

**Naming convention:**
```
sh.helm.release.v1.<release-name>.v<revision>
```
Example: `sh.helm.release.v1.my-release.v3`

**Secret structure:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sh.helm.release.v1.my-release.v1
  namespace: prod
  labels:
    owner: helm
    status: superseded              # deployed | superseded | failed | pending-install
    name: my-release
    version: "1"
    modifiedAt: "1705284134"
    chart: mychart-1.0.0
type: helm.sh/release.v1
data:
  release: <base64(gzip(JSON))>     # the release payload
```

The `data.release` field contains a **base64-encoded, gzip-compressed JSON** document with the full release information: name, info (status, deployment timestamps), chart metadata, merged configuration values, all template files, the rendered manifest, version number, and namespace.

**Inspecting a release Secret:**
```bash
kubectl get secret sh.helm.release.v1.my-release.v1 -n prod \
  -o jsonpath='{.data.release}' | base64 -d | gunzip | jq .
```

**Key points to mention:**
- Secrets by default; ConfigMaps as alternative via `HELM_DRIVER`.
- Naming: `sh.helm.release.v1.<name>.v<rev>`.
- Data is base64-encoded gzip-compressed JSON.
- Contains full chart, values, and manifest for the revision.
- Labels encode status, version, and chart metadata.

---

### Q32. Explain the three-way strategic merge in Helm

**What the interviewer is looking for:** Understanding of how Helm computes the diff between old manifest, new manifest, and live state.

**Model answer:**

Helm's three-way strategic merge is the algorithm used during `helm upgrade` to determine what changes to apply. It compares three versions of the manifest:

1. **The old manifest** — the full rendered manifest from the previous revision (stored in the release Secret).
2. **The new manifest** — the freshly rendered manifest from the current chart + values.
3. **The live state** — the actual state of each resource in the Kubernetes cluster (fetched from the API server).

**The merge logic:**

```
For each resource in the union of old + new manifests:

1. If resource is in old AND in new:
   -> Apply a three-way patch:
     - Fields present in new but NOT in old -> ADD (desired change)
     - Fields present in old but NOT in new -> REMOVE (user deleted from template)
     - Fields present in BOTH old and new, but different -> UPDATE
     - Fields present in live but NOT in either old or new -> PRESERVE (live mutation)

2. If resource is in old but NOT in new:
   -> DELETE the resource (user removed it from the chart)

3. If resource is in new but NOT in old:
   -> CREATE the resource (user added it to the chart)
```

The critical insight: by comparing against the **old manifest** (not just the live state), Helm distinguishes between:
- Fields the user intentionally removed from the template (present in old, absent in new -> should be removed).
- Fields added externally (present in live, absent in both old and new -> should be preserved).

**Key points to mention:**
- Three sources: old manifest, new manifest, live state.
- Old manifest = previous revision's stored manifest.
- Live state = actual resources in the cluster.
- Preserves live mutations not in either manifest.
- More sophisticated than `kubectl apply`'s two-way merge.
- Enables detection of intentional removals from templates.

---

### Q33. What is the difference between --set and --set-string?

**What the interviewer is looking for:** Understanding of YAML type coercion and when it matters.

**Model answer:**

Both `--set` and `--set-string` pass values at the command line, but they differ in **type handling**:

- `--set` lets Helm/YAML parser infer the type. Numeric-looking strings become numbers, `true`/`false` become booleans, `null` becomes null.

```bash
--set image.tag=2.1      # becomes the number 2.1 (float), not string "2.1"
--set enabled=true       # becomes boolean true
--set replicaCount=3     # becomes integer 3
```

- `--set-string` forces the value to be a **string** regardless of how it looks.

```bash
--set-string image.tag=2.1      # becomes string "2.1"
--set-string replicaCount=3     # becomes string "3"
--set-string enabled=true       # becomes string "true"
```

**When it matters:**

```yaml
# Template expects a string
image: "myapp:{{ .Values.image.tag }}"

# --set image.tag=2.1
# -> image: "myapp:2.1"   <- Works by accident because YAML quotes it

# But:
# --set image.tag=3.0
# -> .Values.image.tag becomes the number 3, not string "3.0"
# Use --set-string for version tags that look like numbers
```

The real problem emerges with tags like `"3.0"` where type ambiguity exists. YAML parses `3.0` as the number `3`. Use `--set-string` when you absolutely need a string value.

**Key points to mention:**
- `--set` infers YAML type (numbers, booleans, nulls).
- `--set-string` forces string type.
- Critical for version tags like `3.0` (becomes `3` with `--set`).
- Use `--set-string` for values that must remain strings.

---

### Q34. How does value precedence work in Helm?

**What the interviewer is looking for:** Complete understanding of the layered value resolution order.

**Model answer:**

Helm merges values from multiple sources with a strict precedence order (highest priority wins):

```
Highest  <- --set-string key=value
            --set key=value
            Multiple -f/--values files (rightmost wins)
            Single -f/--values file
            Parent chart's subchart values (specific key)
            Parent chart's global values
Lowest   <- Subchart's own values.yaml
            Chart's values.yaml
```

**Detailed explanation:**

1. **`values.yaml`** — the chart author's defaults. The foundation.

2. **Parent chart overrides** (for subcharts) — the parent's `values.yaml` can override subchart values either via the subchart's key (e.g., `postgresql.auth.password`) or via `global:` keys that all subcharts inherit.

3. **User-provided values files** (`-f` / `--values`) — multiple files merge left to right; the rightmost file's values take precedence over earlier files. For nested objects, it's a deep merge; for lists, it's a full replacement.

4. **`--set` overrides** — inline CLI values override anything in files.

5. **`--set-string` overrides** — identical to `--set` but forces string type. Highest precedence over `--set` for the same key.

**Important nuance about lists:** Helm does **not** merge lists — it replaces them entirely. If `values.yaml` defines `env: [{name: FOO, value: bar}]` and you override with `--set env[0].name=BAZ`, the entire `env` list is replaced, not merged.

**Key points to mention:**
- Strict precedence chain: values.yaml -> parent -> -f files -> --set -> --set-string.
- Multiple `-f` files merge left to right (last wins).
- Lists are replaced, not merged.
- `global:` values propagate to all subcharts.
- Use `helm get values --all` to see the fully merged values.

---

### Q35. What are OCI registries in Helm and how do they differ from traditional repos?

**What the interviewer is looking for:** Understanding of the modern chart distribution mechanism vs the legacy approach.

**Model answer:**

OCI (Open Container Initiative) registries store Helm charts as OCI artifacts alongside container images. This is the modern, recommended distribution mechanism, replacing traditional HTTP Helm repositories.

**Traditional Helm repositories:**
- Simple HTTP server with an `index.yaml` file listing charts.
- Charts are stored as `.tgz` files alongside `index.yaml`.
- Discovery: `helm repo add` + `helm repo update`.
- Integrity: SHA256 in `index.yaml` (manually computed; can be stale).

**OCI registries:**
- Charts stored as OCI artifacts in container registries (Docker Hub, ECR, GCR, ACR, Harbor, GHCR).
- Tags function as chart versions.
- Integrity: Content-addressable storage (SHA256 digests built into the OCI spec).
- Authentication: `helm registry login` — same credentials as container images.
- Built-in support for signing (cosign, notation).

**Commands comparison:**

```bash
# Traditional
helm repo add myrepo https://charts.example.com
helm repo update
helm install my-release myrepo/mychart

# OCI
helm registry login registry-1.docker.io
helm install my-release oci://registry-1.docker.io/myorg/mychart --version 1.0.0
helm pull oci://registry-1.docker.io/myorg/mychart --version 1.0.0
helm push mychart-1.0.0.tgz oci://registry-1.docker.io/myorg/
```

**Why OCI is preferred:**
- Reuses existing container registry infrastructure (no separate chart server needed).
- Content-addressable storage ensures integrity.
- Fine-grained access control (IAM, RBAC from the registry).
- Signing ecosystem (cosign/notation for supply chain security).
- No `helm repo update` needed — tags are resolved directly.

**Key points to mention:**
- OCI = charts as artifacts in container registries.
- Traditional = HTTP server with `index.yaml`.
- OCI provides better integrity, authentication, and signing.
- OCI is the modern recommendation.
- `helm registry login` vs `helm repo add`.

---

### Q36. Explain Helm's template engine (Go templates + Sprig)

**What the interviewer is looking for:** Technical depth on the rendering engine beyond basic usage.

**Model answer:**

Helm's template engine is a combination of three layers:

**1. Go's `text/template` package** — provides the core template syntax:
- `{{ }}` for actions (print, conditionals, loops).
- `{{ .Values.foo }}` for accessing the data context (dot notation).
- Pipelines: `{{ .Values.name | upper | quote }}`.
- Control flow: `{{ if }}`, `{{ else }}`, `{{ end }}`, `{{ range }}`, `{{ with }}`.
- Variables: `{{ $var := .Values.foo }}`.
- Named templates: `{{ define }}`, `{{ template }}`, `{{ include }}`.
- Whitespace control: `{{-` and `-}}`.

**2. Sprig library** — 70+ functions covering:
- String: `upper`, `lower`, `trim`, `replace`, `quote`, `squote`, `nindent`, `indent`, `contains`, `hasPrefix`, `hasSuffix`, `trunc`, `substr`, `camelcase`, `snakecase`, `kebabcase`.
- Math: `add`, `sub`, `mul`, `div`, `max`, `min`, `ceil`, `floor`, `round`.
- Type: `toString`, `toJson`, `toYaml`, `fromJson`, `fromYaml`, `typeOf`, `kindOf`, `empty`, `ternary`, `default`, `required`, `coalesce`.
- Date: `now`, `date`, `dateModify`, `htmlDate`, `duration`.
- List: `list`, `first`, `last`, `append`, `prepend`, `concat`, `reverse`, `uniq`, `without`, `has`, `slice`, `join`, `sortAlpha`, `dict`, `keys`, `values`, `merge`, `mergeOverwrite`.
- Encoding: `b64enc`, `b64dec`, `sha256sum`, `regexMatch`, `regexFindAll`, `regexReplaceAll`.
- Flow: `fail`, `required`, `coalesce`, `ternary`, `default`, `empty`.

**3. Helm-specific built-in objects:**
- `.Release` — Release name, namespace, service, revision, IsInstall, IsUpgrade.
- `.Chart` — Chart.yaml contents.
- `.Values` — Merged configuration values.
- `.Capabilities` — Kubernetes server version and API resources.
- `.Template` — Current template metadata (name, base path).
- `.Files` — Access to non-template files in the chart (via `.Files.Get`, `.Files.Glob`).
- `lookup` function — query live Kubernetes API from within a template.

**Key points to mention:**
- Three layers: Go text/template + Sprig functions + Helm built-in objects.
- Pipelines chain functions together.
- `include` vs `template`: `include` supports pipelines and whitespace control.
- `nindent` is critical for correct YAML indentation.
- `tpl` function evaluates a string as a template (for dynamic template generation).

---

### Q37. What are library charts?

**What the interviewer is looking for:** Understanding of the code reuse pattern for template logic.

**Model answer:**

A library chart is a special type of Helm chart (`type: library` in `Chart.yaml`) whose sole purpose is to contain reusable named templates. It cannot be installed standalone — `helm install` will fail. It produces no Kubernetes resources directly.

**Use case:** When an organization maintains many application charts that need common labels, Pod specifications, resource defaults, or naming conventions, library charts prevent duplication.

```yaml
# library-chart/Chart.yaml
apiVersion: v2
name: common
version: 1.0.0
type: library
```

```yaml
# parent-chart/Chart.yaml
dependencies:
  - name: common
    version: 1.0.0
    repository: "file://../common"
```

```yaml
# parent-chart/templates/deployment.yaml
metadata:
  labels:
    {{- include "common.labels" . | nindent 4 }}
```

**Key points to mention:**
- `type: library` in Chart.yaml.
- Cannot be installed standalone.
- Provides reusable `define`/`include` named templates.
- Prevents duplication across many application charts.
- Managed as a dependency in the parent chart.

---

### Q38. How do subcharts work? How are values passed to them?

**What the interviewer is looking for:** Understanding of the parent-child chart relationship and value scoping.

**Model answer:**

A subchart is a chart that is declared as a dependency of another (parent) chart and is deployed as part of the parent's release. Subcharts provide modularity — a web application chart can depend on PostgreSQL, Redis, and monitoring charts.

**How they work:**
- Declared in `Chart.yaml` under `dependencies`.
- Fetched into `charts/` directory by `helm dependency update`.
- When the parent is installed, subcharts are installed automatically alongside it.
- The parent release owns all subchart resources — there is one release, not multiple.

**Value scoping and passing:**

Subcharts have their own `values.yaml` and their own template scope. Values are passed from the parent in two ways:

1. **Via the subchart's key** — a top-level key in the parent's `values.yaml` that matches the subchart name:

```yaml
# parent values.yaml
postgresql:                      # key matches the subchart name
  auth:
    username: myuser
    password: mypassword
  primary:
    persistence:
      size: 20Gi
```

2. **Via `global` values** — the `global:` key in the parent is inherited by ALL subcharts:

```yaml
# parent values.yaml
global:
  imageRegistry: myregistry.io
  imagePullSecrets:
    - name: regcred
```

Every subchart can access `.Values.global.imageRegistry`.

**Precedence for subchart values:**
```
Subchart's own values.yaml        (lowest)
-> Parent's subchart-specific values (e.g., postgresql.auth.password)
-> Parent's global values           (highest)
```

**Key points to mention:**
- Subcharts deployed as part of the parent release.
- Values passed via subchart-named keys and `global:`.
- Subcharts have isolated template scope.
- List values are replaced, not merged, when overridden.
- Conditions and tags control subchart inclusion/exclusion.

---

### Q39. What are conditions and tags in dependencies?

**What the interviewer is looking for:** Understanding of conditional subchart inclusion.

**Model answer:**

Conditions and tags allow enabling or disabling subchart dependencies at install time without modifying `Chart.yaml`. Both must evaluate to `true` for the dependency to be included (AND logic).

**Conditions:**

A `condition` is a path to a boolean value in the parent's `values.yaml`.

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

```yaml
# values.yaml
postgresql:
  enabled: true    # postgresql subchart is installed
```

**Tags:**

Tags are labels assigned to dependencies. A corresponding top-level `tags:` map in values controls which groups of dependencies are enabled.

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    tags:
      - database
  - name: redis
    version: "18.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    tags:
      - cache
```

```yaml
# values.yaml
tags:
  database: true    # postgresql enabled
  cache: false      # redis disabled
```

**Logic:** For a dependency to be installed, ALL conditions must evaluate to `true` AND ALL tags must evaluate to `true`. If a condition value is not set, it defaults to `true`. If a tag is not set, the dependency is installed.

**Key points to mention:**
- Conditions: path to a boolean in values (e.g., `postgresql.enabled`).
- Tags: group-based enable/disable via `tags:` map.
- AND logic: all conditions AND all tags must be true.
- Overridable at install/upgrade time via `--set`.
- Condition unset = default `true`; tag unset = dependency included.

---

### Q40. How does Helm handle CRDs?

**What the interviewer is looking for:** Understanding of the special `crds/` directory behavior.

**Model answer:**

Helm handles CRDs (Custom Resource Definitions) placed in the `crds/` directory with special, limited behavior:

**Installation:**
- CRD files in `crds/` are installed **before** any templates are rendered and applied.
- This ensures the CRD exists in the cluster before any custom resources are created by templates.

**Upgrade and deletion constraints:**
- CRDs are **never upgraded**. If a CRD changes between chart versions, Helm does not apply the update.
- CRDs are **never deleted** — `helm uninstall` does not remove CRD resources from the cluster.
- CRDs from `crds/` are **not managed by Helm after initial install**. They have no release labels, no revision tracking.

**Why these constraints exist:**
Deleting a CRD also deletes all instances of that custom resource across the entire cluster. The conservative approach prevents accidental cascading deletions.

**Warning:** CRD templates in the `templates/` directory behave differently from those in `crds/`. Templates are fully managed (upgraded, deleted, rolled back). Use `templates/` only if you fully understand the lifecycle implications.

**Key points to mention:**
- `crds/` files installed before templates.
- Never upgraded; never deleted by `helm uninstall`.
- Not part of Helm's lifecycle management.
- Designed to prevent accidental cascading CRD deletions.
- For managed CRDs, use `templates/` instead (with caution).

---

### Q41. What are Helm plugins? Name some useful ones.

**What the interviewer is looking for:** Awareness of the extensibility mechanism and knowledge of the ecosystem.

**Model answer:**

Helm plugins are extensions that add new subcommands to the `helm` CLI. They are standalone executables or scripts that follow a specific plugin protocol. Plugins are installed via `helm plugin install` and managed with `helm plugin list`, `helm plugin update`, and `helm plugin uninstall`.

**Installation:**
```bash
helm plugin install https://github.com/databus23/helm-diff
helm plugin list
helm plugin uninstall helm-diff
```

**Useful plugins:**

| Plugin | Purpose |
|--------|---------|
| **helm-diff** | Shows a colorized diff of what will change during an upgrade (previews the three-way merge output) |
| **helm-secrets** | Integrates with SOPS for encrypting/decrypting secrets in values files |
| **helmfile** | Declarative specification for deploying Helm charts across environments |
| **helm-unittest** | Unit testing for Helm chart templates using YAML-based test definitions |
| **helm-mapkubeapis** | Map deprecated or removed Kubernetes API versions in release metadata |
| **helm-s3** | Use Amazon S3 as a Helm chart repository |

**Key points to mention:**
- Plugins extend Helm with new subcommands.
- Managed via `helm plugin install/list/update/uninstall`.
- helm-diff for previewing changes.
- helm-secrets for SOPS integration.
- helm-unittest for template unit testing.
- helm-mapkubeapis for handling deprecated API versions.

---

### Q42. How do you handle secrets properly in Helm charts?

**What the interviewer is looking for:** Security mindset — secrets should never be in plain text in values.yaml or committed to Git.

**Model answer:**

Secrets should **never** be stored in plain text in `values.yaml` or committed to Git. Multiple strategies exist:

**1. External Secrets Operator (ESO) — Recommended:**
The chart creates an `ExternalSecret` resource that references a secret provider (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault). The operator syncs the secret into a Kubernetes Secret at runtime.

**2. Sealed Secrets:**
Encrypted Kubernetes Secrets committed to Git. Only the Sealed Secrets controller in the cluster can decrypt them.

**3. SOPS + helm-secrets plugin:**
SOPS encrypts entire values files. The `helm-secrets` plugin decrypts them during `helm install`. Encrypted files can be safely committed to Git.

**4. Vault injection (sidecar/init-container):**
HashiCorp Vault injector mutates Pods to include an init-container or sidecar that fetches secrets from Vault at runtime.

**5. Kubernetes Secrets (created externally):**
The chart template references an existing Secret name via values. The Secret is created separately and never managed by Helm.

```yaml
# template referencing externally-created secret
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ .Values.database.existingSecret }}
        key: password
```

**Key points to mention:**
- Never store secrets in plain text in values.yaml or Git.
- ESO / Sealed Secrets / SOPS + helm-secrets / Vault injection.
- Chart templates should reference secret names, not contain secret values.
- Separate secret lifecycle from chart lifecycle.

---

### Q43. What is helm-diff and why is it useful?

**What the interviewer is looking for:** Operational maturity — previewing changes before applying them.

**Model answer:**

`helm-diff` is a Helm plugin that shows a colorized, line-by-line diff of what will change when you run `helm upgrade`. It renders both the old and new manifests and displays the differences in a human-readable format.

```bash
helm diff upgrade my-release ./mychart -f values-prod.yaml
helm diff upgrade my-release ./mychart --detailed-exitcode  # exit 2 if changes detected
helm diff rollback my-release 3                              # preview rollback changes
```

**Why it's useful:**
- **Pre-deployment safety:** Know exactly what will change before running `helm upgrade` in production.
- **CI/CD integration:** Use `--detailed-exitcode` to fail CI if unintended changes are detected.
- **Code review:** Run `helm diff` in a PR pipeline and include the diff output as a comment.
- **Audit:** Compare any two revisions or preview a rollback before executing it.

**Key points to mention:**
- Shows colorized diff between deployed and proposed manifests.
- `--detailed-exitcode` returns 2 if changes exist (useful for CI).
- Works for upgrade, rollback, and install.
- Essential for production safety and CI/CD gating.

---

### Q44. Explain how --reuse-values, --reset-values, and --reset-then-reuse-values differ

**What the interviewer is looking for:** Nuanced understanding of upgrade-time value handling.

**Model answer:**

These flags control how previously applied values (from the last release revision) are handled during `helm upgrade`.

**`--reuse-values`:**
Merges the **previously deployed values** with any new values provided in the upgrade command. The previously deployed values take precedence over the chart's default `values.yaml` but are overridden by the new `-f` or `--set` values from the upgrade command.

```
Chart's values.yaml (lowest)
-> Previously deployed values (from last release)
-> New -f/--set values from upgrade command (highest)
```

**`--reset-values`:**
Discards all previously deployed values. The upgrade uses only the chart's default `values.yaml` merged with the new `-f` or `--set` values from the upgrade command.

```
Chart's values.yaml (lowest)
-> New -f/--set values from upgrade command (highest)
```

**`--reset-then-reuse-values`:**
First resets to the chart's default `values.yaml`, then merges in the previously deployed values, then merges the new `-f`/`--set` values. This is useful when the chart author has removed or renamed values keys.

```
Chart's values.yaml (lowest)
-> Previously deployed values
-> New -f/--set values from upgrade command (highest)
```

**Key points to mention:**
- `--reuse-values`: keep old values, merge with new overrides.
- `--reset-values`: discard old values, use only new overrides.
- `--reset-then-reuse-values`: reset to defaults, then reapply old values, then new overrides.
- All three are superseded by explicitly passing all values via `-f` files (the recommended practice).

---

### Q45. How does Helm handle immutable fields during upgrades?

**What the interviewer is looking for:** Understanding of Kubernetes resource constraints that affect Helm operations.

**Model answer:**

Some Kubernetes resource fields are **immutable** — they cannot be changed after creation. Examples include:
- `spec.selector` in a Deployment
- `metadata.name`
- `spec.clusterIP` in a Service (once assigned)
- `spec.volumeName` in a PersistentVolumeClaim

When a `helm upgrade` attempts to change an immutable field, the Kubernetes API server rejects the update. Helm marks the release as `failed`.

**How to handle:**
- Use `helm diff upgrade` to preview changes before applying.
- Design templates to use stable selectors (e.g., `app.kubernetes.io/instance` and `app.kubernetes.io/name` only).
- For resources like PVCs, use a separate lifecycle.
- If immutable field change is necessary: `helm uninstall` + reinstall (causes downtime) or manually delete the resource before upgrade.

**Key points to mention:**
- Immutable fields cause upgrade failures at the API server level.
- Helm marks the release as `failed`.
- Use `helm diff` to preview changes before applying.
- Stable labels prevent selector changes.
- Manual intervention required for recovery.

---

### Q46. What is the purpose of --history-max?

**What the interviewer is looking for:** Understanding of release history management and storage implications.

**Model answer:**

`--history-max` sets the maximum number of revisions Helm retains for a release. The default is **10**. When a new revision is created and the limit is exceeded, the oldest revision Secret is automatically deleted.

```bash
helm install my-release ./mychart --history-max 5
helm upgrade my-release ./mychart --history-max 5
```

**Why it matters:**

1. **Storage:** Each revision is a Kubernetes Secret (1MB max each). In high-frequency CI/CD environments, 10 revisions accumulate quickly across many releases.

2. **Rollback window:** `--history-max` defines how far back you can roll back. With `--history-max 5`, you can roll back to any of the last 5 revisions.

3. **Security:** Release Secrets contain the full rendered manifest and all configuration values. Minimizing history reduces the attack surface.

**Production recommendations:**
- Set `--history-max` explicitly — don't rely on the default.
- In CI/CD: `--history-max 5` is often sufficient.
- For production rollback safety: `--history-max 10` to `--history-max 20`.
- Consider a cleanup CronJob to periodically purge very old revisions.

**Key points to mention:**
- Limits number of retained revision Secrets (default: 10).
- Old revisions are automatically deleted when the limit is exceeded.
- Trade-off: storage vs. rollback depth.
- Should always be set explicitly in production and CI/CD.

---

### Q47. Explain hook weights and delete policies

**What the interviewer is looking for:** Deep understanding of hook orchestration — ordering and cleanup.

**Model answer:**

**Hook weights** control the **execution order** of hooks that run for the same event.

- Set via annotation: `"helm.sh/hook-weight": "5"`
- Default weight: `0`
- Lower weights execute first
- Hooks with the same weight run in parallel

```yaml
# Hook 1: runs first (weight -1)
annotations:
  "helm.sh/hook": pre-install
  "helm.sh/hook-weight": "-1"

# Hook 2: runs second (weight 0, default)
annotations:
  "helm.sh/hook": pre-install

# Hook 3: runs third (weight 5)
annotations:
  "helm.sh/hook": pre-install
  "helm.sh/hook-weight": "5"
```

**Hook delete policies** control **when** hook resources are cleaned up.

| Policy | Behavior |
|--------|----------|
| `before-hook-creation` | Delete the previous hook resource before creating a new one (default for Jobs) |
| `hook-succeeded` | Delete the hook resource after it completes successfully |
| `hook-failed` | Delete the hook resource if it fails |

Multiple policies can be combined:

```yaml
annotations:
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

**Warning:** Without delete policies, hook resources accumulate in the cluster indefinitely.

**Key points to mention:**
- Weights determine execution order for same-event hooks (lower = first).
- Default weight is 0; same weight = parallel execution.
- Delete policies: `before-hook-creation`, `hook-succeeded`, `hook-failed`.
- Without delete policies, hook resources accumulate.
- Both annotations are independent — a hook can have both.

---

### Q48. What is the difference between install, upgrade, and template commands?

**What the interviewer is looking for:** Clear differentiation of the three core commands and when each is used.

**Model answer:**

| Aspect | `helm template` | `helm install` | `helm upgrade` |
|--------|----------------|----------------|----------------|
| **API Server Contact** | No | Yes | Yes |
| **Creates Kubernetes Resources** | No | Yes | Yes (mutates) |
| **Creates Release Record** | No | Yes (revision 1) | Yes (new revision) |
| **Requires Existing Release** | No | No (fails if exists) | Yes (fails unless `--install`) |
| **Executes Hooks** | No | Yes | Yes |
| **Output** | Rendered YAML to stdout | Resources in cluster | Resources in cluster |
| **Primary Use Case** | GitOps, debugging | Initial deployment | Updates to existing deployment |

**When to use each:**
- **`helm template`**: GitOps workflows; debugging template rendering; environments where Helm CLI can't reach the API server.
- **`helm install`**: First deployment of an application into a cluster/namespace.
- **`helm upgrade`**: Deploying changes to an already-running application.
- **`helm upgrade --install`**: Idempotent deployment — the standard CI/CD command.

**Key points to mention:**
- `template` is local-only, no API server, no state.
- `install` creates a new release.
- `upgrade` mutates an existing release.
- `upgrade --install` is idempotent (CI/CD standard).
- Only `install` and `upgrade` execute hooks and create revision history.

---

### Q49. How do you verify a signed Helm chart?

**What the interviewer is looking for:** Supply chain security awareness — chart provenance verification.

**Model answer:**

Helm supports chart signing using PGP (GPG) keys. The signing process creates a provenance file (`.prov`) that contains the chart's digest and metadata, signed with a private key. Verification checks the signature against a trusted public key.

**Signing a chart:**
```bash
helm package --sign --key 'my-key' --keyring ~/.gnupg/secring.gpg ./mychart
# Produces: mychart-1.0.0.tgz and mychart-1.0.0.tgz.prov
```

**Verifying a chart:**
```bash
helm verify mychart-1.0.0.tgz     # verify using default keyring
helm verify --keyring ~/.gnupg/pubring.gpg mychart-1.0.0.tgz
```

**What verification checks:**
1. The `.prov` file is a valid PGP signature.
2. The signature matches a public key in the specified keyring.
3. The chart digest in the provenance file matches the actual chart `.tgz` file.

**OCI-based signing (modern approach):**
```bash
cosign sign --key cosign.key registry-1.docker.io/myorg/mychart:1.0.0
cosign verify --key cosign.pub registry-1.docker.io/myorg/mychart:1.0.0
```

**Key points to mention:**
- Chart signing uses PGP/GPG keys.
- `.prov` file contains signed digest + metadata.
- `helm verify` checks signature against keyring and digest against actual file.
- OCI approach: cosign/notation for supply chain security.
- Verification should be part of CI/CD pipeline before deployment.

---

### Q50. How does Helm interact with Kubernetes API?

**What the interviewer is looking for:** Understanding of Helm's client-side architecture and API usage.

**Model answer:**

Helm is a **client-only** tool (since Helm 3 — no Tiller). It interacts with the Kubernetes API server directly using the user's kubeconfig.

**Interaction flow during `helm install`:**

1. **Load kubeconfig** — Helm reads the kubeconfig file (`KUBECONFIG` env var or `~/.kube/config`).
2. **Discover API resources** — queries `/api` and `/apis` to determine available resources and API versions.
3. **Render templates** — locally renders chart templates with merged values.
4. **Install CRDs** — if `crds/` directory exists, applies CRDs first via the API.
5. **Apply manifest** — sends each resource in the rendered manifest to the API server using standard Kubernetes REST API calls.
6. **Create release Secret** — stores the release metadata as a Secret in the release namespace.
7. **Execute hooks** — if hooks are defined, runs them in weight order.

**Permissions:** Helm requires the same RBAC permissions the user has. There is no Helm-specific RBAC — if you can `kubectl apply` a resource, you can `helm install` it.

**Key points to mention:**
- Client-only; no server-side component.
- Uses kubeconfig for authentication.
- Queries `/api` and `/apis` for capability discovery.
- Standard REST API calls (POST, PUT, PATCH, DELETE).
- RBAC is the user's kubeconfig permissions — no Helm-specific RBAC.
- `helm template` does not contact the API server at all.

---

### Q51. What is the post-renderer flag used for?

**What the interviewer is looking for:** Awareness of the post-processing extension point for rendered manifests.

**Model answer:**

`--post-renderer` specifies an executable (script or binary) that Helm pipes the rendered manifest through before applying it to the cluster. The post-renderer receives the rendered YAML on stdin, modifies it, and writes the modified YAML to stdout.

```bash
helm install my-release ./mychart --post-renderer ./scripts/patch-manifest.sh
```

**Common use cases:**
- **Kustomize integration:** Render with Helm, then apply Kustomize patches.
- **Add missing fields:** Inject standard labels, annotations, or security contexts.
- **Validation:** Run `kubeconform` against the rendered manifests before applying.
- **Formatting:** Strip null fields, sort keys, or clean up whitespace.
- **Policy enforcement:** Run OPA/Gatekeeper checks before applying.

**Key points to mention:**
- Pipes rendered YAML through an external executable.
- stdin = rendered manifest, stdout = modified manifest.
- Enables Kustomize + Helm composition.
- Use for validation, formatting, or policy enforcement.
- The post-renderer must output valid Kubernetes YAML.

---

### Q52. How do environment variables affect Helm behavior?

**What the interviewer is looking for:** Knowledge of Helm's environment-based configuration surface.

**Model answer:**

Helm reads several environment variables that control its behavior. These are useful for CI/CD pipelines, custom configurations, and debugging.

**Key environment variables:**

| Variable | Purpose | Default |
|----------|---------|---------|
| `HELM_CACHE_HOME` | Directory for cached charts, repo indexes | `~/.cache/helm` |
| `HELM_CONFIG_HOME` | Directory for Helm configuration | `~/.config/helm` |
| `HELM_DATA_HOME` | Directory for Helm data (plugins, starters) | `~/.local/share/helm` |
| `HELM_DRIVER` | Storage backend: `secret` or `configmap` | `secret` |
| `HELM_KUBECONTEXT` | Override the Kubernetes context | Current context |
| `HELM_KUBECONFIG` | Override the kubeconfig file path | `KUBECONFIG` env var or `~/.kube/config` |
| `HELM_MAX_HISTORY` | Default `--history-max` value | 10 |
| `HELM_NAMESPACE` | Default namespace for operations | Namespace from kubeconfig context |
| `HELM_REGISTRY_CONFIG` | Path to the registry config file | `$(HELM_CONFIG_HOME)/registry/config.json` |
| `HELM_REPOSITORY_CACHE` | Path to the repository cache directory | `$(HELM_CACHE_HOME)/repository` |
| `HELM_REPOSITORY_CONFIG` | Path to the repositories configuration file | `$(HELM_CONFIG_HOME)/repositories.yaml` |
| `KUBECONFIG` | Standard Kubernetes config file | `~/.kube/config` |

**Key points to mention:**
- Environment variables control cache, config, plugin, and registry paths.
- `HELM_DRIVER` switches between `secret` and `configmap` storage.
- `HELM_NAMESPACE` overrides the default namespace.
- `KUBECONFIG` and `HELM_KUBECONTEXT` control cluster connectivity.
- Essential for CI/CD configuration and air-gapped environments.

---

### Q53. What is values.schema.json?

**What the interviewer is looking for:** Understanding of values validation and schema enforcement.

**Model answer:**

`values.schema.json` is an optional JSON Schema file at the chart root that defines the structure, types, and constraints for the chart's `values.yaml`. Helm validates the merged values against this schema before `helm install`, `helm upgrade`, `helm lint`, and `helm template` (with `--validate`).

**Example:**
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

**When validation fails:**
```bash
$ helm install my-release ./mychart --set service.port=99999
Error: values don't meet the specifications of the schema(s) in the following chart(s):
mychart:
- service.port: Must be less than or equal to 65535
```

**Key points to mention:**
- JSON Schema at chart root; validated before install/upgrade.
- Defines types, required fields, enum values, numeric ranges.
- Provides early validation and serves as documentation.
- `helm lint` checks schema, `helm install` enforces it.
- Supports JSON Schema draft-07.

---

### Q54. How do you debug a failed Helm release?

**What the interviewer is looking for:** Systematic troubleshooting methodology.

**Model answer:**

When a Helm release is in a `failed` state, follow this systematic approach:

**1. Check release status:**
```bash
helm status my-release --show-resources
helm history my-release
```

**2. View the rendered manifest:**
```bash
helm get manifest my-release
```

**3. View the merged values:**
```bash
helm get values my-release --all
```

**4. Inspect the release Secret:**
```bash
kubectl get secret sh.helm.release.v1.my-release.v<N> -n <namespace> \
  -o jsonpath='{.data.release}' | base64 -d | gunzip | jq .info
```

**5. Check Kubernetes resources:**
```bash
kubectl get all -n <namespace> -l app.kubernetes.io/instance=my-release
kubectl describe deployment <name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

**6. Debug with `helm template` and `--dry-run`:**
```bash
helm template my-release ./mychart -f values-prod.yaml --debug
helm install my-release ./mychart -f values-prod.yaml --dry-run --debug
```

**Common causes:**
- Image pull errors (wrong registry, wrong tag, missing credentials).
- Resource quota exhaustion.
- Scheduling failures (affinity, taints, node resources).
- CrashLoopBackOff (application errors).
- Immutable field changes.
- Hook failure.

**Key points to mention:**
- Systematic: status -> manifest -> values -> Kubernetes resources -> events -> logs.
- `helm get manifest` and `helm get values --all` are the starting point.
- Release Secret contains full error description.
- Check Kubernetes-level issues (events, describe, logs).
- `--dry-run --debug` for pre-deployment validation.

---

### Q55. What is the difference between helm get manifest and helm template?

**What the interviewer is looking for:** Understanding of the distinction between stored state and live rendering.

**Model answer:**

`helm get manifest` retrieves the **actual manifest that was applied** to the cluster for an installed release. It reads from the release Secret stored in Kubernetes.

`helm template` renders the chart **locally** with given values — it does not contact the Kubernetes API server and does not read release history.

| Aspect | `helm get manifest` | `helm template` |
|--------|--------------------|--------------------|
| **Source** | Release Secret in cluster (stored state) | Local chart directory (live rendering) |
| **API Server** | Yes (reads Secrets) | No |
| **Requires Release** | Yes (must be installed) | No |
| **Values Used** | Values from when the release was deployed | Values you provide (`-f`, `--set`) or defaults |
| **Use Case** | Inspect what was actually deployed | Preview what would be deployed |

**Key points to mention:**
- `get manifest` = what WAS deployed (reads release Secret).
- `template` = what WOULD be deployed (local rendering).
- `get manifest` requires an installed release; `template` does not.
- `get manifest --revision <N>` shows any historical revision's manifest.

---

### Q56. How are release revisions numbered?

**What the interviewer is looking for:** Understanding of the monotonically increasing revision scheme.

**Model answer:**

Release revisions are **monotonically increasing integers** starting at 1. Every `helm install`, `helm upgrade`, and `helm rollback` creates a new revision with the next number in sequence.

| Action | Revision Number |
|--------|----------------|
| `helm install` | 1 |
| `helm upgrade` | Previous + 1 |
| `helm rollback to 2` | Previous + 1 (NOT 2 — a new revision is created) |

**Critical nuance about rollback:**
```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         deployed    Upgrade complete

$ helm rollback my-release 2

4         deployed    Rollback to 2
```

Revision 4 has the **same manifest** as revision 2, but it is a **new, distinct revision**. Revision numbers are never reused, never decrease. Revision numbers reflect deployment events, not semantic versions.

**Key points to mention:**
- Monotonically increasing, starting at 1.
- Never reused, never decreased.
- Rollback creates a new revision (it does not "go back" in the sequence).
- Revision numbers reflect deployment events, not semantic versions.

---

### Q57. How does Helm handle dependencies during rollback?

**What the interviewer is looking for:** Understanding of holistic rollback — the entire release, not just the parent chart.

**Model answer:**

When you roll back to a previous revision, Helm rolls back the **entire release**, including all subchart dependencies. It uses the dependency state that was recorded at that revision.

**How it works:**
1. Helm retrieves the release Secret for the target revision.
2. It extracts the stored manifest (which includes all resources from the parent chart and all subcharts at that revision).
3. It applies the three-way strategic merge against the old manifest, the target revision's manifest, and the current live state.
4. The entire application stack — parent chart + all subcharts — returns to the state defined in that revision.

**Warning:** Rolling back database subcharts downgrades the Kubernetes resources but does NOT downgrade the actual data. Schema migrations are not undone by Helm rollback.

**Key points to mention:**
- Rollback affects the entire release, including all subcharts.
- Dependency versions from the target revision are used.
- Data in databases/PVCs is not rolled back — only Kubernetes resources.
- Schema migrations must be handled separately from Helm rollbacks.

---

### Q58. What happens if you run helm install on an existing release?

**What the interviewer is looking for:** Understanding of Helm's idempotency guard.

**Model answer:**

`helm install` fails with an error if a release with the same name already exists in the namespace:

```
Error: INSTALLATION FAILED: cannot re-use a name that is still in use
```

This is a safety mechanism to prevent accidentally overwriting an existing deployment.

**How Helm detects existing releases:**
Helm checks for the existence of release Secrets (`sh.helm.release.v1.<name>.v*`) in the target namespace. If any are found with the same name, `helm install` refuses to proceed.

**What to use instead:**
- `helm upgrade --install` — upgrade if the release exists, install if it doesn't (idempotent).
- `helm uninstall my-release` followed by `helm install` (destructive).
- Use a different release name.

**Best practice for CI/CD:**
Always use `helm upgrade --install` instead of `helm install`. This makes deployments idempotent.

**Key points to mention:**
- `helm install` fails if the release name is already in use.
- Detected via existing release Secrets in the namespace.
- Use `helm upgrade --install` for idempotent deployments.
- Use different release name if a separate release is genuinely needed.

---

### Q59. Explain the chart lifecycle from development to production

**What the interviewer is looking for:** Understanding of the full chart development-to-deployment pipeline.

**Model answer:**

A Helm chart goes through a structured lifecycle:

**1. Development (local workstation):**
- `helm create mychart` — scaffold a new chart.
- Edit templates, `values.yaml`, `Chart.yaml`.
- `helm lint ./mychart` — static validation.
- `helm template my-release ./mychart --debug` — verify rendering.

**2. Testing (development cluster):**
- `helm install test-release ./mychart --namespace dev` — deploy to dev cluster.
- `helm test test-release` — run integration tests.

**3. Packaging:**
- Update `version` in `Chart.yaml` (SemVer bump).
- `helm dependency update` — fetch latest compatible dependencies.
- `helm package ./mychart` — create `.tgz` archive.

**4. Publishing:**
- Traditional repo: Push `.tgz` to a chart repository, update `index.yaml`.
- OCI: `helm push mychart-1.0.0.tgz oci://registry.example.com/charts/`.
- Git: Commit the chart source to a Git repository for GitOps workflows.

**5. CI/CD Integration:**
- Chart linting and template validation in CI pipeline.
- Push to chart repository on merge to main.
- Deployment to staging via `helm upgrade --install --atomic --wait`.

**6. Promotion (staging -> production):**
- Same chart version, different values files.
- `helm diff upgrade` to preview production changes.
- `helm upgrade --install --atomic --wait --history-max 10` for production.
- Smoke tests against production after deployment.

**7. Deprecation:**
- Set `deprecated: true` in `Chart.yaml`.
- Provide a sunset period before removal.

**Key points to mention:**
- Chart version in Git should match `version` in `Chart.yaml`.
- Same chart version promoted across environments (different values files).
- `helm diff upgrade` before production deployment.
- `--atomic` for automatic rollback on failure.
- Don't rebuild the chart per environment — promote the same `.tgz`.

---

### Q60. How do you manage chart repositories in an air-gapped environment?

**What the interviewer is looking for:** Operational knowledge of disconnected/offline deployments.

**Model answer:**

In air-gapped (disconnected) environments, the Helm CLI cannot access public repositories. Charts must be available locally or through an internal repository.

**Strategy 1: Internal OCI Registry (Recommended):**
- Mirror required charts to an internal OCI registry (Harbor, Artifactory, internal Docker Registry).
- Push charts to the internal registry from a connected "bastion" host:
  ```bash
  helm pull oci://registry-1.docker.io/bitnamicharts/nginx --version 18.2.0
  helm push nginx-18.2.0.tgz oci://internal-registry.local/charts/
  ```
- Air-gapped cluster pulls from the internal registry.

**Strategy 2: Local Chart Directory:**
- Download and extract charts into a shared filesystem or Git repository.
- Reference charts by local path:
  ```bash
  helm install my-release /path/to/nginx/
  helm install my-release ./charts/nginx-18.2.0.tgz
  ```

**Strategy 3: Pre-cached Dependencies:**
- For charts with dependencies, run `helm dependency update` in a connected environment.
- Commit the `charts/` directory (with `.tgz` files) to Git.
- In the air-gapped environment, install directly from Git.

**Key considerations:**
- Container images must also be mirrored — Helm only manages manifests.
- Use `helm repo index` to generate `index.yaml` for a directory of `.tgz` files.
- Pre-cache everything in a connected bastion, then transfer.

**Key points to mention:**
- Mirror charts to internal OCI registry or HTTP repository.
- Commit dependencies (`charts/` directory) to Git.
- Container images must also be mirrored — Helm only manages manifests.
- Use `helm repo index` for building custom internal repos.

---

### Q61. What is the difference between `helm repo add` and `helm registry login`?

**What the interviewer is looking for:** Clear understanding of the two distribution mechanisms' authentication models.

**Model answer:**

`helm repo add` registers a **traditional HTTP Helm repository** locally. It stores the repository name and URL — authentication is typically basic auth.

```bash
helm repo add myrepo https://charts.example.com --username user --password pass
```

`helm registry login` authenticates with an **OCI-compliant container registry**. It stores credentials in the local registry config file, similar to `docker login`.

```bash
helm registry login registry-1.docker.io -u myuser -p mypass
```

| Aspect | `helm repo add` | `helm registry login` |
|--------|----------------|----------------------|
| **Registry type** | Traditional HTTP Helm repository | OCI container registry |
| **Authentication** | Per-repository basic auth | OCI-compliant token/credential auth |
| **Credential storage** | In `repositories.yaml` | In `registry/config.json` |
| **Subsequent commands** | `helm search repo myrepo/...` | `helm pull oci://...` |
| **URL prefix** | `https://` or `http://` | `oci://` |

**Key points to mention:**
- `repo add` = traditional HTTP Helm repos.
- `registry login` = OCI container registries.
- Different credential storage and authentication models.
- `oci://` URL prefix distinguishes OCI from traditional.
- `helm registry login` is analogous to `docker login`.

---

### Q62. How does Helm handle namespace creation?

**What the interviewer is looking for:** Understanding of namespace lifecycle management in Helm.

**Model answer:**

Helm does **not** automatically create namespaces by default. If the target namespace doesn't exist, `helm install` fails:

```
Error: create: failed to create: namespaces "prod" not found
```

**Creating namespaces with Helm:**

1. **`--create-namespace` flag:**
```bash
helm install my-release ./mychart --namespace prod --create-namespace
```
This creates the namespace if it doesn't exist as part of the install operation.

2. **Template-based namespace creation:**
You can include a Namespace resource in your templates:
```yaml
# templates/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Release.Namespace }}
```
However, using `--create-namespace` is generally preferred.

**Important behavior:**
- `helm uninstall` does NOT delete the namespace (even if Helm created it with `--create-namespace`).
- `helm list` defaults to the current kubeconfig context namespace — use `--all-namespaces` or `-A` to see all releases.

**Key points to mention:**
- No automatic namespace creation (without `--create-namespace`).
- `--create-namespace` creates the namespace as part of install.
- `helm uninstall` never deletes the namespace.
- Namespace is part of the release identity and scope.

---

### Q63. What is the `lookup` function in templates and when should you use it?

**What the interviewer is looking for:** Understanding of the Kubernetes API query function and its appropriate use cases.

**Model answer:**

The `lookup` function queries the live Kubernetes API server from within a template and returns the result. It allows a template to read existing cluster state and use it to dynamically generate configuration.

**Syntax:**
```yaml
{{- $secret := lookup "v1" "Secret" .Release.Namespace "my-secret" -}}
apiVersion: v1
kind: Secret
data:
  copied-password: {{ $secret.data.password }}
```

**When to use:**
- Checking if a CRD is installed before creating custom resources.
- Discovering pre-existing Secrets or ConfigMaps.
- Determining cluster-specific values (storage classes, ingress controllers).

**When NOT to use:**
- **Generating passwords/secrets:** creates a dependency on cluster state that breaks declarative principles.
- **`helm template` compatibility:** `lookup` requires a live API server; it fails with `helm template`.
- **Deterministic output:** `lookup` makes template output non-deterministic.

**Best practice:** Minimize `lookup` usage. Prefer passing all required information via values. Use `lookup` only for truly dynamic, cluster-specific data.

**Key points to mention:**
- Queries live Kubernetes API from within templates.
- Returns the resource object or nil if not found.
- Breaks `helm template` (needs live API server).
- Use sparingly — prefer passing data via values.
- Common use: checking if a CRD exists, discovering cluster properties.

---

### Q64. How do you handle multi-environment deployments with Helm?

**What the interviewer is looking for:** Practical patterns for dev/staging/prod deployment management.

**Model answer:**

Multi-environment deployment with Helm is handled through **environment-specific values files** combined with a **single, versioned chart**.

**Pattern:**

```
mychart/
├── Chart.yaml
├── values.yaml           # Common defaults + dev-safe settings
├── templates/
│   └── ...
└── env/
    ├── values-dev.yaml
    ├── values-staging.yaml
    └── values-prod.yaml
```

**Deployment commands:**
```bash
# Dev
helm upgrade --install myapp-dev ./mychart -f env/values-dev.yaml -n dev --create-namespace

# Staging
helm upgrade --install myapp-staging ./mychart -f env/values-staging.yaml -n staging

# Production
helm upgrade --install myapp-prod ./mychart -f env/values-prod.yaml -n prod
```

**Best practices:**
1. **Single chart per application** — promote the same chart version across environments.
2. **Separate values files per environment** — eliminate copy-paste and environment drift.
3. **Release naming** — encode the environment in the release name: `myapp-prod`, `myapp-staging`, `myapp-dev`.
4. **Namespace per environment** — `prod`, `staging`, `dev` namespaces (or separate clusters).
5. **Common values in `values.yaml`** — shared settings across all environments.

**Advanced: Umbrella chart pattern:**
An umbrella chart defines dependencies per environment, allowing different subchart combinations per environment.

**Key points to mention:**
- Single chart version promoted across environments.
- Environment-specific values files, not environment-specific charts.
- Release name encodes environment.
- Namespace per environment.
- Umbrella chart for complex multi-service deployments.

---

### Q65. What is the difference between `tpl` and `include`?

**What the interviewer is looking for:** Understanding of template evaluation vs template inclusion.

**Model answer:**

`include` inserts a named template's rendered output, optionally piping it through further functions. `tpl` evaluates a **string** as a template, processing any Go template directives within it.

**`include` — Named template inclusion:**
```yaml
{{- define "mychart.labels" -}}
app: {{ .Chart.Name }}
release: {{ .Release.Name }}
{{- end -}}

metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```

**`tpl` — String template evaluation:**
```yaml
# values.yaml
podAnnotations:
  prometheus.io/scrape: "true"
  custom: "{{ .Release.Name }}-custom-value"

# template
metadata:
  annotations:
    {{- range $key, $val := .Values.podAnnotations }}
    {{ $key }}: {{ tpl $val $ }}
    {{- end }}
```

`tpl` evaluates a string value as a Go template. This allows values to contain template expressions that reference built-in objects.

**Key difference:**
- `include` renders a named template **defined in a `.tpl` file** using `{{ define }}`.
- `tpl` renders a **string** (usually from `.Values`) as if it were a template.

**Key points to mention:**
- `include "templateName" .` — renders a defined named template.
- `tpl "string" .` — evaluates a string as a template.
- `include` is for code reuse within templates.
- `tpl` is for evaluating template expressions in values.
- Both support pipelines.

---

### Q66. How do `required` and `default` functions work in templates?

**What the interviewer is looking for:** Understanding of template-level validation and fallback mechanisms.

**Model answer:**

**`default` — Provides a fallback value:**

```yaml
{{ .Values.image.tag | default "latest" }}
{{ .Values.image.tag | default .Chart.AppVersion }}
```

If the piped value is empty (zero, nil, empty string, empty map/slice), `default` returns the specified fallback value. Common patterns:
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
replicas: {{ .Values.replicaCount | default 1 }}
```

**`required` — Enforces mandatory values:**

```yaml
{{ required "A valid .Values.service.port is required!" .Values.service.port }}
{{ required "ingress.host must be set when ingress is enabled" .Values.ingress.host }}
```

If the piped value is empty, `required` immediately fails template rendering with the provided error message.

Conditional usage:
```yaml
{{- if .Values.ingress.enabled }}
host: {{ required "ingress.host is required when ingress.enabled=true" .Values.ingress.host }}
{{- end }}
```

**When to use each:**
- `default` — for settings with safe, sensible defaults (replicaCount=1, pullPolicy=IfNotPresent).
- `required` — for settings that have no safe default and must be explicitly set (database passwords, ingress hosts, TLS certificates).

**Key points to mention:**
- `default` provides a fallback when the value is empty/nil/zero.
- `required` fails template rendering if the value is empty.
- `required` takes an error message as its first argument.
- Use `default` for safe defaults; `required` for mandatory configuration.

---

### Q67. What is Helm's behavior with StatefulSet upgrades?

**What the interviewer is looking for:** Understanding of stateful workload upgrade constraints and Helm's role.

**Model answer:**

Helm treats StatefulSet upgrades similarly to Deployment upgrades — it computes the three-way strategic merge and applies the changes. However, the Kubernetes StatefulSet controller has different upgrade semantics.

**Helm's role:**
- Updates the StatefulSet spec (image, env, resource limits).
- Creates a new release revision storing the updated manifest.

**Kubernetes StatefulSet controller behavior:**
1. **`RollingUpdate` strategy:** Pods are updated one at a time, in reverse ordinal order (pod-N -> pod-N-1 -> ... -> pod-0). Each Pod is terminated and recreated with the new spec.
2. **`OnDelete` strategy:** Existing Pods are NOT updated automatically. Only Pods that are manually deleted are recreated with the new spec.
3. **PVC immutability:** PVCs created by `volumeClaimTemplates` are NOT modified or deleted by StatefulSet updates.

**Helm-specific considerations:**
- Use `--wait` to ensure Helm monitors the StatefulSet rollout to completion.
- Set `--timeout` to a higher value for slower rolling updates.
- Schema migrations and data format changes must be handled separately.

**Key points to mention:**
- Helm computes the diff and applies it; the StatefulSet controller manages Pod updates.
- `RollingUpdate` vs `OnDelete` strategies control Pod replacement behavior.
- PVCs from `volumeClaimTemplates` are never modified by upgrades.
- Use `--wait` and extended `--timeout` for stateful workloads.
- Data/schema migrations are outside Helm's scope.

---

### Q68. How do you upgrade only a subchart?

**What the interviewer is looking for:** Understanding of the holistic release model and its constraints.

**Model answer:**

You **cannot** upgrade a subchart independently. In Helm's release model, a subchart is not an independent release — it is part of the parent release. Upgrading the parent release upgrades everything, including all subcharts.

**Why:**
- There is one release Secret, one revision history, and one manifest for the entire parent + subcharts group.
- Subcharts do not have their own release name or revision tracking.

**If you need to update a subchart:**

1. **Update the dependency in `Chart.yaml`:**
```yaml
dependencies:
  - name: postgresql
    version: "14.0.0"   # was 12.1.0
    repository: "https://charts.bitnami.com/bitnami"
```

2. **Update dependencies:**
```bash
helm dependency update ./mychart
```

3. **Upgrade the parent release:**
```bash
helm upgrade my-release ./mychart -f values-prod.yaml -n prod
```

**Design alternative:** If you need independent lifecycle management for components, deploy them as separate Helm releases (not subcharts). Use Helmfile or Argo CD ApplicationSet to manage multiple independent releases.

**Key points to mention:**
- Subcharts are not independent releases — they're part of the parent release.
- Upgrading the parent upgrades everything.
- To change a subchart version, update the dependency and upgrade the parent.
- For independent lifecycles, use separate releases, not subcharts.

---

### Q69. What does `--skip-crds` do?

**What the interviewer is looking for:** Understanding of the CRD lifecycle control flag.

**Model answer:**

`--skip-crds` prevents Helm from installing CRD files from the `crds/` directory during `helm install` or `helm upgrade`. The chart's CRDs are ignored entirely for that operation.

```bash
helm install my-release ./mychart --skip-crds
helm upgrade my-release ./mychart --skip-crds
```

**When to use:**
1. **CRDs already installed:** The CRDs are already present in the cluster.
2. **CRDs managed by a different tool:** CRDs are managed by an operator's own lifecycle (OLM, crossplane, etc.).
3. **CRD permissions:** The Helm user lacks cluster-scoped permissions to create CRDs. `--skip-crds` allows installing only the namespaced resources.
4. **GitOps workflows:** CRDs are managed by a separate, privileged reconciliation loop.

**Important reminder:** CRDs in the `crds/` directory are never upgraded or deleted by Helm anyway — even without `--skip-crds`. The flag primarily controls the initial installation.

**Key points to mention:**
- Prevents CRD installation during `helm install/upgrade`.
- Useful when CRDs are pre-installed or managed by another tool.
- CRDs require cluster-scoped permissions — `--skip-crds` helps with restricted RBAC.
- CRDs are never upgraded/deleted by Helm even without this flag.

---

### Q70. How do you clean up old release revisions?

**What the interviewer is looking for:** Operational hygiene — preventing Secret accumulation.

**Model answer:**

Old release revision Secrets accumulate over time, especially in CI/CD environments with frequent deployments.

**Methods:**

1. **Set `--history-max` (preventive):**
```bash
helm install my-release ./mychart --history-max 5
helm upgrade my-release ./mychart --history-max 5
```

2. **Clean up after the fact:**
```bash
# Delete all superseded revisions for a release (keep latest 5)
kubectl get secrets -n prod -l owner=helm -l name=my-release -l status=superseded \
  --sort-by=.metadata.labels.version \
  -o name | head -n -5 | xargs -r kubectl delete -n prod
```

3. **CronJob for automated cleanup:**
Create a CronJob that periodically deletes superseded release Secrets across all namespaces, keeping the last N revisions.

**Warning:** Do not delete the currently `deployed` revision Secret — Helm will lose track of the release. Only delete `superseded` revisions.

**Key points to mention:**
- `--history-max` prevents accumulation at the source.
- Delete only `superseded` revision Secrets — never `deployed`.
- CronJob for periodic cleanup in high-churn environments.
- Manual deletion via `kubectl delete secret` with label selectors.


---

## HARD — Architecture, Scenarios, & Production (30 Questions)

---

### Q71. Design a Helm-based deployment strategy for a microservices platform with 50+ services

**What the interviewer is looking for:** Systems design thinking — chart organization, repository strategy, CI/CD, promotion, and operational concerns at scale.

**Model answer:**

**1. Chart organization — Two approaches:**

**Option A: One chart per service (recommended for independent teams):**
```
charts/
├── user-service/
├── order-service/
├── payment-service/
├── notification-service/
├── api-gateway/
└── ... (50+ service charts)
```
Each service team owns its chart. Charts share a library chart (`common`) for standard labels, probes, security contexts.
- Pros: Independent versioning, independent deployments, clear ownership.
- Cons: Chart duplication risk (mitigated by library chart).

**Option B: Umbrella chart with subcharts (recommended for platform team):**
```yaml
# umbrella/Chart.yaml
dependencies:
  - name: user-service
    version: "2.1.0"
    repository: "oci://registry.example.com/charts/"
    condition: user-service.enabled
  - name: order-service
    version: "1.5.0"
    repository: "oci://registry.example.com/charts/"
    condition: order-service.enabled
```
Single `helm install` deploys the entire platform. Individual services toggled via conditions.
- Pros: Single deployment command, holistic versioning.
- Cons: Tight coupling — upgrading one service upgrades all.

**2. Repository strategy:**
- OCI registry per environment (dev, staging, prod registries).
- Artifact promotion: Push to dev registry -> promote (copy) to staging -> promote to prod. Same digest, different registry.
- Chart versioning: SemVer; version numbers in `Chart.yaml` must be unique across all services.

**3. Values management:**
```
values/
├── dev/
│   ├── user-service.yaml
│   └── ...
├── staging/
└── prod/
```
Values files per environment, per service, committed to Git in a separate `platform-config` repository.

**4. CI/CD pipeline per service:**
```
PR -> lint + test -> build image -> push image -> package chart -> push chart ->
deploy to dev -> smoke test -> [manual gate] -> promote to staging ->
integration test -> [manual gate] -> promote to prod -> canary -> full rollout
```

**5. Deployment orchestration:**
- Helmfile or Argo CD ApplicationSet for managing 50+ releases.
- Declarative release definitions; `helmfile apply` or Argo CD sync to reconcile.
- Namespace per service or per domain (e.g., `users`, `orders`, `payments`).

**6. Rollback strategy:**
- `--history-max 10` on all releases.
- `--atomic` for automatic rollback on upgrade failure.
- Canary deployments for production changes.

**7. Observability:**
- Standard labels on all resources (`app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`).
- Helm release status monitoring (Prometheus + Grafana dashboards).

**Key points to mention:**
- Chart per service vs umbrella chart trade-offs.
- Library chart for shared template logic.
- OCI registry promotion (same digest across environments).
- Git-based values management (separate config repo).
- Helmfile or Argo CD for multi-release orchestration.
- Canary releases for production safety.

---

### Q72. How would you implement a GitOps workflow with Helm and ArgoCD?

**What the interviewer is looking for:** Understanding of Helm's role in a GitOps pipeline and ArgoCD's Helm integration.

**Model answer:**

**Architecture:**
```
Developer pushes -> Git (chart + values) -> ArgoCD detects drift ->
ArgoCD renders/installs Helm chart -> Reconciles cluster state ->
Cluster matches desired state
```

**Implementation steps:**

1. **Repository structure:**
```
gitops-repo/
├── charts/
│   └── myapp/             # Helm chart source
├── environments/
│   ├── dev/
│   │   └── values.yaml
│   ├── staging/
│   │   └── values.yaml
│   └── prod/
│       └── values.yaml
└── apps/
    ├── dev-myapp.yaml     # ArgoCD Application manifest
    ├── staging-myapp.yaml
    └── prod-myapp.yaml
```

2. **ArgoCD Application definition:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prod-myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: main
    path: charts/myapp
    helm:
      valueFiles:
        - ../../environments/prod/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

3. **Helm integration modes in ArgoCD:**
- **Source-based (recommended):** ArgoCD renders the Helm chart from the source in Git, applying values files. No `helm install` CLI — ArgoCD uses the Helm SDK to render templates and applies them.
- **OCI/Repository reference:** ArgoCD pulls charts from OCI registries and applies values from Git.
- **Pre-rendered manifests:** Use `helm template` in CI to produce raw YAML committed to a separate repo.

4. **Promotion workflow:**
- Dev -> auto-sync on push to `main`.
- Staging -> auto-sync on Git tag creation.
- Prod -> manual sync (or auto-sync with required approvers via ArgoCD's RBAC).

5. **Secret handling:**
- Sealed Secrets or External Secrets Operator (ESO) in Git.
- ArgoCD deploys SealedSecrets or ExternalSecrets; the controller decrypts/fetches actual secrets at runtime.

**Key points to mention:**
- Git is source of truth; ArgoCD reconciles cluster to Git.
- Application CR declares source (chart + values) and destination (cluster + namespace).
- Auto-sync with prune and self-heal for continuous reconciliation.
- Helm source mode — ArgoCD renders templates, applies them.
- Promotion via Git branches/tags/environments.
- Secrets via Sealed Secrets or ESO — never in plain text in Git.

---

### Q73. Explain how you would handle database migrations in a Helm deployment

**What the interviewer is looking for:** Understanding of the intersection between stateless infrastructure (Helm) and stateful data (databases).

**Model answer:**

Database migrations are **not** handled by Helm directly. Helm manages Kubernetes resources; it does not execute SQL or modify data. Migrations must be run through a separate mechanism integrated with the Helm lifecycle.

**Approach 1: Helm Hook (Kubernetes Job for migrations) — Most common:**

```yaml
# templates/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"            # Run before application Pods start
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: "{{ .Values.migrations.image }}:{{ .Values.migrations.tag }}"
          command: ["./migrate", "up"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.database.existingSecret }}
                  key: url
```

The migration Job runs before the application is deployed (weight `-5`). Must succeed before the release proceeds.

**Approach 2: Init Container:**
```yaml
spec:
  initContainers:
    - name: db-migrate
      image: "migrations:{{ .Values.migrations.tag }}"
      command: ["./migrate", "up"]
```
Runs before the main container starts. Simpler but runs on every Pod restart.

**Approach 3: Separate release (decoupled lifecycle):**
Deploy migrations as a separate Helm release that must succeed before deploying the application:
```bash
helm upgrade --install myapp-migrations ./migration-chart --atomic --wait
helm upgrade --install myapp ./myapp-chart --atomic --wait
```

**Rollback considerations:**
- Helm rollback does NOT reverse database migrations. If you roll back an application, the database schema remains at the newer version.
- You need **down migrations** that can be triggered manually or via a rollback hook.
- Version your migration scripts and track which version has been applied.

**Key points to mention:**
- Helm Jobs with `pre-upgrade`/`pre-install` hooks for migrations.
- Hook weights ensure migrations run before the application starts.
- Helm rollback does NOT reverse database migrations.
- Init containers as an alternative.
- Separate release for complex migration scenarios.
- Always have tested down migrations.

---

### Q74. How do you implement canary deployments with Helm?

**What the interviewer is looking for:** Understanding of progressive delivery patterns with Helm.

**Model answer:**

Canary deployments route a small percentage of traffic to a new version while the old version continues serving the majority. Helm can be used to create the Kubernetes resources that enable canary deployments.

**Approach 1: Two Deployments + Service selector manipulation (native Kubernetes):**

```yaml
# Deploy the stable version
helm install myapp-stable ./mychart -f values-stable.yaml

# Deploy the canary version (different release name, same labels)
helm install myapp-canary ./mychart -f values-canary.yaml \
  --set canary.enabled=true \
  --set replicaCount=1

# The Service selects both stable and canary Pods via a shared label
# Ratio controlled by adjusting replica counts:
# Stable: 9 replicas, Canary: 1 replica -> 10% canary traffic
```

**Approach 2: Service Mesh (Istio/Linkerd):**

```yaml
# templates/virtual-service.yaml
{{- if .Values.canary.enabled }}
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  hosts:
    - {{ .Values.service.host }}
  http:
    - route:
        - destination:
            host: {{ include "mychart.fullname" . }}-stable
          weight: {{ sub 100 .Values.canary.weight }}
        - destination:
            host: {{ include "mychart.fullname" . }}-canary
          weight: {{ .Values.canary.weight }}
{{- end }}
```

**Approach 3: Argo Rollouts + Helm:**

Replace Deployment with Argo Rollout in the Helm chart:
```yaml
# templates/rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```

**Key points to mention:**
- Helm creates resources; Istio/Argo Rollouts manages traffic shifting.
- Two-release pattern: stable + canary releases with shared labels.
- Service mesh (Istio/Linkerd) for sophisticated traffic management.
- Argo Rollouts replaces Deployments for native canary functionality.
- Helm alone can do simple replica-count-based canaries.

---

### Q75. How do you implement blue/green deployments with Helm?

**What the interviewer is looking for:** Understanding of zero-downtime deployment patterns with Helm.

**Model answer:**

Blue/green deployment maintains two complete environments (blue = current, green = new). Traffic is switched from blue to green after the green environment is fully validated.

**Approach 1: Two Helm releases with Service switch:**

```bash
# Blue is live (initial)
helm install myapp-blue ./mychart -f values.yaml --set color=blue
# Service points to blue

# Deploy green (not yet receiving traffic)
helm install myapp-green ./mychart -f values.yaml --set color=green

# Validate green (direct access via green-specific Service or port-forward)
kubectl port-forward service/myapp-green 8080:80

# Switch traffic from blue to green by updating the Service selector
kubectl patch service myapp -p '{"spec":{"selector":{"deploy":"green"}}}'

# Keep blue for rollback; delete after validation period
helm uninstall myapp-blue
```

**Approach 2: Single release with TrafficSplit/Service configuration:**

```yaml
# values.yaml
deployment:
  color: blue
  green:
    enabled: false
```

```bash
# Deploy blue
helm install myapp ./mychart --set deployment.color=blue

# Upgrade to deploy green alongside blue
helm upgrade myapp ./mychart --set deployment.color=blue --set deployment.green.enabled=true

# After validation, switch traffic to green
helm upgrade myapp ./mychart --set deployment.color=green --set deployment.green.enabled=false
```

**Approach 3: Istio VirtualService for zero-downtime switching:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  http:
    - route:
        - destination:
            host: myapp
            subset: {{ .Values.deployment.activeColor }}  # blue or green
      weight: 100
```

Update `.Values.deployment.activeColor` and run `helm upgrade` to switch traffic instantly.

**Key points to mention:**
- Two complete environments (blue/green) created via separate or single Helm releases.
- Traffic switching via Service selector, Kubernetes Ingress, or service mesh.
- Green environment validated before traffic switch.
- Blue kept for immediate rollback.
- Resource cost: double the resources during the transition period.
- Cleanup: uninstall the old environment after validation.

---

### Q76. What is Helm's 3-way strategic merge and how does it differ from kubectl apply?

**What the interviewer is looking for:** Deep understanding of the patch computation algorithm and its advantages.

**Model answer:**

**Helm's 3-way strategic merge** computes changes by comparing three data sources:

1. **Old manifest** — the rendered manifest from the previous release revision (stored in the release Secret).
2. **New manifest** — the freshly rendered manifest from the current chart + values.
3. **Live state** — the actual state of each resource in the cluster (fetched from the API server).

**How it works:**

```
For each resource:
  if resource in new AND in old:
    PATCH with three-way merge:
      - Fields IN new but NOT IN old -> ADD (intended addition)
      - Fields IN old but NOT IN new -> DELETE (intended removal)
      - Fields IN live but NOT IN new AND NOT IN old -> PRESERVE (external mutation)
  elif resource in new but NOT in old:
    CREATE the resource
  elif resource in old but NOT in new:
    DELETE the resource
```

**kubectl apply (two-way merge)** compares only:
1. **New manifest** — the local file being applied.
2. **Live state** — the actual resource in the cluster.
3. **`last-applied-configuration` annotation** — the previous applied spec, stored as an annotation.

**Key advantage of Helm's approach:**
- Helm doesn't rely on annotations on resources (no `last-applied-configuration` annotation needed).
- Helm can distinguish between intentional removals and external mutations by comparing against the old manifest.
- This is especially useful for resources that don't support the `last-applied-configuration` annotation or when Kubernetes controllers modify resources.

**Key points to mention:**
- Three sources: old manifest, new manifest, live state (vs two sources for kubectl).
- No reliance on `last-applied-configuration` annotation.
- Intentional removals detected by old-vs-new comparison.
- External mutations preserved because they're in live but not in old or new.
- Enables clean rollback to any previous revision.

---

### Q77. How would you design a chart that can deploy to multiple cloud providers?

**What the interviewer is looking for:** Design for portability — abstracting cloud-specific differences.

**Model answer:**

A multi-cloud chart abstracts cloud-specific configuration into values, using conditional templates to adapt to each provider.

**Design principles:**

1. **Provider abstraction via values:**
```yaml
# values.yaml
cloud:
  provider: aws    # aws | gcp | azure
  region: us-east-1

storageClass:
  name: ""         # cloud-specific; set per provider env file

ingress:
  annotations: {}  # provider-specific annotations
```

2. **Environment-specific values files:**
```yaml
# env/values-aws.yaml
cloud:
  provider: aws
storageClass:
  name: gp3
ingress:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing

# env/values-gcp.yaml
cloud:
  provider: gcp
storageClass:
  name: premium-rwo
ingress:
  annotations:
    kubernetes.io/ingress.class: gce
```

3. **Conditional templates:**
```yaml
{{- if eq .Values.cloud.provider "aws" }}
apiVersion: v1
kind: StorageClass
metadata:
  name: {{ .Values.storageClass.name }}
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
{{- else if eq .Values.cloud.provider "gcp" }}
apiVersion: v1
kind: StorageClass
metadata:
  name: {{ .Values.storageClass.name }}
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
{{- end }}
```

4. **Use `.Capabilities` to detect cluster features:**
```yaml
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1/Ingress" }}
  # Use networking.k8s.io/v1 Ingress API
{{- else }}
  # Fall back to extensions/v1beta1
{{- end }}
```

5. **Library chart for shared logic:**
A `common` library chart provides cloud-agnostic templates (labels, pod specs, naming conventions) while each application chart handles cloud-specific resources.

**Best practice:** The chart should "just work" with sensible defaults and require minimal cloud-specific override for common use cases.

**Key points to mention:**
- Abstract cloud differences into values (not templates).
- Environment-specific values files per cloud provider.
- Conditional templates using `{{ if eq }}` on cloud provider value.
- `.Capabilities` for API version detection.
- Library chart for shared, cloud-agnostic logic.
- Avoid hardcoding cloud-specific resources; use values-driven configuration.

---

### Q78. Explain the security implications of Helm's architecture and how to mitigate them

**What the interviewer is looking for:** Comprehensive security analysis — from Tiller removal to modern concerns.

**Model answer:**

**Historical: Tiller (Helm 2) — Major Security Risk:**
- Tiller ran as a cluster-wide superuser (cluster-admin by default).
- Any user with access to Tiller could deploy anything anywhere.
- Tiller's release storage was in `kube-system` — shared across tenants.
- Mitigation: Helm 3 removed Tiller entirely. Helm CLI now uses the user's kubeconfig, following existing RBAC.

**Current Security Concerns (Helm 3):**

1. **Secrets in values.yaml / Git:**
   - Risk: Credentials committed to version control in plain text.
   - Mitigation: External Secrets Operator, Sealed Secrets, SOPS + helm-secrets, or Vault injection. Never store plain-text secrets in values.yaml.

2. **Release Secrets contain full history:**
   - Risk: Release Secrets store the complete rendered manifest and all merged values. If secrets were passed via values, they're in the release Secret forever.
   - Mitigation: Set `--history-max` to a small number. Use external secret stores. Rotate credentials.

3. **Chart supply chain:**
   - Risk: Malicious or compromised charts from untrusted repositories.
   - Mitigation: Use only trusted repositories. Verify chart signatures (`helm verify`). Use OCI registries with cosign/notation signing. Scan charts for vulnerabilities.

4. **RBAC and permissions:**
   - Risk: Helm uses the user's kubeconfig — if a developer has cluster-admin, `helm install` runs as cluster-admin.
   - Mitigation: Least-privilege RBAC. Namespace-scoped Helm operations. Service accounts with minimal permissions for CI/CD.

5. **CRD management:**
   - Risk: CRDs are cluster-scoped. A chart with CRDs can affect all namespaces.
   - Mitigation: Use `--skip-crds` when users lack cluster-admin. Manage CRDs separately.

6. **Chart provenance and integrity:**
   - Risk: Man-in-the-middle attacks on chart downloads.
   - Mitigation: Use HTTPS for repositories. Verify SHA256 digests in `Chart.lock`. Sign charts with GPG or cosign. Use OCI registries with content-addressable storage.

**Key points to mention:**
- Tiller removal in Helm 3 eliminated the biggest security risk.
- Never store secrets in values.yaml or commit to Git.
- Release Secrets contain full history — limit with `--history-max`.
- Verify chart provenance (signatures, digests).
- Least-privilege RBAC for Helm operations.
- Separate CRD management from application chart deployment.

---

### Q79. How do you handle Helm in a multi-tenant cluster?

**What the interviewer is looking for:** Understanding of isolation, RBAC, and namespace management for multiple tenants.

**Model answer:**

In a multi-tenant cluster, Helm must respect tenant isolation boundaries. Each tenant should only be able to deploy and manage releases in their assigned namespaces.

**Strategy:**

1. **Namespace-per-tenant:**
```
tenant-a/ -> namespace: tenant-a
tenant-b/ -> namespace: tenant-b
```

2. **RBAC per tenant:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: helm-user
  namespace: tenant-a
rules:
- apiGroups: [""]
  resources: ["secrets", "configmaps", "pods", "services"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
# ... other resources the tenant's charts need
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: helm-user-binding
  namespace: tenant-a
subjects:
- kind: User
  name: tenant-a-deployer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: helm-user
  apiGroup: rbac.authorization.k8s.io
```

3. **Release naming convention:**
```
<tenant-id>-<app-name>[-env]
Example: tenant-a-myapp-prod, tenant-b-myapp-dev
```
Or use tenant-scoped namespaces: the release name only needs to be unique within the namespace.

4. **Storage backend isolation:**
- Since Helm 3 stores release Secrets in the release namespace, each tenant's releases are naturally isolated in their own namespace.
- No cross-tenant release Secret access.

5. **Repository and registry isolation:**
- Per-tenant OCI registries or repository paths.
- Or a shared registry with tenant-specific namespaces/paths.

6. **Network and resource isolation:**
- NetworkPolicies to isolate tenant traffic.
- ResourceQuotas to prevent one tenant from consuming all cluster resources.
- LimitRanges to enforce sensible defaults.

7. **CRD considerations:**
- CRDs are cluster-scoped. Tenants should not be able to install CRDs.
- Use `--skip-crds` or manage CRDs at the platform level only.
- If tenants need CRDs, use a controlled approval process.

**Key points to mention:**
- Namespace-per-tenant with RBAC restricting Helm operations.
- Helm 3's namespace-scoped Secrets provide natural release isolation.
- Release naming convention that prevents collisions.
- NetworkPolicies and ResourceQuotas for tenant isolation.
- CRDs managed at platform level, not tenant level.
- Per-tenant OCI registry paths for chart storage.

---

### Q80. How would you migrate from Helm 2 to Helm 3?

**What the interviewer is looking for:** Understanding of the major architectural shift and migration strategy.

**Model answer:**

Helm 3 removed Tiller, changed release storage to Secrets (from ConfigMaps), and introduced several new features. Migration requires careful planning.

**Migration steps:**

1. **Install Helm 3 alongside Helm 2:**
```bash
# Helm 2 binary typically at /usr/local/bin/helm
# Helm 3 binary at /usr/local/bin/helm3 (or use helm3 plugin)
```

2. **Run the helm-2to3 plugin:**
```bash
helm3 plugin install https://github.com/helm/helm-2to3
```

3. **Migrate configuration:**
```bash
# Dry-run to see what will be migrated
helm3 2to3 move config --dry-run

# Migrate repositories, plugins, and other config
helm3 2to3 move config
```

4. **Convert releases:**
```bash
# Dry-run
helm3 2to3 convert my-release --dry-run

# Convert a specific release (Tiller -> Helm 3 storage)
helm3 2to3 convert my-release

# Convert all releases managed by Tiller
helm3 2to3 convert --all
```

The conversion process:
- Reads the release from Tiller's ConfigMap storage (`kube-system` namespace).
- Writes it as a Helm 3 Secret in the release's namespace.
- Creates a Helm 3 release record.

5. **Verify conversion:**
```bash
helm3 list -A
helm3 history my-release
helm3 get manifest my-release
```

6. **Clean up Helm 2:**
```bash
# Clean up Tiller (after verifying all releases are migrated)
helm3 2to3 cleanup --tiller-ns kube-system

# Remove Helm 2 config
helm3 2to3 cleanup --config
```

**Key changes to be aware of:**
- Release names are now scoped to namespaces (same name can exist in different namespaces).
- `helm install` without a release name auto-generates one.
- Chart API version: `apiVersion: v2` for Helm 3 charts (`v1` still supported for backwards compatibility).
- `requirements.yaml` is replaced by `dependencies` in `Chart.yaml`.
- CRDs are managed differently — use `crds/` directory.

**Key points to mention:**
- Use `helm-2to3` plugin for migration.
- `2to3 convert` reads from Tiller, writes to Helm 3 Secrets.
- `2to3 move config` migrates repositories and plugins.
- Clean up Tiller after verifying all releases are migrated.
- Test in a non-production cluster first.

---

### Q81. How do you implement a release promotion pipeline (dev -> staging -> prod)?

**What the interviewer is looking for:** Understanding of artifact promotion — the same chart artifact moves through environments, not rebuilt.

**Model answer:**

A Helm release promotion pipeline moves the **same chart package** (same `.tgz`, same digest) through environments using different values files. The chart is built once, then promoted.

**Pipeline design:**

```
Build Stage:
  git push main -> CI builds chart -> helm package -> push to dev OCI registry

Promote to Staging:
  Copy chart artifact from dev registry to staging registry (same digest)
  -> helm upgrade --install myapp-staging oci://registry-staging.example.com/myapp
     -f env/values-staging.yaml --atomic --wait

Promote to Production:
  Copy chart artifact from staging registry to prod registry (same digest)
  -> helm diff upgrade myapp-prod oci://registry-prod.example.com/myapp
     -f env/values-prod.yaml
  -> helm upgrade --install myapp-prod oci://registry-prod.example.com/myapp
     -f env/values-prod.yaml --atomic --wait
```

**Key principles:**

1. **Build once, deploy many:** The chart `.tgz` is packaged once. The same artifact (verified by SHA256) is promoted through environments without rebuilding.

2. **Registry promotion:**
```bash
# Promote chart from dev to staging
skopeo copy --all oci://dev-registry.example.com/myapp:v1.2.3 \
                   oci://staging-registry.example.com/myapp:v1.2.3
```

3. **Environment values separate from chart:**
```
platform-config/
└── env/
    ├── dev/
    │   ├── myapp-values.yaml
    ├── staging/
    │   ├── myapp-values.yaml
    └── prod/
        ├── myapp-values.yaml
```
Values files committed to a separate Git repository with strict access controls per environment directory.

4. **Gates and approvals:**
- Dev: auto-deploy on merge to main.
- Staging: auto-deploy on successful dev smoke tests.
- Prod: manual approval gate (GitHub PR, GitLab MR, or ticketing system).

5. **Testing at each stage:**
- Dev: `helm test` smoke tests after install.
- Staging: integration and performance tests.
- Prod: canary deployment with monitoring before full rollout.

**Key points to mention:**
- Same chart artifact (same digest) promoted through environments.
- Chart built once, values files differ per environment.
- Registry promotion via `skopeo copy` or equivalent.
- Manual approval gate for production.
- Progressive testing at each stage (dev -> staging -> prod).

---

### Q82. How does Helm handle rollbacks of StatefulSets with PVCs?

**What the interviewer is looking for:** Deep understanding of the interaction between Helm rollback and StatefulSet PVC lifecycle.

**Model answer:**

Helm rollback reverts the StatefulSet spec (image, env vars, resource limits, etc.) to a previous revision. However, PVCs created by `volumeClaimTemplates` are **not** affected by Helm rollback.

**What happens during rollback:**

1. Helm retrieves the manifest from the target revision.
2. The StatefulSet spec is reverted (image, env, replicas, etc.).
3. The StatefulSet controller reconciles the Pods to match the new (old) spec.
4. Pods are recreated with the reverted spec BUT bound to the **same PVCs**.

**What does NOT happen:**
- PVCs are not resized, recreated, or modified.
- Data in PVCs is not rolled back.
- `volumeClaimTemplates` changes are not applied to existing PVCs (they are immutable after creation).

**Implications:**
- If revision 5 used `image: myapp:v2` that wrote data in a new format, and you roll back to revision 3 with `image: myapp:v1`, the Pods will start but attempt to read data written by v2 — potentially causing application errors.
- PVC size increases are not rolled back (Kubernetes PVCs can only grow, not shrink).

**Mitigation:**

1. **Pre/post-rollback hooks for data migration:**
```yaml
annotations:
  "helm.sh/hook": pre-rollback
  "helm.sh/hook-weight": "-5"
```
Run a Job that performs data format downgrade before the rollback.

2. **Snapshot PVCs before upgrade:**
Use a pre-upgrade hook to take volume snapshots. If rollback is needed, restore from snapshot.

3. **Use Operators for stateful workloads:**
For databases (PostgreSQL, MySQL, MongoDB), use Kubernetes operators that handle data migration and rollback internally, rather than managing StatefulSets directly via Helm.

**Key points to mention:**
- Helm rollback reverts StatefulSet spec but NOT PVCs or data.
- PVCs from `volumeClaimTemplates` are immutable after creation.
- Data format changes between versions can cause application errors on rollback.
- Use hooks for data migration during rollback.
- Consider operators for complex stateful workloads.

---

### Q83. Explain how Helm's storage driver works. What are the alternatives?

**What the interviewer is looking for:** Deep understanding of the release storage architecture and configuration options.

**Model answer:**

Helm's storage driver is the component that persists release information. It stores the full release metadata (chart, values, manifest, status) as Kubernetes resources.

**Default driver: Secrets**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sh.helm.release.v1.<release-name>.v<revision>
  namespace: <release-namespace>
  labels:
    owner: helm
    status: deployed | superseded | failed | pending-install
    name: <release-name>
    version: "<revision>"
type: helm.sh/release.v1
data:
  release: <base64(gzip(JSON))>
```

**Storage driver options:**

| Driver | Type | Pros | Cons |
|--------|------|------|------|
| `secret` (default) | Kubernetes Secrets | Encrypted at rest (KMS), RBAC controllable | 1MB size limit per Secret |
| `configmap` | Kubernetes ConfigMaps | No size limit on values | Not encrypted at rest, visible in plain text |
| `sql` | SQL database (PostgreSQL, MySQL) | No API server storage, queryable | Requires external database, not built-in by default |

**Configuring the driver:**

```bash
# Via environment variable
export HELM_DRIVER=configmap
helm install my-release ./mychart

# Via flag
helm install my-release ./mychart --set helm.sh/storage=configmaps

# SQL driver (requires additional setup)
export HELM_DRIVER=sql
export HELM_DRIVER_SQL_CONNECTION_STRING="postgresql://..."
```

**How the driver works:**

1. On `helm install`: Creates a Secret (or ConfigMap) with the release metadata. Labeled with `owner: helm`, `status: deployed`, `version: 1`.

2. On `helm upgrade`: Marks the current `deployed` revision as `superseded`. Creates a new Secret with `status: deployed` and `version: N+1`.

3. On `helm rollback`: Marks the current `deployed` as `superseded`. Copies the target revision's data into a new Secret with `status: deployed` and a new version number.

4. On `helm uninstall`: Deletes all release Secrets (unless `--keep-history`).

5. On `helm list`: Queries Secrets (or ConfigMaps) with label `owner: helm` across namespaces.

**Key points to mention:**
- Default: Secrets with label `owner: helm` in the release namespace.
- `HELM_DRIVER` env var controls the storage backend.
- `secret` = encrypted at rest (default), `configmap` = plain text, `sql` = external DB.
- Each revision is a separate resource.
- Labels encode status (`deployed`, `superseded`, `failed`) and version.

---

### Q84. How would you implement automated Helm chart testing in CI/CD?

**What the interviewer is looking for:** Understanding of multi-layered testing from linting to integration tests.

**Model answer:**

Automated Helm chart testing should be multi-layered, catching issues at each stage of the pipeline.

**Testing layers:**

**1. Static Analysis (fast, no cluster needed):**

```bash
# Chart linting
helm lint ./mychart --strict

# Template rendering check (all values files)
helm template test-release ./mychart -f env/values-dev.yaml > /dev/null
helm template test-release ./mychart -f env/values-staging.yaml > /dev/null
helm template test-release ./mychart -f env/values-prod.yaml > /dev/null

# Values schema validation
helm lint ./mychart --values env/values-prod.yaml

# YAML linting (kubeconform)
helm template test-release ./mychart -f env/values-prod.yaml | \
  kubeconform -strict -kubernetes-version 1.30.0
```

**2. Unit Tests (helm-unittest):**

```yaml
# tests/deployment_test.yaml
suite: Deployment tests
templates:
  - deployment.yaml
tests:
  - it: should set default replica count to 1
    asserts:
      - equal:
          path: spec.replicas
          value: 1
  - it: should set custom replica count
    set:
      replicaCount: 5
    asserts:
      - equal:
          path: spec.replicas
          value: 5
  - it: should set correct image
    set:
      image:
        repository: myapp
        tag: v2.1.0
    asserts:
      - contains:
          path: spec.template.spec.containers[0].image
          content: "myapp:v2.1.0"
```

```bash
helm unittest ./mychart
```

**3. Integration Tests (requires a cluster):**

```bash
# Deploy to ephemeral namespace
helm upgrade --install test-release ./mychart \
  -f env/values-ci.yaml \
  -n test-${CI_PIPELINE_ID} \
  --create-namespace \
  --wait --timeout 5m

# Run chart tests
helm test test-release -n test-${CI_PIPELINE_ID} --logs

# Verify resources
kubectl get deployment test-release-mychart -n test-${CI_PIPELINE_ID} \
  -o jsonpath='{.status.readyReplicas}' | grep 1

# Run application-level integration tests (curl, API calls)
curl -f http://test-release-mychart.test-${CI_PIPELINE_ID}.svc.cluster.local/healthz

# Cleanup
helm uninstall test-release -n test-${CI_PIPELINE_ID}
kubectl delete namespace test-${CI_PIPELINE_ID}
```

**4. Upgrade Testing:**

```bash
# Install old version
helm install test-release oci://registry.example.com/myapp --version 1.2.0 \
  -n test-upgrade-${CI_PIPELINE_ID} --create-namespace --wait

# Upgrade to new version
helm upgrade test-release ./mychart \
  -n test-upgrade-${CI_PIPELINE_ID} --wait

# Verify upgrade succeeded
helm history test-release -n test-upgrade-${CI_PIPELINE_ID} --max 1
```

**CI pipeline structure:**
```yaml
# .github/workflows/chart-test.yml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: helm lint ./mychart --strict

  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: helm unittest ./mychart

  integration-test:
    needs: [lint, unit-test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: helm/kind-action@v1  # creates kind cluster
      - run: |
          helm upgrade --install test-release ./mychart -n test --create-namespace --wait
          helm test test-release -n test --logs
```

**Key points to mention:**
- Multi-layered: lint -> template validation -> unit tests -> integration tests -> upgrade tests.
- `helm lint --strict` for static analysis.
- `helm-unittest` for template unit testing.
- Ephemeral namespaces for integration tests (create and destroy per pipeline run).
- Upgrade testing to catch compatibility issues.
- CI/CD: test all values files for all environments.

---

### Q85. How do you handle breaking changes in chart upgrades?

**What the interviewer is looking for:** Understanding of backwards compatibility, migration strategies, and user communication.

**Model answer:**

Breaking changes in a Helm chart require careful handling to avoid production outages. A breaking change is any modification that, if applied with the same values file, would cause the upgrade to fail or the application to misbehave.

**Types of breaking changes:**
- Renamed, removed, or restructured values keys.
- Changed default values that affect application behavior.
- Changed template structure (resource names, labels, selectors).
- Changed dependency versions.
- Removed or renamed templates.

**Strategy for handling breaking changes:**

**1. Semantic versioning for charts:**
- **MAJOR** version bump for breaking changes (1.x.y -> 2.0.0).
- **MINOR** version bump for backwards-compatible new features.
- **PATCH** version bump for backwards-compatible fixes.

**2. Deprecation period:**
- Announce the breaking change in the release notes.
- Keep old values keys working alongside new keys for one MINOR version.
- Add deprecation warnings in `NOTES.txt` or via validation.

```yaml
# Support both old and new keys during deprecation
database:
  host: {{ .Values.database.host | default .Values.databaseHost }}
  # "databaseHost" is deprecated, use "database.host" instead
```

**3. Schema validation for migration guidance:**
```json
{
  "properties": {
    "databaseHost": {
      "type": "string",
      "deprecated": true,
      "description": "DEPRECATED: Use database.host instead"
    }
  }
}
```

**4. Migration documentation:**
```
## Upgrading from 2.x to 3.0.0

### Breaking Changes:
- `service.port` is now `service.ports.http`
- `ingress.annotations` is now `ingress.metadata.annotations`

### Migration Steps:
1. Update your values files:
   - Replace `service.port: 80` with `service.ports.http: 80`
   - Move `ingress.annotations` to `ingress.metadata.annotations`
2. Run `helm diff upgrade` to verify changes
3. Run `helm upgrade` with updated values
```

**5. CI/CD guardrails:**
- Test upgrades from previous version in CI.
- `helm diff upgrade` with `--detailed-exitcode` to detect changes before applying.
- Autogenerate migration scripts where possible (e.g., JSON path transforms).

**6. Communication:**
- CHANGELOG.md in the chart repository.
- Release notes on OCI registry or chart repository.
- Slack/Teams/email notification for major version bumps.

**Key points to mention:**
- SemVer for charts — MAJOR bump for breaking changes.
- Deprecation period with dual support for old and new keys.
- `values.schema.json` for documenting deprecated fields.
- Migration guides for each major version.
- Test upgrades in CI/CD from previous versions.
- `helm diff upgrade` for pre-deployment verification.

---

### Q86. Design a disaster recovery strategy for Helm-managed deployments

**What the interviewer is looking for:** Comprehensive DR planning — from release metadata backup to full cluster recovery.

**Model answer:**

A disaster recovery strategy for Helm must cover two things: recovering Helm's release metadata (so Helm can manage the releases again) and recovering the actual Kubernetes resources and application data.

**1. Backup of release metadata:**

Option A: **Velero backup of release Secrets:**
```bash
velero backup create helm-releases --include-resources secrets \
  --selector owner=helm
```

Option B: **Scripted export of release data:**
```bash
#!/bin/bash
# Export all release metadata
for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  for secret in $(kubectl get secrets -n $ns -l owner=helm -o name); do
    kubectl get $secret -n $ns -o yaml > "backup/$ns-$(echo $secret | cut -d/ -f2).yaml"
  done
done
```

Option C: **Git-based manifest backup:**
```bash
# Export all release manifests to Git (not full metadata, but sufficient for recovery)
for release in $(helm list -A -q); do
  ns=$(helm list -n $release -o json | jq -r '.[0].namespace')
  helm get manifest $release -n $ns > "manifests/$ns-$release.yaml"
  helm get values $release -n $ns --all > "values/$ns-$release.yaml"
done
```

**2. Backup of values and chart sources:**
- Chart source code in Git (covered by standard Git backup).
- Environment values files in Git (covered by standard Git backup).
- OCI registry backup (covered by registry replication).

**3. Full cluster DR scenario:**

```
Recovery steps:
1. Restore or recreate the Kubernetes cluster (infrastructure).
2. Restore CRDs first (from crds/ directory or Git).
3. Restore containers: image registry should be available (replicate registries).
4. Restore release Secrets from backup (Velero or manual script).
5. Helm now "sees" the releases — helm list -A should show them.
6. Restore persistent data: PVCs from storage snapshots.
7. Verify all releases: helm status <release> -n <namespace>.
```

**4. Testing DR:**
- Regular DR drills — restore a subset of releases to a test cluster.
- Validate that restored releases are functional.
- Measure RTO (Recovery Time Objective) and RPO (Recovery Point Objective).

**5. Multi-region DR:**
```
Active region: full stack running.
Passive region: OCI registry replicated, values in Git,
                Velero backups replicated to passive region's object storage.
Recovery: deploy Helm releases in passive region, restore PVCs from snapshots.
```

**Key points to mention:**
- Backup release Secrets (metadata) + values files (configuration) + chart source (Git).
- Velero for automated backup of release Secrets.
- Git-based manifest and values export as a lightweight alternative.
- Restore CRDs before releases.
- Regular DR drills to validate recovery procedures.
- Multi-region: replicate OCI registries and backup storage.

---

### Q87. How does Helm work with custom resources (CRDs) and admission webhooks?

**What the interviewer is looking for:** Understanding of the interaction between Helm and Kubernetes extensibility mechanisms.

**Model answer:**

**CRDs with Helm:**

CRDs are cluster-scoped resources that define new resource types. Helm handles them in two ways:

1. **`crds/` directory (special handling):**
- Files in `crds/` are installed before any templates.
- NEVER upgraded, NEVER deleted by Helm.
- No Helm labels, no revision tracking.
- **Implication:** If a chart template creates a custom resource (instance of a CRD), the CRD must already exist. The `crds/` directory ensures this.

2. **`templates/` directory (standard handling):**
- CRDs in `templates/` ARE managed like any resource.
- Can be upgraded, deleted, rolled back.
- Can use Go template syntax.
- **Risk:** Deleting a CRD deletes all instances of that CR across the cluster. Use with extreme caution.

**Admission webhooks with Helm:**

Admission webhooks (ValidatingWebhookConfiguration, MutatingWebhookConfiguration) intercept API requests and can modify or reject them. Interaction with Helm:

1. **Webhook blocks Helm operations:**
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
webhooks:
  - name: require-labels.example.com
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        resources: ["pods"]
    failurePolicy: Fail  # If webhook is down, ALL pod creates fail
```
If a webhook has `failurePolicy: Fail` and the webhook service is unavailable, Helm upgrades will fail.

2. **Ordering problem:**
- A chart installs a webhook service AND creates resources that the webhook validates.
- During `helm install`, the webhook service may not be ready when other resources are created -> the webhook call fails -> the install fails.
- **Solution:** Use `helm.sh/hook-weight` to ensure the webhook service is created and ready before other resources.

3. **Webhook for Helm release Secrets:**
- A MutatingWebhookConfiguration could encrypt/decrypt release Secrets automatically.
- A ValidatingWebhookConfiguration could enforce that all Helm releases meet certain policies before being applied.

**Best practices:**

1. **CRDs:** Put CRDs in `crds/` for simple cases. For managed CRDs (with lifecycle), put in `templates/` but understand the risks.

2. **Webhooks with `failurePolicy: Fail`:**
- Ensure the webhook service is highly available.
- Consider `failurePolicy: Ignore` for non-critical validations.
- Use hook weights to order webhook installation before resources that depend on it.

3. **Testing:** Always test charts with admission webhooks enabled. An `--atomic` upgrade can roll back if a webhook blocks the operation.

**Key points to mention:**
- CRDs in `crds/` = install-only, no lifecycle management.
- CRDs in `templates/` = fully managed, but dangerous (deleting CRDs deletes all CR instances).
- Webhooks with `failurePolicy: Fail` can block Helm operations if webhook is unavailable.
- Hook weights solve webhook ordering issues.
- `--atomic` provides safety if a webhook blocks an operation.

---

### Q88. What are the tradeoffs between using Helm, Kustomize, and raw kubectl?

**What the interviewer is looking for:** Nuanced understanding of when to use each tool and their respective strengths/weaknesses.

**Model answer:**

| Aspect | Helm | Kustomize | Raw kubectl |
|--------|------|-----------|-------------|
| **Approach** | Templating (values -> rendered YAML) | Overlay/patch (base -> patches) | Direct YAML application |
| **Reusability** | Charts (packaged, versioned, shared) | Bases + overlays (directory-based) | Manual copy-paste |
| **Versioning** | Built-in (chart version, app version, revisions) | Git-based (no built-in versioning) | None |
| **Lifecycle** | Full (install, upgrade, rollback, uninstall, history) | None (only `kubectl apply -k`) | None (only `kubectl apply -f`) |
| **State tracking** | Release Secrets track what was deployed | No state tracking (relies on `last-applied-configuration` annotation) | No state tracking |
| **Rollback** | `helm rollback` (deterministic, based on stored revisions) | `kubectl rollout undo` (Deployment only) | Manual re-apply |
| **Dependencies** | Subcharts, `dependencies` in Chart.yaml | `resources:` field | Manual ordering |
| **Learning curve** | Moderate (Go templates) | Lower (mostly YAML) | Low (but doesn't scale) |
| **Distribution** | OCI registries, chart repos | Git repositories | N/A |
| **Kubernetes integration** | External tool (requires separate install) | Built into `kubectl` (`kubectl kustomize`) | Built into `kubectl` |
| **Best for** | Complex applications, package distribution, multi-env deployments | Environment-specific patching, GitOps, simple config customization | Ad-hoc operations, one-off resources |

**When to use Helm:**
- Complex multi-resource applications.
- You need versioning, rollback, and release lifecycle management.
- You're distributing an application to others (chart as a package).
- Multi-environment deployments with parameterization.
- CI/CD pipelines that need idempotent `upgrade --install`.

**When to use Kustomize:**
- Simple environment-specific patching (dev image vs prod image).
- GitOps workflows where Git is the single source of truth (ArgoCD native).
- You want to avoid Go template complexity.
- Modifying third-party YAML (not designed for your environment).

**When to use raw kubectl:**
- Quick ad-hoc operations (`kubectl create`, `kubectl delete`).
- Debugging (apply a temporary debug Pod).
- One-off resources that don't need lifecycle management.

**They can be combined:**
```bash
# Helm renders templates, Kustomize applies environment-specific patches
helm template my-release ./mychart -f values.yaml | kustomize build | kubectl apply -f -
```

**Key points to mention:**
- Helm = package manager with lifecycle; Kustomize = configuration customization; kubectl = direct API calls.
- Helm's key advantage: revision tracking, deterministic rollback.
- Kustomize's key advantage: simpler YAML-based patching, native kubectl integration.
- They complement each other — use Helm for packaging, Kustomize for final patching.
- ArgoCD supports both natively in the same Application.

---

### Q89. How would you troubleshoot a production outage caused by a Helm upgrade?

**What the interviewer is looking for:** Incident response methodology — fast diagnosis, rollback, and root cause analysis.

**Model answer:**

**Immediate response (first 5 minutes):**

**1. Roll back — stop the bleeding:**
```bash
# Quickest: roll back to the last known good revision
helm rollback my-release -n prod

# If revision numbers are unclear:
helm history my-release -n prod
helm rollback my-release <last-successful-revision> -n prod
```

If `helm rollback` fails (e.g., release in `failed` state):
```bash
# Force uninstall and reinstall from a known good chart version
helm uninstall my-release -n prod --keep-history
helm install my-release oci://registry.example.com/myapp --version <known-good> \
  -f values-prod.yaml -n prod --wait --atomic
```

**2. Diagnose the failure (concurrent or after rollback):**

```bash
# See what Helm tried to deploy
helm get manifest my-release -n prod --revision <failed-revision>

# Compare with the previous good revision
diff <(helm get manifest my-release -n prod --revision <good>) \
     <(helm get manifest my-release -n prod --revision <failed>)

# Check values that were used
helm get values my-release -n prod --revision <failed> --all

# Check Kubernetes events from the failure window
kubectl get events -n prod --sort-by='.lastTimestamp' | tail -50

# Inspect release Secret for error details
kubectl get secret sh.helm.release.v1.my-release.v<failed> -n prod \
  -o jsonpath='{.data.release}' | base64 -d | gunzip | jq .info.description
```

**Root cause analysis (post-recovery):**

1. **Template changes:** Did the chart template change cause the issue? Use `helm diff` to see what changed.
2. **Values changes:** Did someone pass incorrect values? Check the failed revision's `helm get values`.
3. **Image changes:** Was a bad container image deployed? Check if the image tag changed.
4. **Immutable field changes:** Did someone try to change an immutable field (e.g., Deployment selector)?
5. **Resource constraints:** Did the new configuration exceed resource quotas?
6. **Hook failure:** Did a pre-upgrade hook fail, blocking the upgrade?
7. **Admission webhook:** Did a ValidatingWebhookConfiguration reject the change?
8. **Kubernetes API version deprecation:** Was a now-removed API version used?

**Prevention (post-incident improvements):**

1. **Always preview with `helm diff` before production upgrades.**
2. **Use `--atomic` for automatic rollback on failure.**
3. **Implement CI/CD approval gates for production.**
4. **Test upgrades from previous version in CI.**
5. **Monitor Helm release status in production alerts.**

**Key points to mention:**
- First action: `helm rollback` to stop the bleeding.
- Use `helm diff` to compare good vs failed revisions.
- Check release Secrets for error descriptions.
- Root cause categories: template change, values change, bad image, immutable field, hook failure, webhook rejection.
- Prevention: `helm diff` before upgrade, `--atomic`, CI/CD testing.

---

### Q90. Explain how Helm's release secrets are structured and how to inspect them manually

**What the interviewer is looking for:** Deep operational knowledge — ability to inspect Helm's internals without the Helm CLI.

**Model answer:**

Helm stores each release revision as a Kubernetes Secret (by default) in the release's namespace. Understanding their structure is essential for debugging when the Helm CLI is unavailable or behaving unexpectedly.

**Naming convention:**
```
sh.helm.release.v1.<release-name>.v<revision>
```
Examples: `sh.helm.release.v1.myapp-prod.v1`, `sh.helm.release.v1.myapp-prod.v7`

**Secret structure:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sh.helm.release.v1.myapp-prod.v3
  namespace: prod
  labels:
    owner: helm                         # Identifies as Helm-managed
    status: deployed                    # deployed | superseded | failed | pending-install | pending-upgrade | pending-rollback
    name: myapp-prod                    # Release name
    version: "3"                        # Revision number (string)
    modifiedAt: "1705284134"            # Unix timestamp of modification
    chart: myapp-1.2.0                  # Chart name and version
    app.kubernetes.io/managed-by: Helm  # Standard K8s label
  annotations:
    # Optional: additional metadata
type: helm.sh/release.v1
data:
  release: <base64(gzip(JSON))>
```

**The `data.release` payload (decoded):**

```json
{
  "name": "myapp-prod",
  "info": {
    "first_deployed": "2026-07-11T10:00:00Z",
    "last_deployed": "2026-07-11T14:30:00Z",
    "deleted": "",
    "description": "Upgrade complete",
    "status": "deployed"
  },
  "chart": {
    "metadata": {
      "name": "myapp",
      "version": "1.2.0",
      "appVersion": "2.1.0",
      "description": "A Helm chart for MyApp"
    },
    "values": { /* All default values from values.yaml */ },
    "templates": [
      { "name": "templates/deployment.yaml", "data": "<base64-template-content>" },
      { "name": "templates/service.yaml", "data": "<base64-template-content>" }
    ],
    "files": [],
    "dependencies": [
      {
        "metadata": { "name": "postgresql", "version": "12.1.0" },
        "values": { /* subchart values */ }
      }
    ]
  },
  "config": { /* Merged configuration values used for this release */ },
  "manifest": "---\n# Source: myapp/templates/deployment.yaml\napiVersion: apps/v1\n...",
  "version": 3,
  "namespace": "prod"
}
```

**Manual inspection commands:**

```bash
# List all release secrets
kubectl get secrets -n prod -l owner=helm

# Find the deployed revision
kubectl get secrets -n prod -l owner=helm -l name=myapp-prod -l status=deployed

# Extract and parse release data
kubectl get secret sh.helm.release.v1.myapp-prod.v3 -n prod \
  -o jsonpath='{.data.release}' | base64 -d | gunzip | jq .

# Get just the status
kubectl get secret sh.helm.release.v1.myapp-prod.v3 -n prod \
  -o jsonpath='{.data.release}' | base64 -d | gunzip | jq .info.status

# Get the rendered manifest
kubectl get secret sh.helm.release.v1.myapp-prod.v3 -n prod \
  -o jsonpath='{.data.release}' | base64 -d | gunzip | jq -r .manifest

# Get the values used
kubectl get secret sh.helm.release.v1.myapp-prod.v3 -n prod \
  -o jsonpath='{.data.release}' | base64 -d | gunzip | jq .config
```

**When this knowledge is useful:**
- Helm CLI is unavailable or misconfigured.
- Helm binary version mismatch with the cluster.
- Debugging `helm list` failures (manually query Secrets).
- Auditing values that were actually deployed.
- Migrating releases between clusters.

**Key points to mention:**
- Naming: `sh.helm.release.v1.<name>.v<rev>`.
- Labels: `owner: helm`, `status`, `name`, `version`.
- Data: base64-encoded, gzip-compressed JSON.
- Contains full chart, values, manifest, and metadata.
- Inspection: `base64 -d | gunzip | jq .`

---

### Q91. How do you handle Helm in a zero-trust environment?

**What the interviewer is looking for:** Security-first architecture — every component authenticated, authorized, and verified.

**Model answer:**

A zero-trust environment assumes no implicit trust — every component, connection, and artifact must be authenticated, authorized, and verified.

**1. Chart provenance verification:**
```bash
# Verify chart signatures (no unsigned charts admitted)
helm verify myapp-1.0.0.tgz --keyring /etc/helm/trusted-keys.gpg

# OCI charts with cosign
cosign verify --key cosign.pub oci://registry.internal/myapp:v1.0.0
```
CI/CD pipeline rejects any unsigned chart.

**2. Authentication and authorization:**
- **Short-lived credentials:** Use workload identity (IRSA, Workload Identity Federation) instead of static kubeconfig files.
- **Least-privilege RBAC:** Helm service accounts have the minimum permissions needed. One SA per environment.
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm-deployer
  namespace: prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: helm-deployer
  namespace: prod
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: helm-deployer
  namespace: prod
subjects:
- kind: ServiceAccount
  name: helm-deployer
  namespace: prod
roleRef:
  kind: Role
  name: helm-deployer
  apiGroup: rbac.authorization.k8s.io
```

**3. Network policies:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-helm-access
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: [] # Deny all egress from release namespace except to API server
```

**4. Audit logging:**
- Enable Kubernetes audit logging to track all Helm API calls.
- Ship audit logs to a SIEM for alerting on anomalous Helm activity.
```yaml
# Audit policy — log all Helm-related API calls
rules:
- level: RequestResponse
  users: ["system:serviceaccount:prod:helm-deployer"]
  verbs: ["create", "update", "patch", "delete"]
```

**5. Policy enforcement (OPA/Gatekeeper):**
```rego
# Reject Helm releases that don't meet security requirements
package helm

violation[{"msg": msg}] {
  input.review.kind.kind == "Secret"
  input.review.object.metadata.labels.owner == "helm"
  # Check that release has required security annotations
  not input.review.object.metadata.annotations["security.example.com/audited"]
  msg := "Helm release must have security audit annotation"
}
```

**6. Immutable infrastructure:**
- Use `--atomic` for all deployments — no partial states.
- `--wait` to ensure resources are healthy before marking complete.
- Read-only file systems for chart directories after verification.
- Ephemeral build environments (CI/CD containers) that are destroyed after deployment.

**7. Registry and repository security:**
- Internal OCI registries only (no public registries).
- Mutual TLS between Helm CLI and internal registries.
- Image and chart scanning in registries.

**Key points to mention:**
- Verify all chart signatures before deployment.
- Short-lived credentials; least-privilege RBAC.
- NetworkPolicies to restrict Helm's blast radius.
- Audit logging for all Helm API calls.
- OPA/Gatekeeper for automated policy enforcement.
- Internal registries only; mutual TLS.

---

### Q92. What is the `Capabilities` built-in object and how is it used?

**What the interviewer is looking for:** Understanding of cluster-aware template rendering.

**Model answer:**

The `.Capabilities` built-in object provides information about the target Kubernetes cluster during template rendering. It allows templates to adapt their output based on what the cluster supports.

**Structure:**

```json
{
  "APIVersions": [
    "v1",
    "apps/v1",
    "networking.k8s.io/v1",
    "cert-manager.io/v1",
    ...
  ],
  "KubeVersion": {
    "Major": "1",
    "Minor": "30",
    "GitVersion": "v1.30.0",
    "Platform": "linux/amd64"
  },
  "HelmVersion": {
    "Version": "v3.15.0",
    "GitCommit": "..."
  }
}
```

**Checking API version availability:**

```yaml
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1/Ingress" }}
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  {{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1/IngressClass" }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
{{- else if .Capabilities.APIVersions.Has "networking.k8s.io/v1beta1/Ingress" }}
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
{{- else }}
apiVersion: extensions/v1beta1
kind: Ingress
{{- end }}
```

**Checking Kubernetes version:**

```yaml
{{- if semverCompare ">=1.30-0" .Capabilities.KubeVersion.Version }}
  # Use features available in K8s 1.30+
{{- else }}
  # Fall back to older API versions
{{- end }}
```

**Checking for specific CRDs (via API versions):**

```yaml
{{- if .Capabilities.APIVersions.Has "monitoring.coreos.com/v1/ServiceMonitor" }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
# ... Prometheus monitoring configuration
{{- end }}
```

**Querying at render time:**
`.Capabilities` is populated by Helm querying the Kubernetes API server at render time (during `helm install`/`helm upgrade`). With `helm template` (no API server), `.Capabilities` contains only the built-in Kubernetes API versions.

**Key points to mention:**
- Provides `.APIVersions` (list of available API versions) and `.KubeVersion` (cluster version info).
- Use `.Capabilities.APIVersions.Has "group/version/Kind"` for conditional API version selection.
- Use `semverCompare` to check `.Capabilities.KubeVersion.Version`.
- Enables writing charts that work across multiple Kubernetes versions.
- With `helm template`, only built-in API versions are available.

---

### Q93. How do you implement progressive delivery with Helm and service mesh?

**What the interviewer is looking for:** Advanced deployment patterns combining Helm, Istio/Linkerd, and progressive delivery.

**Model answer:**

Progressive delivery extends canary deployments with automated analysis and decision-making. Helm creates the Kubernetes resources; the service mesh manages traffic; an analysis tool verifies health.

**Architecture:**

```
Helm (creates resources) -> Service Mesh (traffic routing) -> Analysis Tool (verification)
```

**Implementation with Helm + Istio + Argo Rollouts:**

**1. Helm chart includes Istio resources:**

```yaml
# templates/destination-rule.yaml (Istio)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  host: {{ include "mychart.fullname" . }}
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
---
# templates/virtual-service.yaml (Istio)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  hosts:
    - {{ include "mychart.fullname" . }}
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: {{ include "mychart.fullname" . }}
            subset: canary
    - route:
        - destination:
            host: {{ include "mychart.fullname" . }}
            subset: stable
          weight: 100
```

**2. Progressive delivery flow:**

```bash
# Step 1: Deploy stable version
helm upgrade --install myapp-stable ./mychart \
  --set deployment.version=stable \
  -n prod --wait

# Step 2: Deploy canary version alongside stable
helm upgrade --install myapp-canary ./mychart \
  --set deployment.version=canary \
  --set replicaCount=1 \
  -n prod --wait

# Step 3: Route 5% traffic to canary via Istio VirtualService update
helm upgrade myapp-stable ./mychart \
  --set canary.trafficWeight=5 \
  -n prod --wait

# Step 4: Automated analysis (Prometheus queries)
# Check canary error rate vs stable error rate
# Check canary latency p99 vs stable latency p99

# Step 5: If analysis passes, increase traffic
helm upgrade myapp-stable ./mychart \
  --set canary.trafficWeight=25 \
  -n prod --wait

# Step 6: Repeat analysis -> 50% -> 100% -> promote canary to stable

# Step 7: If analysis fails at any step, rollback
helm upgrade myapp-stable ./mychart \
  --set canary.trafficWeight=0 \
  -n prod --wait
helm uninstall myapp-canary -n prod
```

**3. Automated analysis with tools like Flagger or Analysis Template:**

```yaml
# Argo Rollouts AnalysisTemplate
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
    - name: error-rate
      interval: 1m
      successCondition: result < 0.01
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            rate(http_requests_total{version="canary",status=~"5.."}[1m]) /
            rate(http_requests_total{version="canary"}[1m])
```

**4. Blast radius control:**
- Start with `< 5%` traffic to canary.
- Use HTTP header-based routing for internal testing (`x-canary: true`).
- Set maximum error budget before auto-rollback.
- Monitor canary health metrics independently.

**Key points to mention:**
- Helm creates resources (Deployments, Services, Istio resources).
- Service mesh (Istio/Linkerd) handles traffic splitting and routing.
- Analysis tool verifies canary health before promoting.
- Automated rollback if canary metrics degrade.
- Increase traffic gradually: 5% -> 25% -> 50% -> 100%.
- Separation of concerns: Helm for deployment, mesh for traffic, analysis for verification.

---

### Q94. How would you build an internal Helm chart platform for a large organization?

**What the interviewer is looking for:** Platform engineering thinking — building a self-service chart ecosystem at scale.

**Model answer:**

An internal Helm chart platform provides self-service chart development, publishing, discovery, and deployment for hundreds of developers across multiple teams.

**Platform components:**

**1. Chart development toolkit:**
```
- Starter templates (helm create --starter)
- Library chart with standardized labels, probes, security contexts
- Chart development CLI with scaffolding, linting, testing
- Documentation templates
- VS Code extension for Helm template editing
```

**2. Centralized OCI registry:**
```
Internal Harbor, Artifactory, or ECR registry
├── /charts/public/        # Organization-wide charts
├── /charts/team-a/        # Team-specific charts
├── /charts/team-b/
└── /charts/platform/      # Platform/infrastructure charts
```

**3. CI/CD pipeline per chart (automated):**
```yaml
# .github/workflows/chart-release.yml (template for all chart repos)
name: Release Chart
on:
  push:
    tags: ['v*']
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lint
        run: helm lint . --strict
      - name: Unit Test
        run: helm unittest .
      - name: Integration Test
        run: | # Deploy to test cluster
      - name: Package
        run: helm package .
      - name: Push to OCI
        run: |
          helm registry login $REGISTRY
          helm push *.tgz oci://$REGISTRY/charts/
      - name: Sign
        run: cosign sign oci://$REGISTRY/charts/myapp:$VERSION
```

**4. Chart catalog (internal "ArtifactHub"):**
- Backstage plugin showing available charts, versions, maintainers.
- API for programmatic chart discovery.
- Deprecation and vulnerability scanning visibility.

**5. Deployment platform (self-service):**
```yaml
# Developer Portal — deploy a chart with a few clicks/form fields
apiVersion: example.com/v1
kind: HelmDeployment
metadata:
  name: myapp-prod
spec:
  chart: oci://registry.internal/charts/myapp
  version: 2.1.0
  values:
    replicaCount: 3
    resources:
      limits:
        memory: 1Gi
  environment: prod
  namespace: myapp-prod
```

Backed by an operator or ArgoCD that reconciles `HelmDeployment` CRs.

**6. Governance and policy:**
- OPA/Gatekeeper policies for chart compliance.
- Required image scanning before chart can be released.
- Approval workflow for production deployments.
- Audit trail of all deployments.

**7. Metrics and observability:**
```
- Charts published per month
- Active chart versions in use
- Deployment frequency per team
- Deployment failure rate
- Time from chart push to production
```

**8. Support structure:**
- Internal documentation and runbooks.
- Office hours for chart development help.
- Chart review process (similar to code review).
- #helm-help Slack channel.

**Key points to mention:**
- Self-service: developers can create, publish, and deploy charts without platform team intervention.
- Standardized tooling: starter templates, library charts, CI/CD pipelines.
- Centralized OCI registry with RBAC per team.
- Internal catalog (Backstage) for chart discovery.
- Governance via OPA/Gatekeeper policies.
- Metrics for platform health monitoring.

---

### Q95. How does the Helm SDK work and when would you use it?

**What the interviewer is looking for:** Understanding of Helm as a Go library for programmatic chart operations.

**Model answer:**

The Helm SDK exposes Helm's core functionality as Go packages, enabling programmatic chart installation, upgrade, rollback, and uninstallation from Go applications (operators, controllers, custom CLIs).

**Core packages:**

```go
import (
    "helm.sh/helm/v3/pkg/action"      // Actions: install, upgrade, rollback
    "helm.sh/helm/v3/pkg/chart"       // Chart loading and parsing
    "helm.sh/helm/v3/pkg/chart/loader" // Chart loading from filesystem/OCI
    "helm.sh/helm/v3/pkg/cli"         // CLI environment setup
    "helm.sh/helm/v3/pkg/release"     // Release data structures
    "helm.sh/helm/v3/pkg/repo"        // Repository operations
    "helm.sh/helm/v3/pkg/storage/driver" // Storage backends
)
```

**Example: Programmatic chart install:**

```go
package main

import (
    "log"
    "os"
    "helm.sh/helm/v3/pkg/action"
    "helm.sh/helm/v3/pkg/chart/loader"
    "helm.sh/helm/v3/pkg/cli"
)

func main() {
    settings := cli.New()

    // Create install action
    actionConfig := new(action.Configuration)
    if err := actionConfig.Init(
        settings.RESTClientGetter(),
        "default",           // namespace
        os.Getenv("HELM_DRIVER"), // storage driver
        log.Printf,
    ); err != nil {
        log.Fatal(err)
    }

    // Create install client
    client := action.NewInstall(actionConfig)
    client.ReleaseName = "my-release"
    client.Namespace = "default"
    client.CreateNamespace = true
    client.Wait = true
    client.Atomic = true

    // Load chart
    chartPath := "./mychart"
    chart, err := loader.Load(chartPath)
    if err != nil {
        log.Fatal(err)
    }

    // Set values
    vals := map[string]interface{}{
        "replicaCount": 3,
        "image": map[string]interface{}{
            "tag": "v2.1.0",
        },
    }

    // Install
    release, err := client.Run(chart, vals)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Installed release: %s (revision: %d)", release.Name, release.Version)
}
```

**When to use the Helm SDK:**

1. **Kubernetes Operators:** An operator that manages complex applications composed of multiple charts. The operator uses Helm SDK to install and upgrade charts as part of its reconciliation loop.

2. **Custom deployment platforms:** An internal platform that wraps Helm with custom business logic, approval workflows, and audit trails.

3. **ArgoCD/Flux-style GitOps controllers:** Tools that reconcile Git state with cluster state using Helm charts.

4. **Custom CLI tools:** Internal tools that combine Helm operations with organization-specific logic (e.g., automatic environment-specific configuration).

5. **Automated testing frameworks:** Tools that programmatically install charts, run tests, and clean up.

**Key points to mention:**
- Core packages: `pkg/action` (install/upgrade/rollback), `pkg/chart` (loading), `pkg/cli` (environment).
- Helm SDK powers tools like ArgoCD, Helmfile, and Terraform Helm provider.
- Full programmatic control over install, upgrade, rollback, list, uninstall.
- Useful for operators, custom platforms, GitOps controllers.
- Go-only; no SDK for other languages (but REST API wrappers exist).

---

### Q96. Explain how to implement Helm chart signing and verification in an automated pipeline

**What the interviewer is looking for:** Supply chain security — signing, verification, and key management in CI/CD.

**Model answer:**

Chart signing ensures the integrity and provenance of Helm charts. An automated pipeline should sign on publish and verify before deployment.

**Pipeline implementation:**

**1. Signing (publish pipeline):**

```yaml
# .github/workflows/release.yml
name: Release and Sign Chart
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Package chart
        run: |
          helm dependency update .
          helm package . --sign \
            --key "$GPG_KEY_NAME" \
            --keyring <(echo "$GPG_PRIVATE_KEY" | gpg --import)

      - name: Push to OCI registry
        run: |
          helm registry login $REGISTRY -u $USER -p $PASSWORD
          helm push *.tgz oci://$REGISTRY/charts/

      - name: Sign OCI artifact with cosign
        run: |
          cosign sign --key cosign.key \
            oci://$REGISTRY/charts/myapp:${{ github.ref_name }}

      - name: Verify signing for quality gate
        run: |
          cosign verify --key cosign.pub \
            oci://$REGISTRY/charts/myapp:${{ github.ref_name }}
```

**2. Verification (deployment pipeline):**

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  workflow_dispatch:
    inputs:
      chart_version:
        required: true
      environment:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Verify chart signature (cosign)
        run: |
          cosign verify --key $TRUSTED_PUBLIC_KEY \
            oci://$REGISTRY/charts/myapp:${{ inputs.chart_version }}

      - name: Verify chart signature (PGP - alternative)
        run: |
          helm pull oci://$REGISTRY/charts/myapp \
            --version ${{ inputs.chart_version }}
          gpg --import $TRUSTED_GPG_PUBKEY
          helm verify myapp-${{ inputs.chart_version }}.tgz

      - name: Deploy chart
        run: |
          helm registry login $REGISTRY -u $USER -p $PASSWORD
          helm upgrade --install myapp-${{ inputs.environment }} \
            oci://$REGISTRY/charts/myapp \
            --version ${{ inputs.chart_version }} \
            -f env/values-${{ inputs.environment }}.yaml \
            --atomic --wait --timeout 10m
```

**3. Key management (critical):**

```bash
# Generate keys (store securely in Vault/KMS, not in CI variables)
cosign generate-key-pair kms://gcpkms://projects/myproject/locations/global/keyRings/helm/cryptoKeys/signing

# In CI, keys are accessed via OIDC/Workload Identity Federation
# AWS: IRSA
# GCP: Workload Identity Federation
# Azure: Workload Identity Federation

# Key rotation
cosign generate-key-pair  # create new key
# Update trusted key list in verification pipeline
# Sign all chart versions with both old and new keys during rotation period
```

**4. Policy enforcement:**

```rego
# OPA policy — reject unsigned charts
package helm.verify

deny[msg] {
  input.request.operation == "CREATE"
  input.request.kind.kind == "Secret"
  input.request.object.metadata.labels.owner == "helm"
  not has_valid_signature(input.request.object)
  msg := "Chart must be signed with a trusted key"
}
```

**5. Trusted key distribution:**

```
- Store public keys in a well-known location (Git, internal KMS, Vault).
- CI/CD pipeline fetches trusted keys at verification time.
- Key rotation: publish new public key; grace period before old key is removed.
- Key revocation: ability to revoke a compromised key.
```

**Key points to mention:**
- Sign on publish, verify on deploy — mandatory in the pipeline.
- OCI: cosign/notation; Traditional: PGP/GPG.
- Key management via KMS/Vault, not CI environment variables.
- OPA/Gatekeeper policy to reject unsigned charts at the admission level.
- Key rotation with grace period for both old and new keys.
- Verification MUST occur before deployment — never skip.

---

### Q97. How do you handle Helm when deploying to edge/far-edge environments?

**What the interviewer is looking for:** Understanding of constraints in edge computing — low bandwidth, intermittent connectivity, resource constraints.

**Model answer:**

Edge environments (IoT gateways, retail stores, factories) present unique challenges: limited bandwidth, intermittent connectivity, constrained compute/memory, and potentially no direct internet access.

**Challenges and solutions:**

**1. Chart distribution to edge:**

- **Pre-cached charts:** Package all needed charts and dependencies into a single bundle distributed to edge nodes via USB, satellite, or pre-loaded images.
- **Local OCI registry:** Run a lightweight OCI registry (e.g., distribution/distribution) on the edge cluster. Sync charts from central registry during connectivity windows.
- **Chart as part of the OS image:** Embed commonly used charts in the edge node's base image.

```bash
# Bundle all charts into a tarball for edge distribution
helm dependency update ./mychart
tar -czf edge-bundle.tar.gz ./mychart/ ./charts/

# On the edge node
tar -xzf edge-bundle.tar.gz
helm install my-release ./mychart --values edge-values.yaml
```

**2. Values management for heterogeneous edge:**

```yaml
# Edge values are stored locally on the device
# config/edge-values.yaml (generated at device provisioning time)
edge:
  siteId: "store-42-nyc"
  region: us-east
  resources:
    limits:
      cpu: "500m"
      memory: "512Mi"
storageClass: local-path  # Edge-specific storage
```

**3. Reduced resource footprint:**

```yaml
# values-edge.yaml — minimal configuration
replicaCount: 1                # No HA on edge
resources:
  limits:
    cpu: "200m"
    memory: "256Mi"
persistence:
  enabled: false               # Stateless where possible
monitoring:
  enabled: false               # Offload monitoring to central
```

**4. Handling intermittent connectivity:**

- **Self-contained releases:** Each edge deployment is fully self-contained — no external dependencies (databases, APIs) that require constant connectivity.
- **Local Helm operation:** Helm binary on the edge device manages releases locally. No `helm repo update` — charts are pre-loaded.
- **Offline-first:** Deployments work completely offline. Syncs happen when connectivity is available.

**5. Fleet management:**

```yaml
# Fleet-wide Helm upgrade via GitOps
# Central Git repository defines desired state per site
gitops-repo/
├── sites/
│   ├── store-001/
│   │   ├── values.yaml        # Site-specific values
│   │   └── helmfile.yaml      # Which charts, which versions
│   ├── store-002/
│   │   └── ...
│   └── store-NNN/
```

Use a lightweight GitOps agent (Flux, Fleet, or custom) at each edge site that pulls desired state when connected and reconciles locally.

**6. Rollback and recovery:**

- `--history-max 5` — keep minimal history on resource-constrained edge.
- Scheduled snapshots of release Secrets for disaster recovery.
- Ability to reset to factory state by redeploying the chart bundle.

**7. Observability:**

```yaml
# Push metrics to central monitoring (when connected)
# or buffer locally and forward when connectivity returns
```

**Key points to mention:**
- Pre-cache all charts and dependencies; distribute as bundles.
- Local OCI registry or embedded charts (no remote dependency).
- Minimal configuration for edge resource constraints.
- Offline-first operation; sync when connected.
- Fleet management via GitOps with per-site values files.
- Limited revision history (`--history-max 5`).
- Embedded charts in device base image for factory reset capability.

---

### Q98. What are your strategies for managing Helm chart versioning at scale?

**What the interviewer is looking for:** Version management strategy across dozens of charts, teams, and environments.

**Model answer:**

At scale, with 50+ charts across multiple teams, versioning strategy must balance independence, traceability, and consistency.

**1. Semantic versioning for charts:**

```
CHART VERSION: MAJOR.MINOR.PATCH

MAJOR — Breaking changes (removed/renamed values, template restructuring, dependency major version bump).
MINOR — New features (new templates, new values, new dependency, backwards-compatible).
PATCH — Bug fixes (template bug fix, default value correction, documentation update).
```

Enforce via CI:
```bash
# Check that version is bumped appropriately
helm lint . --strict  # Fails if chart version not bumped
```

**2. Versioning strategies:**

**Strategy A: Independent versioning (recommended):**
```
myapp-chart: 2.1.0, 2.1.1, 2.1.2...
db-chart:    1.5.0, 1.5.1...
cache-chart: 3.0.0, 3.0.1...
```
Each chart has its own version. Changes to one chart don't force version bumps in others.
- Pros: Independence, minimal version noise.
- Cons: Harder to track which versions work together.

**Strategy B: Coordinated versioning (for umbrella charts):**
```
platform-umbrella: 4.2.0
  ├── myapp-chart: 2.1.0
  ├── db-chart: 1.5.0
  └── cache-chart: 3.0.0
```
One version for the entire platform. All subcharts are pinned at specific versions.
- Pros: Single version to reason about, guaranteed compatibility.
- Cons: Coupling — a db-chart patch needs a new umbrella version.

**3. Dependency version pinning:**

```yaml
# Chart.yaml — pin exact versions, not ranges
dependencies:
  - name: postgresql
    version: "12.1.0"     # Exact version, not ">=12.0.0"
    repository: "https://charts.bitnami.com/bitnami"

# Chart.lock MUST be committed to Git
# Ensures reproducible builds across all environments
```

**4. Automated version bumping:**

```bash
# CI: auto-bump patch version on every merge to main
# CI: require manual MAJOR/MINOR bump for breaking changes

# Auto-bump using semver tool
VERSION=$(cat Chart.yaml | grep "^version:" | awk '{print $2}')
NEW_VERSION=$(semver bump patch $VERSION)
sed -i "s/^version:.*/version: $NEW_VERSION/" Chart.yaml
```

**5. Version traceability:**

```yaml
# Chart.yaml annotations
annotations:
  git.commit: "abc123def456"
  git.branch: "main"
  ci.pipeline: "https://ci.example.com/builds/12345"
  build.timestamp: "2026-07-11T14:30:00Z"
```

Every chart version knows its Git commit, CI pipeline, and build time.

**6. Release notes automation:**

```bash
# Auto-generate release notes from Git history
git log $(git describe --tags --abbrev=0)..HEAD --oneline > RELEASE_NOTES.md

# Add to OCI artifact annotations
oras annotate oci://registry.example.com/myapp:v1.2.3 \
  --annotation "org.opencontainers.image.description=$(cat RELEASE_NOTES.md)"
```

**7. Version compatibility matrix:**

Maintain a matrix showing which chart versions are compatible with which Kubernetes versions and which dependency versions:

```
| Chart Version | Min K8s Version | PostgreSQL | Redis |
|---------------|-----------------|------------|-------|
| myapp 2.1.0   | 1.28            | 12.x       | 7.x   |
| myapp 2.0.0   | 1.27            | 12.x       | 7.x   |
| myapp 1.5.0   | 1.25            | 11.x       | 6.x   |
```

**Key points to mention:**
- SemVer for charts (MAJOR.MINOR.PATCH).
- Independent versioning for individual charts; coordinated for umbrella.
- Pin exact dependency versions in `Chart.yaml`; commit `Chart.lock`.
- Automated patch version bumping in CI.
- Git commit and CI pipeline annotations for traceability.
- Auto-generated release notes.
- Version compatibility matrix for supported platforms.

---

### Q99. How do you implement Helm with Policy as Code (OPA/Gatekeeper)?

**What the interviewer is looking for:** Understanding of enforcing organizational policies on Helm deployments.

**Model answer:**

Policy as Code ensures that Helm charts and their rendered resources comply with organizational security, compliance, and operational standards. OPA/Gatekeeper is the most common implementation.

**1. Chart-level policies (before deployment):**

```bash
# OPA Conftest — validate rendered manifests before applying
helm template my-release ./mychart -f values-prod.yaml > rendered.yaml

conftest test rendered.yaml --policy policies/
```

**2. Cluster-level policies (Gatekeeper):**

Gatekeeper enforces policies at admission time — every `kubectl apply` or Helm operation is validated.

```yaml
# Require all Helm-managed resources to have specific labels
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_].key}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
```

```yaml
# Enforce the constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: helm-require-standard-labels
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod", "Service", "ConfigMap", "Secret"]
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    labelSelector:
      matchLabels:
        app.kubernetes.io/managed-by: Helm
  parameters:
    labels:
      - key: app.kubernetes.io/name
      - key: app.kubernetes.io/instance
      - key: app.kubernetes.io/managed-by
```

**3. Common policies for Helm:**

```rego
# No privileged containers in Helm releases
package helm.security

violation[{"msg": msg}] {
  input.review.object.metadata.labels["app.kubernetes.io/managed-by"] == "Helm"
  container := input.review.object.spec.templates.spec.containers[_]
  container.securityContext.privileged == true
  msg := "Privileged containers are not allowed in Helm releases"
}

# Require resource limits on all Helm-managed Pods
violation[{"msg": msg}] {
  input.review.object.metadata.labels["app.kubernetes.io/managed-by"] == "Helm"
  container := input.review.object.spec.template.spec.containers[_]
  not container.resources.limits.cpu
  msg := "CPU limits must be set for all containers"
}

# Block deployments to production namespace without approval annotation
violation[{"msg": msg}] {
  input.review.object.metadata.namespace == "prod"
  input.review.object.metadata.labels["app.kubernetes.io/managed-by"] == "Helm"
  not input.review.object.metadata.annotations["deployment.example.com/approved"]
  msg := "Production deployments require approval annotation"
}

# Block use of deprecated API versions
violation[{"msg": msg}] {
  deprecated_apis := {"extensions/v1beta1", "apps/v1beta1", "apps/v1beta2"}
  input.review.object.apiVersion == deprecated_apis[_]
  msg := sprintf("API version %s is deprecated", [input.review.object.apiVersion])
}
```

**4. Pre-deployment validation in CI:**

```bash
# In CI pipeline — validate rendered manifests before deploying
helm template my-release ./mychart -f values-prod.yaml > /tmp/manifest.yaml

# Run Conftest/OPA against rendered manifests
conftest test /tmp/manifest.yaml --policy policies/

# If policies pass, proceed with deployment
helm upgrade --install my-release ./mychart -f values-prod.yaml --atomic --wait
```

**5. Helm-specific policies:**

```rego
# Prevent helm install with --set for sensitive values
# (forces use of external secret stores)
violation[{"msg": msg}] {
  input.review.kind.kind == "Secret"
  input.review.object.type == "helm.sh/release.v1"
  config := json.unmarshal(base64.decode(input.review.object.data.release))
  # Check that no plain-text passwords are in the release config
  contains(config, "password")
  msg := "Secrets detected in Helm release values — use external secret store"
}
```

**6. Enforcement modes:**

- **Dry-run (warn):** Gatekeeper `enforcementAction: dryrun` — violations are logged but not blocked.
- **Deny:** `enforcementAction: deny` — violations block the operation.
- **Gradual rollout:** Start with dry-run, review violations, then switch to deny.

**Key points to mention:**
- Pre-deployment: Conftest/OPA on rendered manifests in CI.
- Admission time: Gatekeeper constraints on Helm-created resources.
- Common policies: required labels, no privileged containers, resource limits, deprecated API versions.
- Helm release Secret inspection for sensitive value detection.
- Gradual rollout: dry-run -> deny after auditing existing violations.
- Label selector `app.kubernetes.io/managed-by: Helm` targets only Helm-managed resources.

---

### Q100. If you were to redesign Helm from scratch, what would you change?

**What the interviewer is looking for:** Critical thinking, deep understanding of Helm's strengths and weaknesses, and vision for improvement.

**Model answer:**

This question tests whether you truly understand Helm's architecture well enough to critique it. The answer should acknowledge what Helm got right while identifying genuine pain points.

**What Helm got right (should keep):**

1. **Client-only architecture (Helm 3):** Removing Tiller was the right call. Keep it client-only.
2. **Release concept:** Chart + Values + Namespace = Release. This abstraction is powerful.
3. **Revision-based rollback:** Storing full manifests per revision enables deterministic rollback.
4. **OCI support:** Moving to OCI registries aligns with the broader container ecosystem.
5. **Template engine (Go templates + Sprig):** Sufficiently powerful for 90% of use cases.

**What I would change:**

**1. Template engine — adopt CUE or a typed configuration language:**

Go templates with Sprig are Turing-complete but error-prone. String-based templating leads to YAML indentation bugs, type confusion, and poor IDE support.

**Alternative:** Use CUE (or a similar typed configuration language) that:
- Provides type checking at authoring time.
- Eliminates YAML indentation issues.
- Has native IDE support (VS Code LSP).
- Supports constraints and validation natively (replaces `values.schema.json`).

```cue
// CUE-based chart definition (conceptual)
package myapp

replicas: int | *1
image: {
    repository: string
    tag:        string | *"latest"
}
```

**2. Values management — native secret integration:**

Helm's values system has no built-in secret handling. Every organization invents their own solution (SOPS, ESO, Vault, Sealed Secrets). This should be a first-class feature.

**Proposal:**
```yaml
# Native secret reference in values.yaml
database:
  password:
    secretRef: arn:aws:secretsmanager:us-east-1:123456:secret:db-password
    key: password
```
Helm resolves secret references at render time from a pluggable secret backend.

**3. Declarative state management — merge vs. apply:**

Helm's three-way strategic merge is powerful but complex and poorly understood. It also creates confusion with `kubectl apply`.

**Proposal:** Adopt the Kubernetes Server-Side Apply (SSA) model. Helm becomes a "field manager" — it owns specific fields in resources. The API server handles merge conflicts. This eliminates the need for Helm's custom merge logic and aligns with Kubernetes native patterns.

**4. Release storage — not in Secrets/ConfigMaps:**

Storing release state in Kubernetes Secrets creates a circular dependency: Helm needs a cluster to manage Helm state. This complicates disaster recovery and multi-cluster management.

**Proposal:** External state store as a first-class option:
```yaml
# helm-config.yaml
storage:
  backend: postgresql
  connection: "postgresql://..."
# or
storage:
  backend: s3
  bucket: helm-releases
  region: us-east-1
```
Secrets remain the default for simplicity, but external stores are fully supported.

**5. Chart dependencies — adopt a lockfile-first model:**

Helm's dependency system (`Chart.yaml` + `Chart.lock`) has confused semantics around `helm dependency update` vs `helm dependency build`.

**Proposal:** Like npm/pip/Cargo — the lock file is the source of truth. `Chart.yaml` declares semantic version constraints. `helm dependency install` (or `helm install`) automatically resolves and locks. No separate `update`/`build` commands. The lock file is always respected unless explicitly updated.

**6. CRDs — full lifecycle management with safeguards:**

The `crds/` directory's "install-only, never update, never delete" behavior is too conservative. It forces organizations to manage CRDs outside of Helm, which creates version skew.

**Proposal:**
- CRDs in `crds/` get full lifecycle management (upgrade, rollback, delete).
- Helm adds a safety check: "This operation will delete/modify a CRD and all its instances. Confirm? [y/N]".
- Option to mark CRDs as "managed" or "unmanaged" per chart.

**7. Better developer experience:**

- **Native VS Code extension** with autocomplete for `.Values`, jump-to-definition for named templates, and inline rendering preview.
- **`helm debug` command** that creates a temporary namespace, installs the chart, and provides an interactive debugging session.
- **Built-in diff** — `helm diff` should be a built-in command, not a plugin.

**Key points to mention:**
- Keep: client-only, releases, revisions, OCI support.
- Change: typed configuration language (CUE), native secret integration, Server-Side Apply.
- Change: external state stores, lockfile-first dependencies, CRD lifecycle management.
- Change: better DX — IDE support, built-in diff, debug command.
- The answer should show you understand WHY each change matters, not just WHAT you'd change.

