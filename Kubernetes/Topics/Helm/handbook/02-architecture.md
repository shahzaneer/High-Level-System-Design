# Chapter 2: Helm Architecture

---

## 2.1 High-Level Architecture Overview

```
+-----------------------------------------------------------------------------+
|                       HELM ARCHITECTURE -- DATA FLOW                        |
|                                                                              |
|  +----------+     +---------------+     +---------------+     +------------+ |
|  |  User    |---->|  Helm CLI     |---->|  Template     |---->|  Rendered  | |
|  |          |     |  (Go binary)  |     |  Engine       |     |  YAML      | |
|  +----------+     |               |     |               |     | (manifest) | |
|                   |  +---------+  |     |  +---------+  |     +------+-----+ |
|                   |  | Load    |  |     |  | Go text/|  |            |       |
|                   |  | Chart   |--+-----+->| template|  |            |       |
|                   |  +---------+  |     |  | + Sprig |  |            |       |
|                   |               |     |  +---------+  |            |       |
|                   |  +---------+  |     |               |            |       |
|                   |  | Merge   |  |     |  +---------+  |            |       |
|                   |  | Values  |--+-----+->| Built-in|  |            |       |
|                   |  +---------+  |     |  | Objects |  |            |       |
|                   +------+--------+     +--+------------+            |       |
|                          |                                           |       |
|                          v                                           v       |
|                   +--------------+                          +--------------+ |
|                   |  Kubernetes  |<-------------------------|  Apply       | |
|                   |  API Server  |    HTTP REST + client-go |  via         | |
|                   |              |                          |  kubeconfig  | |
|                   |  +--------+  |                          +--------------+ |
|                   |  |Release |  |                                            |
|                   |  |Secrets |  |  <-- Release history stored as Secrets     |
|                   |  +--------+  |                                            |
|                   |  +--------+  |                                            |
|                   |  |Actual  |  |  <-- Deployments, Services, etc.           |
|                   |  |K8s     |  |                                            |
|                   |  |Objects |  |                                            |
|                   |  +--------+  |                                            |
|                   +--------------+                                            |
+-----------------------------------------------------------------------------+
```

**The complete pipeline in 8 steps:**

```
Step 1: User runs 'helm install my-release ./mychart'
         |
Step 2: Helm CLI reads ~/.kube/config for cluster connection info
         |
Step 3: Helm loads the chart from local disk (or pulls from repo/OCI)
         |
Step 4: Helm merges values (defaults -> file overrides -> CLI overrides)
         |
Step 5: Template engine renders templates/ with merged values + built-in objects
         |
Step 6: Helm generates a multi-document YAML manifest
         |
Step 7: Helm sends the manifest to Kubernetes API via REST calls
         |       (uses client-go library internally)
         |
Step 8: Helm stores the release as a Kubernetes Secret for future tracking
         |
         v
      Kubernetes controllers reconcile desired state -> Pods are created
```

---

## 2.2 Detailed Pipeline -- Step by Step

### Step 1: Chart Loading

```
  helm install my-release ./mychart
       |
       v
  +------------------+
  | Is the chart     |
  | local or remote? |
  +----+---------+---+
       |         |
   LOCAL     REMOTE
       |         |
       v         v
  +---------+  +---------------------------------------+
  | Read    |  | Resolve from:                         |
  | disk    |  |  - Helm repo (HTTP)                   |
  | path    |  |    - Download index.yaml              |
  |         |  |    - Find matching version            |
  | ./myapp/|  |    - Download .tgz                    |
  +----+----+  |  - OCI registry                       |
       |       |    - helm registry login              |
       |       |    - Pull OCI artifact                |
       |       |  - URL (.tgz)                         |
       |       |    - HTTP GET .tgz                    |
       |       +--------------------+------------------+
       |                            |
       +----------+-----------------+
                  |
                  v
  +------------------------------------------------+
  | Load into memory:                              |
  |  Parse Chart.yaml                              |
  |  Load values.yaml                              |
  |  Load values.schema.json (if exists)           |
  |  Read all files in templates/                  |
  |  Read all files in crds/                       |
  |  Load dependencies from charts/                |
  |  Read .helmignore patterns                     |
  +------------------------------------------------+
```

### Step 2: Values Merging

```
  Precedence (lowest -> highest):

  +---------------------+
  | 1. chart/values.yaml|  <-- Chart author's defaults
  +---------+-----------+
            |
            v
  +---------------------+     +----------------------------------+
  | 2. --values file1   |---->| Deep-merge into defaults         |
  |    --values file2   |     | (maps merge, lists replace)       |
  +---------+-----------+     +----------------------------------+
            |
            v
  +---------------------+     +----------------------------------+
  | 3. --set key=val    |---->| Convert dotted keys to nested    |
  |    --set a.b=c      |     | map, deep-merge into result       |
  +---------+-----------+     +----------------------------------+
            |
            v
  +---------------------+     +----------------------------------+
  | 4. --set-string     |---->| Same as --set but forces         |
  |    key=val          |     | string type (no type coercion)    |
  +---------+-----------+     +----------------------------------+
            |
            v
  +-----------------------------------------------------------+
  | FINAL MERGED VALUES MAP (Go map[string]interface{})        |
  |                                                            |
  | This map is passed as .Values to every template execution. |
  +-----------------------------------------------------------+

  Example:
    Default:  {replicas: 1, resources: {cpu: 100m, mem: 128Mi}}
    File:     {replicas: 3, resources: {cpu: 500m}}
    --set:    replicas=5
    Result:   {replicas: 5, resources: {cpu: 500m, mem: 128Mi}}
                 ^                           ^       ^
                 |                           |       +-- From default
                 |                           +-- From file
                 +-- From --set (highest precedence)
```

