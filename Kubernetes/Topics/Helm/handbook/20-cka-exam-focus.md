# Chapter 20: CKA Exam Focus

## Table of Contents

1. [What the CKA Exam Expects for Helm](#what-the-cka-exam-expects-for-helm)
2. [Most Frequently Used Commands (Ranked)](#most-frequently-used-commands-ranked)
3. [Commands You Must Memorize](#commands-you-must-memorize)
4. [Critical Flags to Memorize](#critical-flags-to-memorize)
5. [Typical Helm Tasks on CKA](#typical-helm-tasks-on-cka)
6. [Time-Saving Techniques](#time-saving-techniques)
7. [Exam Workflow for a Typical Helm Task](#exam-workflow-for-a-typical-helm-task)
8. [Fast Debugging Workflow](#fast-debugging-workflow)
9. [Common Pitfalls on CKA](#common-pitfalls-on-cka)
10. [How to Avoid Wasting Time](#how-to-avoid-wasting-time)
11. [Keyboard Shortcuts for the Exam Terminal](#keyboard-shortcuts-for-the-exam-terminal)
12. [Mental Checklist Before Every Helm Command](#mental-checklist-before-every-helm-command)

---

## What the CKA Exam Expects for Helm

The CKA exam includes Helm tasks that test your ability to package, deploy, and manage applications in Kubernetes. The Helm section is weighted as part of the "Service and Networking" or "Workloads" domain (approximately 5-10% of the exam, but Helm knowledge is instrumental across scenarios).

### Core Competencies Tested

| Competency | What You'll Do |
|---|---|
| **Install a chart** | Pull a chart from a repository (Bitnami, etc.) and install it with custom values |
| **Upgrade a release** | Change a value (`replicaCount`, `image.tag`, etc.) and upgrade a running release |
| **Roll back** | Identify the target revision from `helm history` and roll back |
| **List releases** | Find all releases in a namespace or across all namespaces |
| **Inspect releases** | Retrieve values, manifests, and status of a release |
| **Search for charts** | Find a chart in a repository by keyword |
| **Add and update repos** | Add a Helm repository and fetch its latest index |
| **Uninstall releases** | Cleanly remove a release and all its resources |

### What the CKA Does NOT Test

- Writing complex Helm templates or library charts
- Building charts from scratch (that's more CKAD)
- Helm hooks, dependencies, or advanced `values.schema.json`
- Plugin usage (helm-secrets, helm-diff, etc.)
- Chart repository hosting or `index.yaml` management

**Exam Tip:** The CKA exam is about *using* Helm to deploy applications—not about *authoring* charts. Focus on `install`, `upgrade`, `rollback`, `list`, `history`, and `get` commands. You won't need to write a `Chart.yaml` or `values.yaml` from scratch, but you may need to pass `--set` values or reference a `values.yaml` file given to you.

---

## Most Frequently Used Commands (Ranked)

Based on exam task frequency, weighted by likelihood of appearance:

| Rank | Command | Purpose | Estimated Frequency |
|---|---|---|---|
| 1 | `helm install` | Deploy a chart as a new release | Nearly every Helm task |
| 2 | `helm upgrade` | Modify a running release (values, version, config) | Very common |
| 3 | `helm rollback` | Revert a release to a previous revision | Common |
| 4 | `helm list` | Show all releases in a namespace or all namespaces | Nearly every Helm task |
| 5 | `helm history` | Show revision history (needed before rollback) | Common |
| 6 | `helm get values` | Retrieve the values used by a release | Occasional |
| 7 | `helm get manifest` | Retrieve the full Kubernetes YAML rendered for a release | Occasional |
| 8 | `helm repo add` | Add a chart repository | Every Helm task that uses external charts |
| 9 | `helm repo update` | Refresh local repository cache | Every Helm task that uses external charts |
| 10 | `helm search repo` | Find a chart in repositories | Common for "search and install" tasks |
| 11 | `helm uninstall` | Remove a release | Occasional cleanup tasks |
| 12 | `helm status` | Show the status of a release | Occasional verification tasks |
| 13 | `helm template` | Render chart templates locally (debugging) | Rare (more CKAD) |
| 14 | `helm show values` | Show the values file of a chart | Occasional (checking defaults) |
| 15 | `helm show chart` | Show the chart definition | Rare |
| 16 | `helm pull` | Download a chart to local directory | Rare |
| 17 | `helm dependency update` | Download chart dependencies | Rare (more CKAD) |
| 18 | `helm test` | Run chart tests | Rare |
| 19 | `helm lint` | Validate chart structure | Rare (more CKAD) |
| 20 | `helm verify` | Verify a signed chart | Very rare |
| 21 | `helm package` | Package a chart into .tgz | Very rare (more CKAD) |
| 22 | `helm repo list` | List configured repositories | Occasional |
| 23 | `helm env` | Show Helm environment variables | Very rare |
| 24 | `helm plugin list` | List installed plugins | Very rare |
| 25 | `helm version` | Show Helm version | Rare |

**Exam Tip:** Focus your memorization on ranks 1-10. If you can execute these 10 commands fluently without looking them up, you'll handle 95% of CKA Helm tasks.

---

## Commands You Must Memorize

These commands form the foundation of every Helm task on the exam. Drill them until they're muscle memory:

### 1. `helm install`

```bash
helm install RELEASE CHART -n NAMESPACE --set KEY=VALUE
```

**Variations:**
```bash
# Basic install from a repository
helm install my-nginx bitnami/nginx -n web

# Install with multiple --set values
helm install my-nginx bitnami/nginx -n web \
  --set replicaCount=3 \
  --set service.type=NodePort

# Install with a values file
helm install my-nginx bitnami/nginx -n web -f custom-values.yaml

# Install and auto-create namespace
helm install my-nginx bitnami/nginx -n web --create-namespace

# Install from local chart directory
helm install my-app ./chart -n app-ns
```

### 2. `helm upgrade`

```bash
helm upgrade RELEASE CHART -n NAMESPACE --set KEY=VALUE
```

**Variations:**
```bash
# Upgrade changing a value
helm upgrade my-nginx bitnami/nginx -n web --set replicaCount=5

# Upgrade using a values file
helm upgrade my-nginx bitnami/nginx -n web -f new-values.yaml

# Upgrade or install if not exists (idempotent!)
helm upgrade --install my-nginx bitnami/nginx -n web --set replicaCount=3

# Upgrade with atomic rollback on failure
helm upgrade my-nginx bitnami/nginx -n web --atomic --wait --timeout 5m

# Upgrade and reuse all previous values (dangerous—use sparingly)
helm upgrade my-nginx bitnami/nginx -n web --reuse-values
```

**Note:** The `--set` values in `helm upgrade` **override** existing values—they are not merged additively for lists and maps. To add a list item, you must specify the entire list again.

### 3. `helm rollback`

```bash
helm rollback RELEASE REVISION -n NAMESPACE
```

**Variations:**
```bash
# Rollback to a specific revision
helm rollback my-nginx 3 -n web

# Rollback with wait
helm rollback my-nginx 3 -n web --wait

# Dry-run the rollback first
helm rollback my-nginx 3 -n web --dry-run

# Rollback to the previous revision (equivalent to REVISION = current-1)
# No shorthand exists; you must get the revision from helm history first
```

**Exam Tip:** You MUST know the revision number before rolling back. Run `helm history` first, find the revision you want, then `helm rollback`. A rollback creates a **new revision**—it doesn't delete the bad one.

### 4. `helm list`

```bash
helm list -n NAMESPACE
```

**Variations:**
```bash
# All releases in a namespace
helm list -n web

# All releases across all namespaces
helm list -A

# Only failed releases
helm list -n web --failed

# Only pending releases
helm list -n web --pending

# Only deployed releases (default)
helm list -n web --deployed

# All releases (any status)
helm list -n web --all

# Output in JSON for scripting
helm list -n web -o json
helm list -A -o yaml

# Filter by name
helm list -n web -q  # Quiet: names only
helm list -n web -f 'my-nginx'  # Filter
```

### 5. `helm history`

```bash
helm history RELEASE -n NAMESPACE
```

**Variations:**
```bash
# Full history
helm history my-nginx -n web

# Only the last N revisions
helm history my-nginx -n web --max 5

# Output in YAML for scripting
helm history my-nginx -n web -o yaml

# JSON output (parse with jq)
helm history my-nginx -n web -o json | jq '.[] | {revision, status, chart}'
```

### 6. `helm uninstall`

```bash
helm uninstall RELEASE -n NAMESPACE
```

**Variations:**
```bash
# Basic uninstall
helm uninstall my-nginx -n web

# Keep history record (allows re-install with same name + different values)
helm uninstall my-nginx -n web --keep-history

# Dry-run (shows what will be deleted)
helm uninstall my-nginx -n web --dry-run

# No confirmation prompt (useful in scripts)
# Helm 3 uninstall does not prompt by default; this flag was for Helm 2
```

### 7. `helm repo add`

```bash
helm repo add NAME URL
```

**Variations:**
```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Add with username and password (private repos)
helm repo add my-private https://charts.example.com \
  --username myuser --password mypass

# Add without updating the index immediately
helm repo add bitnami https://charts.bitnami.com/bitnami --no-update
```

### 8. `helm repo update`

```bash
helm repo update
```

**Note:** This command updates ALL configured repositories. You cannot update a single repo. Always run this after `helm repo add` and before `helm search repo` or `helm install` to ensure you have the latest index.

### 9. `helm search repo`

```bash
helm search repo KEYWORD
```

**Variations:**
```bash
# Search by name
helm search repo nginx

# Search across all repos
helm search repo database

# Show all versions (not just latest)
helm search repo nginx --versions

# Limit output to specific columns
helm search repo nginx -o json | jq '.[] | {name, version}'

# Search with regex
helm search repo '^bitnami/mysql$'

# Search Hub (internet-wide; slow, not recommended on CKA)
helm search hub nginx  # Avoid this on the exam!
```

**Exam Tip:** Always use `helm search repo`, not `helm search hub`. The Hub searches the internet and is slow; the CKA exam environment may not have internet access. The repo search only searches locally configured repositories and is instant.

---

## Critical Flags to Memorize

### The Non-Negotiable Flags

| Flag | Short Form | Purpose | Priority |
|---|---|---|---|
| `--namespace` | `-n` | Specify the target namespace | **ALWAYS use it!** |
| `--set` | (none) | Set a value inline | Common |
| `--values` | `-f` | Load values from a YAML file | Common |
| `--dry-run` | (none) | Simulate, don't apply | Use before every install/upgrade |
| `--debug` | (none) | Verbose output including rendered templates | Combine with --dry-run |
| `--atomic` | (none) | Rollback on failure | Production installs/upgrades |
| `--wait` | (none) | Block until resources are ready | Production installs/upgrades |
| `--timeout` | (none) | Max wait time for --wait | Always set with --wait |

### Flag Details

#### `-n / --namespace`

```bash
# ALWAYS specify the namespace. ALWAYS.
helm install my-nginx bitnami/nginx -n prod        # ✅
helm install my-nginx bitnami/nginx                 # ❌ deploys to 'default' namespace
```

**Warning:** On the CKA exam, the task will explicitly state the namespace. If you forget `-n`, your release goes to the `default` namespace and you will lose points even if everything else is correct. Make `-n` a reflexive habit.

#### `--set`

```bash
# Set simple values
--set replicaCount=3
--set image.tag=v1.2.3
--set service.type=NodePort

# Set nested values (dot notation)
--set resources.limits.cpu=500m
--set persistence.enabled=true
--set ingress.hosts[0].host=myapp.example.com

# Set multiple values in one flag (comma-separated)
--set replicaCount=3,image.tag=v1.2.3,service.type=ClusterIP
```

**Note:** `--set` interprets values as YAML. For strings that might be interpreted as other types, use `--set-string` or quote them:
```bash
--set-string name=false          # Forces string "false" (not boolean false)
--set-string image.tag=12345     # Forces string "12345" (not integer 12345)
--set name='false'               # Alternative: single quotes prevent shell/YAML interpretation
```

#### `-f / --values`

```bash
# Load one values file
helm install my-app ./chart -f custom.yaml -n app-ns

# Load multiple values files (last one wins)
helm install my-app ./chart -f base.yaml -f prod.yaml -n app-ns

# Combine with --set (--set overrides -f)
helm install my-app ./chart -f base.yaml --set replicaCount=5 -n app-ns
```

#### `--dry-run`

```bash
# Always dry-run before applying
helm install my-app ./chart -n prod --dry-run

# Dry-run with debug shows rendered templates
helm install my-app ./chart -n prod --dry-run --debug

# Dry-run an upgrade
helm upgrade my-app ./chart -n prod --dry-run --debug
```

**Exam Tip:** `--dry-run` does NOT require a running cluster (it's a client-side render). This is extremely useful on the exam when you want to verify your YAML will render correctly before applying.

#### `--atomic`

```bash
# Install with automatic rollback on failure
helm install my-app ./chart -n prod --atomic --timeout 5m

# Upgrade with automatic rollback on failure
helm upgrade my-app ./chart -n prod --atomic --timeout 5m
```

`--atomic` combines `--wait` with automatic rollback:
- If the install/upgrade fails or times out, Helm automatically rolls back to the previous release state.
- For `helm install`, if there's no previous release, all created resources are purged.
- For `helm upgrade`, the release is reverted to the previous successful revision.

#### `--wait`

```bash
helm install my-app ./chart -n prod --wait --timeout 5m
```

Without `--wait`, Helm returns immediately after sending the manifests to the Kubernetes API. Pods might still be `Pending` or `ContainerCreating`. With `--wait`, Helm blocks until:
- All Deployments/StatefulSets/DaemonSets are "ready"
- All Jobs have completed
- All PVCs are bound

#### `--timeout`

```bash
# Default timeout is 5 minutes (300s)
helm install my-app ./chart -n prod --wait --timeout 10m

# For charts with large images, increase timeout
helm install my-app ./chart -n prod --wait --timeout 15m
```

---

## Typical Helm Tasks on CKA

### Task 1: Install a Specific Chart with Custom Values

**Scenario:** "Install the Bitnami nginx chart as `web-server` in namespace `frontend` with 3 replicas and a NodePort service."

```bash
# Step 1: Add repo if not already added
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Step 2: Install
helm install web-server bitnami/nginx \
  -n frontend \
  --set replicaCount=3 \
  --set service.type=NodePort
```

### Task 2: Upgrade a Release by Changing a Value

**Scenario:** "The `web-server` release in `frontend` needs to be scaled to 5 replicas."

```bash
helm upgrade web-server bitnami/nginx \
  -n frontend \
  --set replicaCount=5 \
  --reuse-values
```

**Note:** `--reuse-values` keeps all previously set values and only changes what you specify with `--set`. Without `--reuse-values`, any values you previously set with `--set` would be lost and revert to chart defaults. Use `--reuse-values` cautiously—it makes upgrades non-declarative.

**Better approach:** Keep all values in a file:
```bash
# Save current values to a file first
helm get values web-server -n frontend --all > values.yaml

# Edit values.yaml (change replicaCount)
# Then upgrade from the file
helm upgrade web-server bitnami/nginx -n frontend -f values.yaml
```

### Task 3: Rollback to a Previous Revision

**Scenario:** "The latest upgrade to `web-server` caused issues. Rollback to the previous working version."

```bash
# Step 1: Find which revision to roll back to
helm history web-server -n frontend

# Output:
# REVISION  UPDATED                  STATUS     CHART         DESCRIPTION
# 1        Tue Jul  8 10:00:00 2026  superseded nginx-18.0.0 Install complete
# 2        Tue Jul  8 10:05:00 2026  deployed   nginx-18.0.0 Upgrade complete (bad!)
# 3        Tue Jul  8 10:10:00 2026  failed     nginx-18.0.0 Upgrade failed

# Step 2: Rollback to revision 1
helm rollback web-server 1 -n frontend
```

### Task 4: Search for a Chart in a Specific Repository

**Scenario:** "Find the MySQL chart in the Bitnami repository and install it as `my-db` in the `database` namespace."

```bash
# Step 1: Add and update the repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Step 2: Search
helm search repo bitnami/mysql

# Step 3: Install (with required root password)
helm install my-db bitnami/mysql \
  -n database \
  --set auth.rootPassword=examPass123
```

### Task 5: List and Inspect Releases

**Scenario:** "List all Helm releases in the cluster and inspect the values used by the `web-server` release."

```bash
# List all releases across all namespaces
helm list -A

# Inspect the values
helm get values web-server -n frontend

# Include default values (--all)
helm get values web-server -n frontend --all

# Get the rendered manifests
helm get manifest web-server -n frontend

# Get the release notes
helm get notes web-server -n frontend

# Get chart metadata
helm get all web-server -n frontend
```

---

## Time-Saving Techniques

### 1. `helm list -A` for a Bird's-Eye View

```bash
# Quick overview of EVERYTHING
helm list -A

# Without -A, you must check each namespace individually:
helm list -n ns1
helm list -n ns2
helm list -n ns3  # Waste of time!
```

### 2. `helm get values --all` for Complete Value State

```bash
# --all shows computed values (chart defaults + user values)
# Without --all: only shows user-supplied overrides
helm get values my-release -n my-ns --all
```

**Exam Tip:** `helm get values <release> -n <ns>` shows only what YOU provided via `--set` or `-f`. `--all` includes every value, including chart defaults. Memory aid: "`get values` = what I set; `get values --all` = everything."

### 3. `helm history` Before Every Rollback

```bash
# Never guess the revision number
helm history my-release -n my-ns   # Find the right revision first
helm rollback my-release 1 -n my-ns  # Then roll back
```

### 4. `--dry-run --debug` Before Applying

```bash
# Validate before committing
helm upgrade my-release ./chart -n prod --dry-run --debug | head -50

# Check for obvious errors in the rendered output
helm upgrade my-release ./chart -n prod --dry-run --debug 2>&1 | grep -i error
```

### 5. Consistent `-n` Flag Usage

Make `-n` a muscle-memory reflex. Write it immediately after the release name:

```bash
helm install <release> <chart> -n <ns> ...   # Pattern: install, release, chart, -n, namespace
helm upgrade  <release> <chart> -n <ns> ...  # Pattern: upgrade, release, chart, -n, namespace
helm rollback <release> <rev> -n <ns>        # Pattern: rollback, release, revision, -n, namespace
```

### 6. Aliases and Short Forms

```bash
# Helm itself has no built-in aliases, but you can set shell aliases:
alias h='helm'
alias hl='helm list'
alias hla='helm list -A'
alias hi='helm install'
alias hu='helm upgrade'
alias hrb='helm rollback'
alias hh='helm history'
alias hgv='helm get values'
alias hgm='helm get manifest'
alias hra='helm repo add'
alias hru='helm repo update'
alias hsr='helm search repo'
```

**Warning:** If you use aliases on the exam, test them first. Some exam environments reset aliases. It's safer to type the full commands—you'll develop speed with practice.

### 7. Pipe to `jq` for Quick Value Extraction

```bash
# JSON output + jq for precise data
helm list -A -o json | jq -r '.[] | "\(.name)\t\(.namespace)\t\(.status)"'
helm history my-release -n my-ns -o json | jq '.[-1]'  # Latest revision
helm get values my-release -n my-ns -o json | jq '.replicaCount'
```

---

## Exam Workflow for a Typical Helm Task

### Step-by-Step Execution Pattern

```
┌─────────────────────────────────────────────────────────────┐
│ 1. READ the task carefully — NOTE THE NAMESPACE!           │
│    Circle or mentally highlight the namespace requirement. │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. ADD/UPDATE REPOSITORY (if chart is from a repo)         │
│    helm repo add <name> <url>                              │
│    helm repo update                                         │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. SEARCH for the chart                                    │
│    helm search repo <keyword>                               │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. INSTALL / UPGRADE with required values                  │
│    helm install <release> <chart> -n <ns> --set KEY=VALUE  │
│    helm upgrade <release> <chart> -n <ns> --set KEY=VALUE  │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. VERIFY                                                 │
│    helm list -n <ns>          # Is it there?               │
│    helm status <release> -n <ns>  # What state?           │
│    helm get values <release> -n <ns> # Right values?       │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. TEST (if required by the task)                          │
│    helm test <release> -n <ns>                             │
│    kubectl get pods -n <ns>                                │
│    curl / wget to verify connectivity                       │
└─────────────────────────────────────────────────────────────┘
```

### Concrete Example Walkthrough

**Task:** "Install the `wordpress` chart from Bitnami as `my-blog` in the `cms` namespace. Set the WordPress username to `admin` and password to `exam2026`. The service must be of type `NodePort`."

```bash
# Step 1: The namespace is "cms" — note it!
export NS="cms"

# Step 2: Ensure the repo exists
helm repo add bitnami https://charts.bitnami.com/bitnami 2>/dev/null
helm repo update

# Step 3: Search (optional but confirms chart name)
helm search repo bitnami/wordpress

# Step 4: Create namespace + install
kubectl create namespace "$NS" 2>/dev/null
helm install my-blog bitnami/wordpress \
  -n "$NS" \
  --set wordpressUsername=admin \
  --set wordpressPassword=exam2026 \
  --set service.type=NodePort \
  --wait --timeout 10m

# Step 5: Verify
helm list -n "$NS"                     # See the release
helm status my-blog -n "$NS"           # Check deployment status
helm get values my-blog -n "$NS"       # Confirm our values took effect

# Step 6: Quick verification
kubectl get pods -n "$NS"              # All Running?
kubectl get svc -n "$NS"               # NodePort assigned?
```

---

## Fast Debugging Workflow

When a Helm task goes wrong on the exam, follow this systematic debugging path:

```
┌─────────────────────────────────────────────────────────┐
│ DEBUGGING ESCALATION PATH                               │
│                                                         │
│ ① helm list -n NAMESPACE                               │
│    ├─ Release not found? Wrong namespace or name.       │
│    └─ Release found? Note the STATUS.                   │
│                                                         │
│ ② helm status RELEASE -n NAMESPACE                     │
│    ├─ deployed → Everything is fine, check values next. │
│    ├─ failed → Read the error message carefully.        │
│    ├─ pending-upgrade → Something is stuck. Check hooks.│
│    └─ pending-install → Still deploying. Wait or check. │
│                                                         │
│ ③ helm history RELEASE -n NAMESPACE                    │
│    └─ How many revisions? Which ones failed?            │
│                                                         │
│ ④ helm get manifest RELEASE -n NAMESPACE               │
│    └─ What was actually deployed? Look for issues.      │
│                                                         │
│ ⑤ helm get values RELEASE -n NAMESPACE --all           │
│    └─ What values were used? Did --set values stick?    │
│                                                         │
│ ⑥ Fix the issue and helm upgrade                       │
│    OR helm rollback RELEASE REV -n NAMESPACE            │
└─────────────────────────────────────────────────────────┘
```

### Debugging Command Sequence

```bash
# 1. Is the release there?
helm list -n frontend
# Output: my-app  frontend  deployed  1  ...

# 2. What's the status?
helm status my-app -n frontend
# Output: STATUS: failed  → "Error: timed out waiting for condition"
#         STATUS: deployed → Move to step 4

# 3. What happened in history?
helm history my-app -n frontend
# REVISION  STATUS     DESCRIPTION
# 1         superseded Install complete
# 2         deployed   Upgrade complete
# 3         failed     Upgrade "my-app" failed: timed out waiting for condition

# 4. What was deployed? (Check for template errors)
helm get manifest my-app -n frontend | grep -A 5 -B 5 image
# Did the image tag render correctly? Is the image pullable?

# 5. What values are set?
helm get values my-app -n frontend --all | grep -i replica
# replicaCount: 5 (is this what we wanted?)

# 6. Fix and redeploy
helm rollback my-app 2 -n frontend   # Go back to the working revision
# OR
helm upgrade my-app bitnami/nginx -n frontend --set replicaCount=3
```

---

## Common Pitfalls on CKA

### Pitfall 1: Forgetting the Namespace Flag

```
❌ helm install my-nginx bitnami/nginx
   → Installs to 'default' namespace. Task expects 'web' namespace.
   → 0 points for this task.

✅ helm install my-nginx bitnami/nginx -n web
```

**Prevention:** Read the task's namespace requirement aloud (in your head) before typing. Make `-n <namespace>` the SECOND thing you type, right after the release name.

### Pitfall 2: Typing the Release Name Wrong

```
❌ helm install nginx-web-server bitnami/nginx -n web     # Installed
   helm list -n web                                        # "nginx-web-server"
   helm history my-nginx -n web      # "Error: release: not found"
   → Task says "Install as 'my-nginx'", but you installed as 'nginx-web-server'
   → Wasted time searching for the wrong name

✅ Install with the EXACT name from the task. Double-check before pressing Enter.
```

### Pitfall 3: Forgetting `helm repo add` Before `helm install`

```
❌ helm install my-nginx bitnami/nginx -n web
   → Error: failed to download "bitnami/nginx"
   → You assumed the repo was already configured. It wasn't.

✅ helm repo add bitnami https://charts.bitnami.com/bitnami
   helm repo update
   helm install my-nginx bitnami/nginx -n web
```

**Prevention:** Always run `helm repo list` first to see what's configured. Or just add the repo anyway—`helm repo add` will fail gracefully if the repo already exists (Helm 3.7+).

### Pitfall 4: Missing Required `--set` Values

```
❌ helm install my-db bitnami/mysql -n database
   → Error: You must specify a root password...
   → Some charts require mandatory values.

✅ helm install my-db bitnami/mysql -n database \
     --set auth.rootPassword=exam123
```

**Prevention:** Run `helm show values <chart>` before installing to see what values are required. Look for `null` values or comments indicating mandatory fields.

```bash
helm show values bitnami/mysql | grep -A 2 rootPassword
# auth:
#   rootPassword: ""   ← empty string means mandatory!
```

### Pitfall 5: Confusing the Revision Number

```
❌ helm history my-app -n web
   REVISION  STATUS     DESCRIPTION
   1         superseded Install complete
   2         superseded Upgrade complete
   3         deployed   Upgrade complete

   helm rollback my-app 3 -n web
   → Error: release "my-app" has no revision 3 that is not the current revision
   → You can't roll back to the CURRENT revision

✅ helm rollback my-app 2 -n web   # Roll back to revision 2
```

**Note:** You cannot roll back to the currently deployed revision. If revision 3 is `deployed`, you must target revision 1 or 2.

### Pitfall 6: Not Using `--dry-run` Before Applying

```
❌ helm upgrade my-app bitnami/nginx -n web --set replicaCount=notANumber
   → Error: failed to render template
   → Now your release might be in a broken state because Helm started the upgrade

✅ helm upgrade my-app bitnami/nginx -n web --set replicaCount=notANumber --dry-run
   → Error caught BEFORE touching the cluster
```

### Pitfall 7: Using `helm install` Instead of `helm upgrade --install`

```
❌ helm install my-app bitnami/nginx -n web
   → Works the first time.
   → Run again: "Error: cannot re-use a name..."

✅ helm upgrade --install my-app bitnami/nginx -n web
   → Installs if not present, upgrades if already installed.
   → Idempotent command—safe to run in scripts and CI.
```

### Pitfall 8: Not Saving Values Before `helm upgrade`

```
❌ helm upgrade my-app bitnami/nginx -n web --set replicaCount=10
   → If the previous install used --set service.type=NodePort,
     it gets LOST because --set in upgrade overwrites previous --set values
     (unless you use --reuse-values)

✅ Always capture values before upgrade if you don't have them in a file:
   helm get values my-app -n web --all > current-values.yaml
   # Edit current-values.yaml
   helm upgrade my-app bitnami/nginx -n web -f current-values.yaml
```

---

## How to Avoid Wasting Time

### 1. Use `helm search repo` (Not `helm search hub`)

```bash
# Fast — searches locally cached repositories
helm search repo nginx         # Instant

# Slow — searches Artifact Hub on the internet
helm search hub nginx           # 5-30 seconds, may fail in exam environment
```

### 2. Always Specify Namespace with `-n`

```bash
# Good: Explicit namespace
helm list -n prod                          # Shows only prod releases

# Wasteful: Check every namespace one by one
helm list -n ns1; helm list -n ns2; helm list -n ns3 ...

# Best for overview: Check all namespaces at once
helm list -A                              # Shows everything
```

### 3. Use `--generate-name` Only If the Task Doesn't Specify a Name

```bash
# When the task says "install a release called my-app":
helm install my-app bitnami/nginx -n web    # ✅ Exact name specified by task

# When the task says "install the nginx chart" (no release name):
helm install --generate-name bitnami/nginx -n web  # ✅ Auto-generated name
```

### 4. Use `-o yaml` or `-o json` for Scripting

```bash
# Extract just the status
helm list -n web -o json | jq -r '.[].status'

# Extract revision count
helm history my-app -n web -o json | jq 'length'

# Extract the latest chart version
helm history my-app -n web -o json | jq -r '.[-1].chart'
```

### 5. Combine Commands to Save Keystrokes

```bash
# Instead of:
kubectl create namespace web
helm install my-app bitnami/nginx -n web

# Use:
helm install my-app bitnami/nginx -n web --create-namespace
```

### 6. Use `helm show values` to See What You Can Configure

```bash
# Before installing, check what values are available
helm show values bitnami/nginx | less

# Find mandatory values (null or empty string)
helm show values bitnami/mysql | grep -B 1 '""'
```

---

## Keyboard Shortcuts for the Exam Terminal

The CKA exam uses a web-based terminal. These Bash/Readline shortcuts save significant time:

### Essential Navigation

| Shortcut | Action | When to Use |
|---|---|---|
| `Ctrl + A` | Go to beginning of line | Quickly add `helm install` at the start |
| `Ctrl + E` | Go to end of line | Append `-n namespace` at the end |
| `Ctrl + F` | Forward one character | Fine-grained cursor movement |
| `Ctrl + B` | Backward one character | Fine-grained cursor movement |
| `Alt + F` | Forward one word | Jump past a word quickly |
| `Alt + B` | Backward one word | Jump back to fix a typo |

### Editing and Deletion

| Shortcut | Action | When to Use |
|---|---|---|
| `Ctrl + U` | Delete from cursor to start of line | Start over after noticing a mistake at the start |
| `Ctrl + K` | Delete from cursor to end of line | Clear everything after cursor |
| `Ctrl + W` | Delete word before cursor | Remove the last argument you typed |
| `Alt + D` | Delete word after cursor | Remove the next argument |
| `Ctrl + Y` | Yank (paste) previously deleted text | Recover accidentally deleted text |

### History and Search

| Shortcut | Action | When to Use |
|---|---|---|
| `Ctrl + R` | Reverse search through history | Find a command you ran 5 minutes ago |
| `Ctrl + R` (again) | Cycle to previous match | Find an older match |
| `Ctrl + G` | Cancel search | Escape from history search |
| `Up Arrow` | Previous command | Quick recall of the last command |
| `Down Arrow` | Next command in history | Move forward through history |
| `!!` | Repeat last command | Rerun the last command (e.g., `sudo !!`) |
| `!helm` | Run last command starting with "helm" | Quick re-execute |
| `!$` | Last argument of previous command | Reuse the namespace from last command |

### Practical Exam Shortcut Patterns

```bash
# Pattern 1: Rerun with change
helm install my-app bitnami/nginx -n web --set replicaCount=3
# Up arrow → Ctrl+W → type 5 → Enter  (Changes replicaCount to 5)

# Pattern 2: Fix namespace at end
# You typed: helm install my-app bitnami/nginx
# Then realize you forgot -n web
# Press: Ctrl+E → space → -n web → Enter

# Pattern 3: Find your last helm command
# Press: Ctrl+R → type "helm i" → Enter (finds helm install...)

# Pattern 4: Reuse namespace
helm install my-app bitnami/nginx -n frontend
helm list -n !$   # Expands to "helm list -n frontend"
```

### Exam Terminal-Specific Tips

```
- Copy/Paste: The exam terminal is inside a browser. Your browser's
  copy/paste (Ctrl+C / Ctrl+V) should work, but test it during the
  tutorial period.

- Ctrl+C in the terminal interrupts the running command. To copy,
  use the browser's right-click → Copy, or check if the exam
  environment provides a copy button.

- The terminal may not have mouse support for cursor positioning.
  Learn the keyboard shortcuts above.

- Tab completion works for kubectl but may not work for Helm commands
  (chart names, release names). Don't rely on it.
```

---

## Mental Checklist Before Every Helm Command

Before pressing Enter on ANY `helm install` or `helm upgrade` command, run through this checklist in your head:

```
┌────────────────────────────────────────────────────────────┐
│ ☐ Did I specify -n NAMESPACE?                             │
│                                                            │
│ ☐ Did I run helm repo add AND helm repo update first?    │
│    (If installing a chart from a repository)               │
│                                                            │
│ ☐ Is the release name correct?                            │
│    (Matches the task exactly, meaningful, no typos)        │
│                                                            │
│ ☐ Do I have the right values?                             │
│    (Checked helm show values? Required values set?)       │
│                                                            │
│ ☐ Should I use --dry-run first?                           │
│    (Almost always YES for upgrades in production-like      │
│     scenarios with complex values)                         │
│                                                            │
│ ☐ Do I need --atomic?                                     │
│    (For CI/CD or automated workflows, YES)                 │
│                                                            │
│ ☐ Do I need --wait?                                       │
│    (For verification steps that depend on ready pods, YES) │
│                                                            │
│ ☐ Is the namespace already created?                       │
│    (If not, add --create-namespace)                        │
└────────────────────────────────────────────────────────────┘
```

### Mental Checklist in Short Form

```
1. -n ?         Namespace specified?
2. Repo ?       Repo added and updated?
3. Name ?       Release name correct?
4. Values ?     Required values set?
5. Dry-run ?    Should I dry-run first?
6. Atomic ?     Do I need automatic rollback?
7. Wait ?       Do I need to wait for readiness?
8. Create-ns ?  Does the namespace exist?
```

**Exam Tip:** Write this checklist on your scratch paper during the tutorial period. Glance at it before every Helm command. The 2 seconds it takes to check will save you 2-5 minutes of debugging if you get something wrong.

### The Golden Rule

```
helm install  RELEASE CHART -n NAMESPACE [--set KEY=VALUE] [--wait] [--atomic]
helm upgrade  RELEASE CHART -n NAMESPACE [--set KEY=VALUE] [--wait] [--atomic]
helm rollback RELEASE REV   -n NAMESPACE
helm list     -n NAMESPACE                     # or -A for all
helm history  RELEASE -n NAMESPACE
helm uninstall RELEASE -n NAMESPACE
```

**Memorize these six command skeletons. Everything else builds on them.**