### Step 3: Template Rendering

```
  For each file in templates/ (excluding _*.tpl partials):

  +----------------------+
  | templates/           |
  | +-- deployment.yaml  |----+
  | +-- service.yaml     |----+
  | +-- ingress.yaml     |----+
  | +-- configmap.yaml   |----+
  | +-- _helpers.tpl     |    (partial -- NOT rendered standalone)
  | +-- NOTES.txt        |----+
  | +-- tests/           |    |
  |     +-- test-con.yaml|----+
  +----------------------+    |
            |                 |
            v                 v
  +----------------------------------------------------------------+
  |                    GO TEMPLATE EXECUTION                        |
  |                                                                 |
  |  Inputs to each template:                                       |
  |                                                                 |
  |  .Values       | .Release        | .Chart         | .Capabilities |
  |  (merged map)  | .Name           | .Name          | .KubeVersion  |
  |                | .Namespace      | .Version       | .APIVersions  |
  |                | .Service        | .AppVersion    |               |
  |                | .IsInstall      | .Type          | .Template     |
  |                | .IsUpgrade      | .Description   | .Name         |
  |                | .Revision       | .ApiVersion    | .BasePath     |
  |                                                           .Files |
  +------------------------------------+---------------------------+
                                       |
                                       v
  +----------------------------------------------------------------+
  |                       RENDERED OUTPUT                          |
  |                                                                 |
  |  ---                                                            |
  |  # Source: mychart/templates/serviceaccount.yaml                |
  |  apiVersion: v1                                                 |
  |  kind: ServiceAccount                                           |
  |  ...                                                            |
  |  ---                                                            |
  |  # Source: mychart/templates/deployment.yaml                    |
  |  apiVersion: apps/v1                                            |
  |  kind: Deployment                                               |
  |  ...                                                            |
  |  ---                                                            |
  |  # Source: mychart/templates/service.yaml                       |
  |  apiVersion: v1                                                 |
  |  kind: Service                                                  |
  |  ...                                                            |
  +----------------------------------------------------------------+
```

### Step 4: Resource Install Ordering

Helm applies resources in a specific order to respect Kubernetes dependencies (e.g., namespaces before resources in that namespace).

| Order | Resource Kind | Reason |
|-------|--------------|--------|
| 1 | Namespace | Must exist before resources in it |
| 2 | NetworkPolicy | Security first |
| 3 | ResourceQuota | Quota before resources that consume it |
| 4 | LimitRange | Limits before pods |
| 5 | PodDisruptionBudget | Availability before workloads |
| 6 | ServiceAccount | Identity before pods that use it |
| 7 | Secret | Credentials before consumers |
| 8 | ConfigMap | Config before consumers |
| 9 | StorageClass | Storage types before claims |
| 10 | PersistentVolume | Volumes before claims |
| 11 | PersistentVolumeClaim | Claims before mounters |
| 12 | CRD | Definitions before instances |
| 13 | ClusterRole | RBAC definitions before bindings |
| 14 | ClusterRoleBinding | Bindings after roles |
| 15 | Role | Namespaced RBAC |
| 16 | RoleBinding | Namespaced bindings |
| 17 | Service | Network before workloads |
| 18 | DaemonSet | Node agents before applications |
| 19 | Pod | Standalone pods |
| 20 | ReplicationController | Legacy controllers |
| 21 | ReplicaSet | Sets before deployments |
| 22 | Deployment | Main workloads |
| 23 | HorizontalPodAutoscaler | Autoscalers after deployments |
| 24 | StatefulSet | Stateful workloads |
| 25 | Job | One-off workloads |
| 26 | CronJob | Scheduled workloads |
| 27 | Ingress | External access last |
| 28 | APIService | Extension APIs |
| 29 | Scale | Scale sub-resources |

**Uninstall order: REVERSE of install order.**

**Production Note:** CRDs in the `crds/` directory are installed before any template rendering and are never updated or deleted by upgrade/rollback. This prevents accidental data loss from CRD modifications.

### Step 5: Kubernetes API Communication

```
+---------------------------------------------------------------------------+
|                     KUBERNETES API COMMUNICATION                           |
|                                                                           |
|  +------------------------------------------------------------------+     |
|  |                        HELM CLI                                   |     |
|  |                                                                   |     |
|  |  kubeconfig: ~/.kube/config                                       |     |
|  |    - clusters          - users                                    |     |
|  |    - contexts          - current-context                          |     |
|  |                                                                   |     |
|  |  action/* packages: install.go, upgrade.go, rollback.go, etc.    |     |
|  +---------------------------+---------------------------------------+     |
|                              |                                            |
|                              v                                            |
|  +------------------------------------------------------------------+     |
|  |                   pkg/kube/ -- The Kube Client                    |     |
|  |                                                                   |     |
|  |  Kubernetes ClientSet (from `client-go`):                         |     |
|  |    REST config: TLS certs, auth token, server URL                  |     |
|  |                                                                   |     |
|  |  Discovery Client:                                                |     |
|  |    GET /api   --> Core API groups (v1)                            |     |
|  |    GET /apis  --> Named API groups (apps/v1, networking/v1, ...)  |     |
|  |                                                                   |     |
|  |  Dynamic Client + RESTMapper:                                     |     |
|  |    Resolves apiVersion+kind --> GVR (GroupVersionResource)        |     |
|  |    Example: apps/v1 Deployment --> /apis/apps/v1/.../deployments  |     |
|  +---------------------------+---------------------------------------+     |
|                              |                                            |
|                              v                                            |
|                    +----------------------+                               |
|                    |  Kubernetes          |                               |
|                    |  API Server          |                               |
|                    |  https://k8s-api:443 |                               |
|                    +----------------------+                               |
+---------------------------------------------------------------------------+
```

**Exam Tip:** Helm uses `client-go`, the official Kubernetes Go library. It creates a `clientset` from kubeconfig, then uses the `dynamic.Interface` client with a `RESTMapper` to work with arbitrary resource types at runtime -- this is how Helm can install any Kubernetes resource without compile-time Go types.

---

## 2.3 How Helm Talks to Kubernetes

### 2.3.1 Kubeconfig Resolution Order

```
1. --kube-context <context>       --> Use specific context
2. --kubeconfig <path>            --> Use specific kubeconfig file
3. KUBECONFIG environment variable --> Use path from env var
4. ~/.kube/config                  --> Default path
5. In-cluster config               --> If running inside a pod (service account)
```

```bash
helm install my-release ./mychart --kubeconfig /path/to/config
helm install my-release ./mychart --kube-context prod-cluster
helm install my-release ./mychart --namespace my-app
helm install my-release ./mychart --kube-apiserver https://k8s.internal:6443
```

### 2.3.2 REST API Calls for `helm install`

| Order | API Call | Purpose |
|-------|----------|---------|
| 1 | `GET /api` | Discover core API group versions |
| 2 | `GET /apis` | Discover named API group versions |
| 3 | `GET /apis/apiextensions.k8s.io/v1/crds` | Check for existing CRDs |
| 4 | `POST /api/v1/namespaces/{ns}/secrets` | Create release Secret (storage) |
| 5 | Apply each manifest resource (in install order) | Create K8s objects |
| 6 | `PATCH ...secrets/sh.helm.release.v1.{name}.v{rev}` | Set status to "deployed" |

### 2.3.3 REST API Calls for `helm upgrade`

| Order | API Call | Purpose |
|-------|----------|---------|
| 1-3 | Same discovery calls | Refresh API resource knowledge |
| 4 | `GET .../secrets/sh.helm.release.v1.{name}.v{n}` | Load previous release |
| 5 | Compute 3-way merge | Determine create/update/delete set |
| 6 | For each resource: GET current, POST/PUT/PATCH/DELETE | Apply the diff |
| 7 | `POST .../secrets` | Store new revision |

### 2.3.4 In-Cluster Configuration

When Helm runs inside a Kubernetes pod (e.g., CI/CD), it uses the pod's service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm-deployer
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: helm-deployer
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: helm-deployer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: helm-deployer
subjects:
  - kind: ServiceAccount
    name: helm-deployer
    namespace: default
```

**Warning:** The ClusterRole above grants full cluster access. In production, scope Helm's RBAC permissions to only the namespaces and resource types it needs. Never give cluster-admin to a CI/CD service account.

---

## 2.4 How Releases Are Stored

### 2.4.1 Secret Naming Convention

```
sh.helm.release.v1.<release-name>.v<revision>
```

Examples:
```
sh.helm.release.v1.my-release.v1
sh.helm.release.v1.my-release.v2
sh.helm.release.v1.my-release.v3
```

### 2.4.2 Revision Storage Architecture

```
Kubernetes Namespace: prod

+-- sh.helm.release.v1.my-release.v1 (type: helm.sh/release.v1)
|     labels:
|       owner: helm
|       name: my-release
|       status: superseded
|       version: "1"
|       modifiedAt: "1705284134"
|       chart: mychart-1.0.0
|     data:
|       release: <base64( gzip({...}) )>
|
+-- sh.helm.release.v1.my-release.v2 (type: helm.sh/release.v1)
|     labels:
|       owner: helm, name: my-release
|       status: superseded, version: "2"
|     data:
|       release: <base64( gzip({...}) )>
|
+-- sh.helm.release.v1.my-release.v3 (type: helm.sh/release.v1)
      labels:
        owner: helm, name: my-release
        status: deployed          <-- CURRENTLY ACTIVE
        version: "3"
      data:
        release: <base64( gzip({...}) )>

Only ONE revision is "deployed" at any time.
Older revisions are "superseded".
Failed installs may be "failed" or "pending-install".
```

### 2.4.3 Release Status Values

| Status | Meaning | When Set |
|--------|---------|----------|
| `unknown` | Initial state | Before any operation completes |
| `deployed` | Currently active revision | After successful install/upgrade/rollback |
| `superseded` | Previous revision | After upgrade or rollback |
| `failed` | Operation failed | After a failed operation |
| `uninstalling` | Uninstall in progress | During `helm uninstall` |
| `pending-install` | Install in progress | During `helm install` |
| `pending-upgrade` | Upgrade in progress | During `helm upgrade` |
| `pending-rollback` | Rollback in progress | During `helm rollback` |

### 2.4.4 The Secret Data Structure

The `release` field in each Secret is base64-encoded gzipped JSON. After decoding:

```json
{
  "name": "my-release",
  "info": {
    "first_deployed": "2026-07-11T10:00:00Z",
    "last_deployed": "2026-07-11T14:30:00Z",
    "description": "Upgrade complete",
    "status": "deployed",
    "notes": "Thank you for installing mychart..."
  },
  "chart": {
    "metadata": {
      "name": "mychart",
      "version": "1.3.0",
      "appVersion": "2.3.0"
    },
    "templates": [
      {"name": "templates/deployment.yaml", "data": "<base64 source>"}
    ],
    "values": {
      "replicaCount": 3,
      "image": {"repository": "myregistry.io/myapp", "tag": "2.3.0"}
    }
  },
  "config": {
    "replicaCount": 3,
    "image": {"repository": "myregistry.io/myapp", "tag": "2.3.0"}
  },
  "manifest": "---\n# Source: ...\napiVersion: apps/v1\nkind: Deployment\n...",
  "version": 3,
  "namespace": "prod"
}
```

**Key insight:** The full manifest, all template source files, and all resolved values are stored in each revision Secret. This is why rollback is always possible and why `helm get values` and `helm get manifest` work for any revision.

Decoding a release Secret manually:

```bash
kubectl get secret sh.helm.release.v1.my-release.v1 \
  -o jsonpath='{.data.release}' | base64 -d | gunzip | jq .
```

### 2.4.5 Storage Driver Interface

Helm abstracts storage through a driver interface:

| Driver | Env Var | Default? | Persistence |
|--------|---------|----------|-------------|
| `secret` | `HELM_DRIVER=secret` | **Yes** | Kubernetes Secrets |
| `configmap` | `HELM_DRIVER=configmap` | No | Kubernetes ConfigMaps |
| `memory` | `HELM_DRIVER=memory` | No | None (testing only) |
| `sql` | `HELM_DRIVER=sql` | No | External SQL database |

```bash
HELM_DRIVER=memory helm install test-release ./mychart
HELM_DRIVER=configmap helm install my-release ./mychart
HELM_DRIVER=sql helm install my-release ./mychart \
  --sql-connection-string "postgresql://user:pass@host:5432/helm?sslmode=require"
```

**Production Note:** Secrets are the default and recommended backend. They are encrypted at rest if Kubernetes encryption is configured. ConfigMaps store data in plaintext. The SQL driver is useful for centralized multi-cluster release state.

---

## 2.5 Revisions -- Deep Dive

### 2.5.1 How Revisions Work

Every Helm operation that modifies cluster state creates a new revision:

```
Time ------------------------------------------------------------->

t0: helm install my-release ./mychart
    --> Revision 1 (status: deployed, manifest: full YAML v1)

t1: helm upgrade my-release ./mychart --set replicas=5
    --> Revision 1 status changed to: superseded
    --> Revision 2 created (status: deployed, manifest: full YAML v2)

t2: helm rollback my-release 1
    --> Revision 2 status changed to: superseded
    --> Revision 3 created (status: deployed, manifest: SAME AS REV 1)

t3: helm uninstall my-release
    --> All Secrets deleted. All resources deleted.
        (Use --keep-history to preserve Secrets)
```

### 2.5.2 Key Properties of Revisions

1. **Monotonic counter:** Revision numbers start at 1 and always increment upward. They never decrease. A rollback creates a higher-numbered revision.

2. **Immutable storage:** Once a revision Secret is created, its data is never modified. Only labels (status) change on old revisions.

3. **Full snapshot:** Each revision contains the complete manifest, all template source code, and all resolved values -- making every revision independently deployable.

4. **Garbage collection:** When new revisions exceed `--history-max` (default 10), the oldest `superseded` revision is deleted. The `deployed` revision is never garbage-collected.

5. **Revival:** You can roll back to any revision. You can also roll forward: `helm rollback my-release 5` (if rev 5 exists).

### 2.5.3 Querying Revisions via Labels

Helm uses Kubernetes labels to track and query revisions:

```
kubectl get secrets -n prod -l owner=helm,name=my-release

  sh.helm.release.v1.my-release.v1  (version: 1, status: superseded)
  sh.helm.release.v1.my-release.v2  (version: 2, status: superseded)
  sh.helm.release.v1.my-release.v3  (version: 3, status: deployed)

Find the current revision:
  kubectl get secrets -n prod -l owner=helm,name=my-release,status=deployed

List history (sorted by version):
  kubectl get secrets -n prod -l owner=helm,name=my-release \
    --sort-by=.metadata.labels.version

Manage max history:
  Count secrets with status=superseded, delete oldest if > max
```

---

## 2.6 Rollback Mechanism Internals

```
Command: helm rollback my-release 2

STEP 1: Retrieve target revision
  GET Secret: sh.helm.release.v1.my-release.v2
  Decode --> base64 --> gunzip --> JSON
  Extract: manifest, config, chart

STEP 2: Retrieve current revision (for 3-way merge)
  GET Secret: sh.helm.release.v1.my-release.v3
  Extract: old manifest (revision 3's manifest)

STEP 3: Compute 3-way merge
  Old Manifest = Revision 3's manifest (current deployed state)
  New Manifest = Revision 2's manifest (target state)
  Live State   = Actual cluster resources (via API GET queries)
  Compute patch = 3-way strategic merge

STEP 4: Apply the patch
  Resources in new but not old --> CREATE
  Resources in both old and new --> PATCH (3-way merge)
  Resources in old but not new --> DELETE

STEP 5: Record the rollback as a new revision
  Create Secret: sh.helm.release.v1.my-release.v4
  {
    "manifest": "<revision 2's manifest>",
    "config": "<revision 2's config>",
    "version": 4,
    "info": {"status": "deployed", "description": "Rollback to 2"}
  }
  Update revision 3's label: status=superseded
```

**Critical insight:** A rollback does NOT re-apply revision 2's resources directly. It creates a **new revision** (rev 4) with revision 2's manifest as desired state, then uses the 3-way merge to reconcile the cluster. This means:
- Manual edits to resources between rev 2 and now may be preserved or overwritten
- The rollback creates a new history entry (audit trail preserved)
- You can roll forward: `helm rollback my-release 3`

---

## 2.7 State Management -- The 3-Way Strategic Merge Patch

### 2.7.1 Why 3-Way Merge?

Kubernetes supports strategic merge patches -- JSON patches that understand K8s API type semantics (e.g., merging container lists by name, not by array index). Helm 3 combines this with a 3-way merge:

```
+--------------------+    +--------------------+    +----------------------+
|  OLD MANIFEST      |    |  NEW MANIFEST      |    |  LIVE STATE          |
|  (previous rev)    |    |  (desired)         |    |  (current cluster)   |
|                    |    |                    |    |                      |
| replicas: 3        |    | replicas: 5        |    | replicas: 3          |
| image: v1.0        |    | image: v2.0        |    | image: v1.0          |
| ports: [80]        |    | ports: [80]        |    | ports: [80, 9090]    |
|                    |    |                    |    |   ^ manually added   |
+---------+----------+    +---------+----------+    +----------+-----------+
          |                         |                          |
          +-------------------------+--------------------------+
                                    |
                                    v
+-------------------------------------------------------------------+
|                        MERGE LOGIC                                 |
|                                                                    |
| 1. Fields CHANGED between old and new:          --> UPDATE in live |
|    replicas: 3 -> 5     image: v1.0 -> v2.0                       |
|                                                                    |
| 2. Fields SAME in old and new, DIFFERENT in live: --> PRESERVE     |
|    ports: [80] -> [80] (same) but live has [80, 9090]             |
|    Manual change is kept (Helm didn't change this field)           |
|                                                                    |
| 3. Fields REMOVED from new (existed in old):   --> DELETE in live  |
|                                                                    |
| 4. Fields ADDED in new (not in old):            --> ADD to live    |
+------------------------------------+-------------------------------+
                                     |
                                     v
+-------------------------------------------------------------------+
|                        RESULT                                      |
|  replicas: 5        <-- Updated (Helm changed it)                  |
|  image: v2.0        <-- Updated (Helm changed it)                  |
|  ports: [80, 9090]  <-- PRESERVED manual port addition             |
+-------------------------------------------------------------------+
```

### 2.7.2 The 3-Way Merge Algorithm

```
Algorithm: 3-Way Strategic Merge Patch

For each resource in (OLD union NEW union LIVE):

  Case 1: Resource in NEW but NOT in OLD
    --> Resource is new in this release
    --> CREATE in cluster

  Case 2: Resource in OLD but NOT in NEW
    --> Resource was removed from the chart
    --> DELETE from cluster

  Case 3: Resource in BOTH OLD and NEW
    --> Compute 3-way strategic merge patch
    a. DIFF = StrategicMergePatch(OLD, NEW)
       (What changed between revisions)
    b. Apply DIFF to LIVE:
       - Fields in DIFF that conflict with LIVE: strategic merge resolves
       - Fields NOT in DIFF: left as-is (preserving LIVE changes)
    c. PATCH the resource in the cluster

  Case 4: Resource in LIVE but NOT in OLD or NEW
    --> Resource exists in cluster but Helm didn't create it
    --> IGNORE (Helm does not touch unowned resources)
```

**Warning:** The 3-way merge preserves manual edits to fields Helm didn't modify. However, if you change a field via `kubectl edit` that Helm ALSO changes in the next upgrade, Helm's change **wins**. There is no automatic reconciliation of conflicting edits.

---

## 2.8 Release Tracking -- Labels and Ownership

### 2.8.1 Standard Labels Applied by Helm

**Recommended Kubernetes labels (`app.kubernetes.io/*`):**

```yaml
metadata:
  labels:
    app.kubernetes.io/name: mychart
    app.kubernetes.io/instance: my-release
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/managed-by: Helm
```

**Helm-specific labels:**

```yaml
metadata:
  labels:
    helm.sh/chart: mychart-1.0.0
```

### 2.8.2 How Helm Identifies Its Resources

```
+-----------------------------------------------------------------------+
|              HOW HELM IDENTIFIES OWNED RESOURCES                       |
|                                                                        |
|  Helm uses these labels to identify and manage resources:              |
|                                                                        |
|  1. app.kubernetes.io/managed-by: Helm    <-- "I own this"            |
|  2. app.kubernetes.io/instance: <release>  <-- "Which release?"       |
|  3. helm.sh/chart: <chart>-<version>       <-- "Which chart+version?" |
|                                                                        |
|  During upgrade, Helm queries the cluster:                             |
|                                                                        |
|  kubectl get all -n prod -l app.kubernetes.io/managed-by=Helm,         |
|    app.kubernetes.io/instance=my-release                               |
|                                                                        |
|  This returns ALL resources belonging to the release, which Helm       |
|  compares against the new manifest to determine:                       |
|    - Which resources to create (in new, not in live)                   |
|    - Which resources to update (in new and live)                       |
|    - Which resources to delete (in live, not in new)                   |
+-----------------------------------------------------------------------+
```

**Production Note:** If you manually add an `app.kubernetes.io/managed-by: Helm` label to a resource you created outside Helm, Helm will treat it as an owned resource and may modify or delete it during upgrades. Never manually apply Helm's ownership labels.

---

## 2.9 Helm's Namespace Awareness

```
+-----------------------------------------------------------------------+
|                    NAMESPACE AWARENESS                                 |
|                                                                        |
|  Namespace resolution order:                                           |
|                                                                        |
|  1. --namespace flag            (explicit)                             |
|  2. HELM_NAMESPACE env var      (environment)                          |
|  3. kubeconfig current-context namespace  (context default)            |
|  4. "default" namespace         (fallback)                             |
|                                                                        |
|  Release storage:                                                      |
|  Release Secrets are stored in the SAME namespace as the release.      |
|  A release in namespace "prod" has its Secrets in namespace "prod".    |
|                                                                        |
|  This is different from Helm 2, where Tiller stored ALL releases       |
|  in kube-system, creating a multi-tenancy nightmare.                   |
|                                                                        |
|  Cross-namespace releases:                                             |
|  A single release can create resources in multiple namespaces          |
|  if templates explicitly set metadata.namespace. However, the          |
|  release itself (and its Secrets) live in the namespace specified      |
|  at install time.                                                      |
|                                                                        |
|  Example:                                                              |
|    helm install my-release ./mychart --namespace app-prod              |
|    --> Release stored in app-prod namespace                            |
|    --> Templates can deploy to app-prod, monitoring, istio-system      |
|        if they specify different namespaces in metadata.namespace      |
+-----------------------------------------------------------------------+
```

### 2.9.1 Creating Namespaces

```bash
# Auto-create namespace if it doesn't exist
helm install my-release ./mychart --namespace app-prod --create-namespace

# Check if namespace exists first
kubectl get namespace app-prod || kubectl create namespace app-prod
helm install my-release ./mychart --namespace app-prod
```

**Exam Tip:** `--create-namespace` is a frequently tested Helm flag. Without it, installing into a non-existent namespace fails. Helm 3.2+ supports this flag.

---

## 2.10 Internal Mechanics of Every Major Command

### 2.10.1 `helm install` -- Internal Flow

```
+-------------------------------------------------------------------------+
|                    helm install my-release ./mychart                     |
|                                                                          |
|  +------------------------------------------------------------------+    |
|  | Step 1: Validate input                                            |    |
|  |   - Check release name is valid (RFC 1123 DNS subdomain)          |    |
|  |   - Check release name is not already in use (in target namespace)|    |
|  |   - Load and parse chart (locally or from remote)                  |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 2: Render CRDs (if any exist in crds/)                       |    |
|  |   - Install CRDs BEFORE any template rendering                     |    |
|  |   - CRDs are NOT added to the manifest                             |    |
|  |   - CRDs are NOT tracked as part of the release                    |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 3: Merge values                                              |    |
|  |   - Load chart/values.yaml (defaults)                              |    |
|  |   - Apply --values file(s)                                         |    |
|  |   - Apply --set / --set-string overrides                           |    |
|  |   - Validate against values.schema.json (if present)              |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 4: Resolve dependencies                                      |    |
|  |   - Load subchart .tgz files from charts/                          |    |
|  |   - Recursively render subcharts with scoped values                |    |
|  |   - Merge subchart manifests into main manifest                    |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 5: Execute all templates                                     |    |
|  |   - For each template file (excluding _*.tpl and tests/):         |    |
|  |     - Execute Go template with merged .Values, .Release, etc.     |    |
|  |     - Validate output is valid YAML                                |    |
|  |     - Append to manifest with YAML document separator              |    |
|  |   - Also render NOTES.txt for post-install output                 |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 6: Create initial release Secret                             |    |
|  |   - Status: pending-install                                        |    |
|  |   - This allows recovery if the install fails partway through      |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 7: Run pre-install hooks (in weight order)                    |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 8: Apply resources to Kubernetes API (in install order)      |    |
|  |   - For each resource in the manifest:                             |    |
|  |     - Add standard labels (app.kubernetes.io/*, helm.sh/chart)     |    |
|  |     - POST to appropriate API endpoint                             |    |
|  |     - Handle errors (retry? fail?)                                 |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 9: Run post-install hooks (in weight order)                   |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 10: Finalize                                                  |    |
|  |   - Update release Secret status: pending-install --> deployed     |    |
|  |   - Print NOTES.txt output to user                                 |    |
|  |   - Return success                                                 |    |
|  +------------------------------------------------------------------+    |
+-------------------------------------------------------------------------+
```

### 2.10.2 `helm upgrade` -- Internal Flow

```
+-------------------------------------------------------------------------+
|                helm upgrade my-release ./mychart --set replicas=5        |
|                                                                          |
|  +------------------------------------------------------------------+    |
|  | Step 1: Validate input                                            |    |
|  |   - Check release exists (query Secrets by label)                  |    |
|  |   - Check current revision status is "deployed" (not "failed")    |    |
|  |   - Load and parse new chart                                       |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 2: Load current release                                      |    |
|  |   - GET Secret: sh.helm.release.v1.{name}.v{current}              |    |
|  |   - Decode -> base64 -> gunzip -> JSON                            |    |
|  |   - Extract: old manifest, old config                              |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 3: Render new manifest                                       |    |
|  |   - Same as install Step 3-5 (merge values, execute templates)    |    |
|  |   - Produces: new manifest                                         |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 4: Create pending-upgrade Secret                             |    |
|  |   - New revision number: current + 1                              |    |
|  |   - Status: pending-upgrade                                        |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 5: Compute 3-Way Merge                                       |    |
|  |   - Old = Old manifest from current revision                       |    |
|  |   - New = New manifest from rendered chart                         |    |
|  |   - Live = Actual cluster state (GET all resources by label)      |    |
|  |   - Determine: Create / Update / Delete for each resource          |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 6: Run pre-upgrade hooks (in weight order)                    |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 7: Apply changes to Kubernetes API                            |    |
|  |   - CREATE new resources                                           |    |
|  |   - PATCH existing resources (3-way merge)                         |    |
|  |   - DELETE removed resources (present in old, absent in new)       |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 8: Run post-upgrade hooks (in weight order)                    |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 9: Finalize                                                    |    |
|  |   - Update new revision Secret: pending-upgrade --> deployed        |    |
|  |   - Update old revision Secret: deployed --> superseded             |    |
|  |   - Apply --history-max garbage collection                          |    |
|  |   - Print NOTES.txt output                                          |    |
|  +------------------------------------------------------------------+    |
+-------------------------------------------------------------------------+
```

### 2.10.3 `helm rollback` -- Internal Flow

```
+-------------------------------------------------------------------------+
|                     helm rollback my-release 2                           |
|                                                                          |
|  +------------------------------------------------------------------+    |
|  | Step 1: Validate                                                  |    |
|  |   - Check release exists                                          |    |
|  |   - Check revision 2 exists (query Secret by label version)       |    |
|  |   - Revision 2's status is not "failed" (cannot rollback to       |    |
|  |     a failed revision unless forced)                               |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 2: Load target revision (rev 2)                              |    |
|  |   - GET Secret, decode, extract manifest and config                |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 3: Load current revision (for old manifest in merge)         |    |
|  |   - GET current deployed Secret, extract manifest                  |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 4: Re-render templates                                       |    |
|  |   - Use the stored chart + stored config from revision 2          |    |
|  |   - Re-render to produce the "new" manifest                        |    |
|  |   - (This rebuild ensures consistency with current cluster state)  |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 5: Create pending-rollback Secret                            |    |
|  |   - New revision number: current + 1                              |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 6: Run pre-rollback hooks (in weight order)                   |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 7: Compute 3-way merge and apply                             |    |
|  |   - Same as upgrade mechanism                                      |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 8: Finalize                                                   |    |
|  |   - Update new revision Secret: pending-rollback --> deployed      |    |
|  |   - Update old revision: deployed --> superseded                   |    |
|  |   - Garbage collect old revisions                                  |    |
|  +------------------------------------------------------------------+    |
+-------------------------------------------------------------------------+
```

### 2.10.4 `helm uninstall` -- Internal Flow

```
+-------------------------------------------------------------------------+
|                      helm uninstall my-release                           |
|                                                                          |
|  +------------------------------------------------------------------+    |
|  | Step 1: Validate                                                  |    |
|  |   - Check release exists (query Secrets by label)                  |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 2: Load release                                              |    |
|  |   - GET the current deployed Secret                                |    |
|  |   - Extract the manifest                                           |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 3: Run pre-delete hooks (in weight order)                      |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 4: Delete Kubernetes resources (reverse install order)       |    |
|  |   - For each resource in the manifest (in REVERSE order):          |    |
|  |     - Query by label to confirm it still exists                    |    |
|  |     - DELETE from API server                                       |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 5: Run post-delete hooks (in weight order)                     |    |
|  +---------------------------+--------------------------------------+    |
|                              |                                          |
|                              v                                          |
|  +------------------------------------------------------------------+    |
|  | Step 6: Clean up                                                   |    |
|  |   - DELETE all release Secrets (unless --keep-history)              |    |
|  |   - Release is fully removed                                       |    |
|  +------------------------------------------------------------------+    |
+-------------------------------------------------------------------------+
```

---

## 2.11 Helm SDK -- Programmatic Helm Usage

Helm exposes a Go SDK (`helm.sh/helm/v3`) that allows embedding Helm functionality in Go applications. This is used by tools like Argo CD, Flux, and custom Kubernetes controllers.

### 2.11.1 Core SDK Packages

| Package | Purpose |
|---------|---------|
| `pkg/action` | High-level operations: install, upgrade, rollback, uninstall |
| `pkg/chart` | Chart loading, parsing, and dependency resolution |
| `pkg/chartutil` | Chart utilities: packaging, values coercion |
| `pkg/release` | Release data structures and status management |
| `pkg/repo` | Repository index operations |
| `pkg/engine` | Template rendering engine |
| `pkg/kube` | Kubernetes client abstraction |
| `pkg/cli` | Environment settings (kubeconfig, namespace, etc.) |
| `pkg/registry` | OCI registry client |
| `pkg/storage/driver` | Storage driver interface and implementations |

### 2.11.2 Basic SDK Usage Example

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

    // Create the install action
    actionConfig := new(action.Configuration)
    if err := actionConfig.Init(
        settings.RESTClientGetter(),
        settings.Namespace(),
        os.Getenv("HELM_DRIVER"),
        log.Printf,
    ); err != nil {
        log.Fatal(err)
    }

    // Load the chart
    chart, err := loader.Load("/path/to/mychart")
    if err != nil {
        log.Fatal(err)
    }

    // Configure the install
    install := action.NewInstall(actionConfig)
    install.ReleaseName = "my-release"
    install.Namespace = "default"
    install.CreateNamespace = true
    install.Wait = true
    install.Timeout = 300 * time.Second

    // Set values
    install.SetValues = map[string]interface{}{
        "replicaCount": 3,
    }

    // Run the install
    release, err := install.Run(chart, nil)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Installed release: %s (revision: %d)", release.Name, release.Version)
}
```

### 2.11.3 SDK Upgrade Example

```go
upgrade := action.NewUpgrade(actionConfig)
upgrade.Namespace = "default"
upgrade.MaxHistory = 10
upgrade.Wait = true
upgrade.ResetValues = false   // Keep previous values, merge new ones
upgrade.ReuseValues = false   // Don't reuse previous --set values

release, err := upgrade.Run("my-release", chart, map[string]interface{}{
    "replicaCount": 5,
})
if err != nil {
    log.Fatal(err)
}
```

### 2.11.4 SDK Rollback Example

```go
rollback := action.NewRollback(actionConfig)
rollback.Version = 2          // Rollback to revision 2
rollback.Wait = true

err := rollback.Run("my-release")
if err != nil {
    log.Fatal(err)
}
```

### 2.11.5 SDK -- Accessing Release Information

```go
// Get release history
history := action.NewHistory(actionConfig)
history.Max = 10
releases, err := history.Run("my-release")

// Get specific release info
get := action.NewGet(actionConfig)
get.Version = 3
release, err := get.Run("my-release")

// Get rendered manifest
getManifest := action.NewGetManifest(actionConfig)
manifest, err := getManifest.Run("my-release")

// Get values
getValues := action.NewGetValues(actionConfig)
getValues.AllValues = true
values, err := getValues.Run("my-release")
```

**Production Note:** The Helm SDK is production-grade and used by Argo CD, Flux CD, Skaffold, Tanka, and many other tools. If you're building a Kubernetes operator or a custom deployment pipeline in Go, consider using the Helm SDK instead of shelling out to the Helm CLI.

---

## 2.12 Chapter Summary

| Component | One-Line Summary |
|-----------|-----------------|
| CLI to K8s path | User -> Helm CLI -> Templates + Values -> Rendered YAML -> client-go -> K8s API |
| Storage backend | Release history stored as base64-gzipped JSON in Secrets (`sh.helm.release.v1.{name}.v{n}`) |
| Revisions | Monotonic counter starting at 1; new revision on install, upgrade, and rollback |
| 3-way merge | Old manifest vs New manifest vs Live state -- preserves manual edits Helm didn't touch |
| Revision labels | `owner=helm`, `name={release}`, `status={deployed|superseded|...}`, `version={n}` |
| Resource ownership | `app.kubernetes.io/managed-by: Helm` + `app.kubernetes.io/instance: {release}` |
| Install order | Namespace first -> Config -> RBAC -> Workloads -> Ingress last |
| Uninstall order | Reverse of install order |
| Namespace awareness | Release per namespace; Secrets stored in release namespace; `--create-namespace` to auto-create |
| SDK | `helm.sh/helm/v3/pkg/action` -- production-grade Go library for programmatic Helm |

---

## 2.13 What's Next

Chapter 3 covers installing Helm across all major platforms, configuring shell completion, managing multiple Helm versions, and verifying your installation.

--> Continue to **Chapter 3: Installing Helm** (`03-installing-helm.md`)
