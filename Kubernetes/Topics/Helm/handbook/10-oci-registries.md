# Chapter 10: OCI Registries

## What Is the Open Container Initiative (OCI)?

The **Open Container Initiative (OCI)** is an open governance structure under the Linux Foundation that defines industry standards for container formats and runtimes. Originally created to standardize container images, OCI has expanded its scope to cover arbitrary artifact types — including Helm charts.

**Why Helm Adopted OCI**

Before OCI support, Helm charts were distributed exclusively through traditional Helm repositories — static index files served over HTTP. This model had several shortcomings:

1. **No built-in authentication** — Auth was bolted on via basic auth or reverse proxies.
2. **No content-addressable storage** — Chart versions were identified solely by filename and version field.
3. **No standard distribution mechanism** — Every registry (ChartMuseum, Harbor, etc.) implemented its own API.
4. **Fragile consistency** — The `index.yaml` file could become out of sync with stored chart packages.
5. **No cross-organization sharing** — Teams already had container registries; Helm required a separate distribution channel.

By adopting OCI, Helm gained access to every OCI-compliant container registry (Docker Hub, ECR, ACR, GAR, Harbor, GHCR, GitLab, etc.) with the same tooling teams already use for container images.

## OCI vs Traditional Helm Repositories

| Aspect | Traditional Helm Repo | OCI Registry |
|---|---|---|
| **Structure** | HTTP server serving `index.yaml` + `.tgz` files | OCI-compliant container registry with layered blobs |
| **Authentication** | Manual (basic auth, reverse proxy, cloudfront signed URLs) | Native via `helm registry login`, credential helpers, platform IAM |
| **Storage** | Flat file storage (S3, GCS, filesystem) | Content-addressable blob storage with deduplication |
| **Distribution** | Pull via HTTP GET; push requires separate tooling (`helm s3`, `helm gcs`, ChartMuseum, etc.) | Push/pull via OCI distribution spec (`helm push`, `helm pull oci://`) |
| **Versioning** | Version field in `Chart.yaml` + entry in `index.yaml` | OCI tags + manifest digests (immutable content addressing) |
| **Signing** | Helm provenance files (`.prov`) with GPG | Cosign / Notation for OCI artifact signing |
| **Cache / CDN** | Relies on upstream CDN (CloudFront, Fastly) | Built-in registry replication, pull-through cache |
| **Dependency resolution** | `helm repo add` + `helm dep update` | Native OCI references in `Chart.yaml` dependencies |
| **Standardization** | Helm-specific; not interoperable with other tools | OCI Distribution Spec v1.0+; any compliant registry works |
| **Garbage collection** | Manual pruning of old `.tgz` files | Registry-level GC policies and retention rules |

## OCI Support Timeline

| Helm Version | OCI Support Status |
|---|---|
| < 3.7.0 | No OCI support |
| 3.7.0 | OCI support introduced (opt-in via `HELM_EXPERIMENTAL_OCI=1`) |
| 3.8.0 | OCI support promoted to GA (general availability); `HELM_EXPERIMENTAL_OCI` removed |
| 3.13.0+ | OCI artifact support mature; `helm dependency update` supports OCI references without additional flags |

**Exam Tip:** Helm 3.8.0 is the key milestone — OCI support became Generally Available without experimental flags.

## `helm registry login`

Authenticates to an OCI-compliant registry. Credentials are stored in `~/.config/helm/registry/config.json` (or `$HELM_REGISTRY_CONFIG`).

### Syntax

```
helm registry login [HOST]
```

### Flags

| Flag | Type | Description |
|---|---|---|
| `--hostname` | string | Registry hostname (alias for HOST positional argument) |
| `--username` / `-u` | string | Registry username |
| `--password` / `-p` | string | Registry password (not recommended — prefer `--password-stdin`) |
| `--password-stdin` | bool | Read password from stdin (secure; no shell history exposure) |
| `--insecure` | bool | Allow connections to TLS registries without valid certificates |
| `--registry-config` | string | Path to registry config file (default: `~/.config/helm/registry/config.json`) |
| `--debug` | bool | Enable verbose debug output |

### Examples

**Interactive login (prompts for credentials):**

```bash
helm registry login registry.example.com
```

**Password via stdin (recommended for automation):**

```bash
echo "$REGISTRY_PASSWORD" | helm registry login registry.example.com \
  --username myuser \
  --password-stdin
```

**Login with explicit flags (not recommended — exposes password in shell history):**

```bash
helm registry login registry.example.com \
  --username myuser \
  --password mypassword
```

**Insecure login (dev environments only):**

```bash
helm registry login myregistry.local:5000 --insecure
```

**Note:** The `--password` flag exposes the password in process listings (`ps aux`) and shell history. Always prefer `--password-stdin` for automation scripts and CI pipelines.

### Registry URL Formats

| Registry Type | URL Format | Example |
|---|---|---|
| Docker Hub | `registry-1.docker.io` or `docker.io` | `oci://registry-1.docker.io/myuser/mychart` |
| AWS ECR | `<account>.dkr.ecr.<region>.amazonaws.com` | `oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/mychart` |
| Azure ACR | `<registry>.azurecr.io` | `oci://myregistry.azurecr.io/helm/mychart` |
| GAR | `<region>-docker.pkg.dev/<project>/<repo>` | `oci://us-docker.pkg.dev/my-project/helm-repo/mychart` |
| Harbor | `<harbor-host>/<project>` | `oci://harbor.example.com/helm-charts/mychart` |
| GHCR | `ghcr.io/<owner>/<repo>` | `oci://ghcr.io/myorg/helm-charts/mychart` |
| GitLab | `registry.gitlab.com/<namespace>/<project>` | `oci://registry.gitlab.com/mygroup/helm-charts/mychart` |
| Quay | `quay.io/<namespace>` | `oci://quay.io/myorg/mychart` |

## `helm registry logout`

Removes stored credentials for a registry from the registry config file.

### Syntax

```
helm registry logout [HOST]
```

### Example

```bash
helm registry logout registry.example.com
```

**Note:** Logging out removes credentials from the local config file. It does not invalidate server-side tokens. If you are using token-based auth (e.g., AWS ECR, ACR), the token may still be valid until it expires.

## `helm push` — Pushing Charts to OCI

Pushes a packaged chart (`.tgz`) to an OCI-compliant registry.

### Syntax

```
helm push <chart.tgz> oci://<registry>/<path>
```

### Examples

**Push a chart with explicit version tag:**

```bash
helm package mychart/
helm push mychart-1.2.3.tgz oci://registry.example.com/charts
```

This creates the OCI reference `registry.example.com/charts/mychart:1.2.3`.

**Push with inline packaging (Helm 3.13+):**

```bash
helm push mychart/ oci://registry.example.com/charts
```

**Push to a custom path:**

```bash
helm push mychart-1.2.3.tgz oci://registry.example.com/team/helm/mychart
```

**Push with multiple tags (via separate pushes — OCI tags are mutable pointers):**

```bash
helm push mychart-1.2.3.tgz oci://registry.example.com/charts
# Tagging latest typically requires registry-specific tooling
# Many registries support retagging via their own CLIs
```

**Warning:** OCI tags are mutable by default. Pushing the same version tag twice overwrites the previous manifest. Unlike traditional Helm repos where `index.yaml` guards against this, OCI registries do not prevent tag overwrites. Use immutable tags or digest pinning in production.

**Production Note:** Always push from CI using a locked version tag that matches `Chart.yaml`. Never rely on `latest` tags for production deployments.

## `helm pull oci://` — Pulling Charts from OCI

Pulls a chart from an OCI registry and optionally extracts it.

### Syntax

```
helm pull oci://<registry>/<path> --version <version> [flags]
```

### Flags

| Flag | Type | Description |
|---|---|---|
| `--version` | string | Specific version to pull (maps to OCI tag) |
| `--destination` / `-d` | string | Directory to write the chart (default: `.`) |
| `--untar` | bool | Extract the `.tgz` after downloading |
| `--untardir` | string | Directory to extract into (implies `--untar`, default: chart name) |
| `--prov` | bool | Pull provenance file (not applicable for OCI; use Cosign/Notation) |
| `--insecure-skip-tls-verify` | bool | Skip TLS certificate verification |
| `--ca-file` | string | CA certificate bundle for TLS verification |
| `--cert-file` | string | Client certificate for mutual TLS |
| `--key-file` | string | Client certificate key for mutual TLS |
| `--keyring` | string | Keyring for provenance verification (traditional repos only) |
| `--pass-credentials` | bool | Pass registry credentials to all domains |
| `--password` | string | Registry password |
| `--username` | string | Registry username |

### Examples

**Pull a specific version:**

```bash
helm pull oci://registry.example.com/charts/mychart --version 1.2.3
```

**Pull and extract to current directory:**

```bash
helm pull oci://registry.example.com/charts/mychart --version 1.2.3 --untar
```

**Pull to a specific destination:**

```bash
helm pull oci://registry.example.com/charts/mychart \
  --version 1.2.3 \
  --destination ./downloads/
```

## `helm install oci://` — Direct Install from OCI

Installs a release directly from an OCI registry without a local pull step.

### Syntax

```
helm install <release-name> oci://<registry>/<path> [flags]
```

### Examples

**Direct install (uses the latest tag by default):**

```bash
helm install myapp oci://registry.example.com/charts/mychart
```

**Install a specific version:**

```bash
helm install myapp oci://registry.example.com/charts/mychart --version 1.2.3
```

**Install with namespace and values:**

```bash
helm install myapp oci://registry.example.com/charts/mychart \
  --version 1.2.3 \
  --namespace production \
  --values production-values.yaml
```

**Note:** `helm install` from OCI is equivalent to `helm pull oci://... && helm install ...`, but the chart is pulled directly into the Helm cache without a local `.tgz` file.

## `helm show oci://` — Inspect Chart from OCI

Displays chart metadata and content directly from an OCI registry.

### Syntax

```
helm show <subcommand> oci://<registry>/<path> [flags]
```

Where `<subcommand>` is one of `all`, `chart`, `readme`, `values`, `crds`.

### Examples

```bash
helm show all oci://registry.example.com/charts/mychart --version 1.2.3
helm show chart oci://registry.example.com/charts/mychart --version 1.2.3
helm show values oci://registry.example.com/charts/mychart --version 1.2.3
helm show readme oci://registry.example.com/charts/mychart --version 1.2.3
helm show crds oci://registry.example.com/charts/mychart --version 1.2.3
```

**Exam Tip:** `helm show` works identically for OCI and traditional repo sources. The only difference is the `oci://` prefix and the reference format.

## OCI Dependencies in Chart.yaml

Helm 3.8+ supports OCI references directly in `Chart.yaml` dependencies. No additional repository configuration is required.

### Syntax

```yaml
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "oci://registry.example.com/charts/postgresql"
  - name: redis
    version: "18.0.0"
    repository: "oci://registry-1.docker.io/bitnamicharts/redis"
  - name: internal-lib
    version: "2.3.0"
    repository: "oci://ghcr.io/myorg/helm-charts/internal-lib"
```

### Resolution

```bash
helm dependency update myapp/
```

This pulls all OCI dependencies and stores them in `charts/` (or uses `--skip-refresh` if already cached).

### Mixing Sources

You can mix traditional repositories and OCI dependencies in the same chart:

```yaml
dependencies:
  - name: nginx
    version: "15.0.0"
    repository: "https://charts.bitnami.com/bitnami"   # traditional
  - name: postgresql
    version: "12.1.0"
    repository: "oci://registry-1.docker.io/bitnamicharts/postgresql"  # OCI
```

**Note:** For OCI dependencies, you must be authenticated (`helm registry login`) before running `helm dependency update`.

## Authentication Methods

### 1. Username and Password (Basic Auth)

```bash
echo "$PASSWORD" | helm registry login registry.example.com \
  --username myuser --password-stdin
```

### 2. Token-Based Authentication

Many registries issue short-lived tokens instead of permanent credentials:

```bash
echo "$TOKEN" | helm registry login registry.example.com \
  --username oauth2accesstoken --password-stdin
```

For some registries (like GHCR), the username is arbitrary when using tokens:

```bash
echo "$GITHUB_TOKEN" | helm registry login ghcr.io \
  --username ignored --password-stdin
```

### 3. Credential Helpers (docker-credential-*)

Helm's OCI support uses the same authentication store as Docker. If you have `docker login` configured, Helm can read those credentials. This also applies to platform-specific credential helpers:

- **AWS:** `docker-credential-ecr-login`
- **GCP:** `docker-credential-gcr`
- **Azure:** `docker-credential-acr`

The credential helper chain is configured in `~/.docker/config.json` under `credHelpers`:

```json
{
  "credHelpers": {
    "123456789012.dkr.ecr.us-east-1.amazonaws.com": "ecr-login",
    "us-docker.pkg.dev": "gcr"
  }
}
```

**Note:** Credential helpers are preferred for cloud registries because they handle automatic token refresh.

## AWS Elastic Container Registry (ECR)

### Full Workflow

#### Step 1: Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name helm/mychart \
  --region us-east-1
```

#### Step 2: Authenticate

```bash
aws ecr get-login-password --region us-east-1 | \
  helm registry login 123456789012.dkr.ecr.us-east-1.amazonaws.com \
    --username AWS --password-stdin
```

The username for ECR is always `AWS` (case-sensitive).

#### Step 3: Push Chart

```bash
helm package mychart/
helm push mychart-1.2.3.tgz oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm/
```

The full reference becomes `123456789012.dkr.ecr.us-east-1.amazonaws.com/helm/mychart:1.2.3`.

#### Step 4: Pull / Install

```bash
helm pull oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm/mychart --version 1.2.3
helm install myapp oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm/mychart --version 1.2.3
```

### IAM Permissions

Minimum required IAM policy for Helm chart operations:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:GetRepositoryPolicy",
        "ecr:DescribeRepositories",
        "ecr:ListImages",
        "ecr:DescribeImages",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "*"
    }
  ]
}
```

**Production Note:** Scope the `Resource` field to specific repository ARNs: `arn:aws:ecr:us-east-1:123456789012:repository/helm/*`.

### CI/CD Integration (GitHub Actions)

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/helm-publisher
    aws-region: us-east-1

- name: Login to ECR
  run: |
    aws ecr get-login-password --region us-east-1 | \
      helm registry login 123456789012.dkr.ecr.us-east-1.amazonaws.com \
        --username AWS --password-stdin

- name: Push chart
  run: |
    helm package ./chart
    helm push *.tgz oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm/
```

## Azure Container Registry (ACR)

### Full Workflow

#### Step 1: Create ACR Instance and Repository

```bash
az acr create --resource-group myRG --name myregistry --sku Standard
az acr repository create --name myregistry --repository helm/mychart
```

#### Step 2: Authenticate

**Option A: `az acr login` (AZ CLI):**

```bash
az acr login --name myregistry
```
This automatically configures Docker credential helpers that Helm can use.

**Option B: Helm-specific login:**

```bash
TOKEN=$(az acr login --name myregistry --expose-token --output tsv --query accessToken)
echo "$TOKEN" | helm registry login myregistry.azurecr.io \
  --username 00000000-0000-0000-0000-000000000000 --password-stdin
```

#### Step 3: Push and Pull

```bash
helm package mychart/
helm push mychart-1.2.3.tgz oci://myregistry.azurecr.io/helm/
helm pull oci://myregistry.azurecr.io/helm/mychart --version 1.2.3
```

### RBAC

Minimum role for push: **AcrPush**
Minimum role for pull: **AcrPull**

```bash
az role assignment create \
  --assignee <service-principal-id> \
  --role AcrPush \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.ContainerRegistry/registries/myregistry
```

**Production Note:** Use repository-scoped tokens with specific permissions rather than admin credentials. ACR supports scoped access tokens with fine-grained repository-level permissions.

## Google Artifact Registry (GAR)

### Full Workflow

#### Step 1: Create Repository

```bash
gcloud artifacts repositories create helm-repo \
  --repository-format=docker \
  --location=us-central1
```

GAR uses the Docker format for OCI artifacts. Helm charts are stored alongside container images in the same repository.

#### Step 2: Authenticate

```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```

This configures the Docker credential helper. Helm's OCI integration reads from the same credential store.

Alternatively, for explicit Helm login:

```bash
gcloud auth print-access-token | \
  helm registry login us-central1-docker.pkg.dev \
    --username oauth2accesstoken --password-stdin
```

#### Step 3: Push and Pull

```bash
helm package mychart/
helm push mychart-1.2.3.tgz oci://us-central1-docker.pkg.dev/my-project/helm-repo/
helm pull oci://us-central1-docker.pkg.dev/my-project/helm-repo/mychart --version 1.2.3
```

### IAM Permissions

Minimum roles:

| Action | Role |
|---|---|
| Pull charts | `roles/artifactregistry.reader` |
| Push charts | `roles/artifactregistry.writer` |
| Manage repository | `roles/artifactregistry.admin` |

## Harbor

### Full Workflow

#### Step 1: Create Project

Via Harbor UI or API, create a **public** or **private** project (e.g., `helm-charts`).

#### Step 2: Create Robot Account

In Harbor UI: Projects → `helm-charts` → Robot Accounts → + New Robot Account.

Set permissions: **Pull** and **Push** to the project. Copy the generated secret.

#### Step 3: Authenticate

```bash
echo "<robot-secret>" | helm registry login harbor.example.com \
  --username "robot\$helm-charts+helm-pusher" --password-stdin
```

**Note:** The `$` in the robot account name must be escaped (`\$`) in bash. Alternatively, use single quotes.

#### Step 4: Push and Pull

```bash
helm package mychart/
helm push mychart-1.2.3.tgz oci://harbor.example.com/helm-charts/
helm pull oci://harbor.example.com/helm-charts/mychart --version 1.2.3
```

### Harbor-Specific Notes

- Harbor stores OCI artifacts as `application/vnd.cncf.helm.chart.config.v1+json` media type.
- OCI support in Harbor requires **Harbor 2.0+**.
- Charts appear in the Harbor UI under the project's artifact browser alongside container images.
- Harbor supports **Cosign** signing for OCI artifacts, which can be used for Helm chart signing.

## Docker Hub OCI

Docker Hub supports OCI artifacts starting from **Docker Hub v2**. Helm charts are stored as OCI artifacts with the Helm-specific media type.

### Full Workflow

#### Step 1: Login

```bash
echo "$DOCKER_PASSWORD" | helm registry login registry-1.docker.io \
  --username mydockeruser --password-stdin
```

Alternatively, use an **access token** (generated at `hub.docker.com/settings/security`):

```bash
echo "$DOCKERHUB_TOKEN" | helm registry login registry-1.docker.io \
  --username mydockeruser --password-stdin
```

**Note:** Docker Hub requires the repository to exist before pushing. You can create it by pushing a dummy container image first, or by creating it via the Docker Hub UI / API.

#### Step 2: Push and Pull

```bash
helm package mychart/
helm push mychart-1.2.3.tgz oci://registry-1.docker.io/mydockeruser/
helm pull oci://registry-1.docker.io/mydockeruser/mychart --version 1.2.3
```

**Production Note:** Docker Hub imposes rate limits on pulls (100/6h for anonymous, 200/6h for authenticated free users). Use authenticated pulls and consider a Pro/Team plan for CI-heavy workloads.

## GitHub Container Registry (GHCR)

### Full Workflow

#### Step 1: Create Personal Access Token (PAT)

In GitHub: **Settings → Developer settings → Personal access tokens → Fine-grained tokens** (or classic tokens).

Minimum scopes for GHCR:

| Scope | Permission |
|---|---|
| `write:packages` | Push charts |
| `read:packages` | Pull charts |
| `delete:packages` | Delete chart versions |

#### Step 2: Authenticate

```bash
echo "$GITHUB_TOKEN" | helm registry login ghcr.io \
  --username <github-username> --password-stdin
```

For GitHub Actions, use the built-in `GITHUB_TOKEN`:

```bash
echo "${{ secrets.GITHUB_TOKEN }}" | helm registry login ghcr.io \
  --username ${{ github.actor }} --password-stdin
```

#### Step 3: Push and Pull

```bash
helm package mychart/
helm push mychart-1.2.3.tgz oci://ghcr.io/myorg/helm-charts/
helm pull oci://ghcr.io/myorg/helm-charts/mychart --version 1.2.3
```

**Important:** The GHCR repository path follows the convention `ghcr.io/<owner>/<repo>`. For organization-owned packages, `<owner>` is the org name. For user-owned packages, it is the username. The package/repo name does not need to match any GitHub repository.

#### Step 4: Make Package Public (Optional)

By default, GHCR packages are **private** and only accessible to the owner. To make them public, go to the package's settings page on GitHub and change visibility.

### GitHub Actions Integration

```yaml
name: Push Helm Chart to GHCR

on:
  push:
    tags:
      - 'chart-v*'

permissions:
  contents: read
  packages: write

jobs:
  push-chart:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set chart version from tag
        run: echo "CHART_VERSION=${GITHUB_REF_NAME#chart-v}" >> "$GITHUB_ENV"

      - name: Login to GHCR
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
            helm registry login ghcr.io \
              --username ${{ github.actor }} --password-stdin

      - name: Package and push
        run: |
          sed -i "s/^version:.*/version: ${CHART_VERSION}/" chart/Chart.yaml
          helm package chart/
          helm push *.tgz oci://ghcr.io/${{ github.repository_owner }}/helm-charts/
```

**Warning:** The `GITHUB_TOKEN` in GitHub Actions is limited to the current repository. For cross-repository or organization-wide packages, use a PAT stored in secrets.

## GitLab Container Registry

### Full Workflow

#### Step 1: Enable Container Registry

GitLab projects have the container registry enabled by default (for self-managed instances, ensure it is configured).

#### Step 2: Authenticate

**Using Personal Access Token:**

```bash
echo "$GITLAB_TOKEN" | helm registry login registry.gitlab.com \
  --username <gitlab-username> --password-stdin
```

**Using Deploy Token:**

```bash
echo "$DEPLOY_TOKEN" | helm registry login registry.gitlab.com \
  --username <deploy-token-username> --password-stdin
```

**Using CI Job Token (GitLab CI):**

```bash
echo "$CI_JOB_TOKEN" | helm registry login registry.gitlab.com \
  --username gitlab-ci-token --password-stdin
```

#### Step 3: Push and Pull

```bash
helm package mychart/
helm push mychart-1.2.3.tgz oci://registry.gitlab.com/<namespace>/<project>/
helm pull oci://registry.gitlab.com/<namespace>/<project>/mychart --version 1.2.3
```

### GitLab CI Example

```yaml
push-helm-chart:
  stage: deploy
  image: alpine/helm:3.13.0
  script:
    - echo "$CI_JOB_TOKEN" | helm registry login registry.gitlab.com --username gitlab-ci-token --password-stdin
    - helm package ./chart
    - helm push *.tgz oci://registry.gitlab.com/$CI_PROJECT_PATH/
  only:
    - main
```

## Private / Self-Hosted Registries

### Authentication Strategies

| Strategy | Description | Use Case |
|---|---|---|
| **Basic Auth** | Static username/password | Dev/test registries behind VPN |
| **mTLS** | Mutual TLS with client certificates | High-security on-premise environments |
| **LDAP / AD** | Delegated auth to directory service | Enterprise registries (Harbor + AD) |
| **OIDC / OAuth** | SSO integration | Team-facing registries with IdP |
| **Robot accounts** | Service-specific limited credentials | CI/CD automation |
| **Proxy auth** | Reverse proxy handles auth (e.g., nginx + basic auth) | Simple setups without built-in auth |
| **VPN + anonymous** | Registry is only accessible within VPN; no auth needed | Air-gapped or VPC-internal registries |

### Self-Hosted Registry Options

- **Harbor** (CNCF-graduated): Full-featured, Helm-native OCI support.
- **Distribution (docker/distribution)** : Lightweight, OCI-compliant. No UI.
- **Zot** : Minimal OCI registry with dedup and replication.
- **Nexus Repository** : Supports OCI alongside Maven, npm, PyPI, etc.

## OCI Annotations

OCI annotations provide metadata about the artifact. Helm sets several OCI annotations automatically when pushing charts.

### Standard OCI Annotations (`org.opencontainers.image.*`)

| Annotation | Description | Source |
|---|---|---|
| `org.opencontainers.image.title` | Chart name from `Chart.yaml` | `helm push` |
| `org.opencontainers.image.version` | Chart version | `helm push` |
| `org.opencontainers.image.description` | Chart description | `helm push` |
| `org.opencontainers.image.authors` | Chart maintainers | `helm push` |
| `org.opencontainers.image.created` | Push timestamp | `helm push` |
| `org.opencontainers.image.source` | Chart source URL (if set in `Chart.yaml`) | `helm push` |

### Inspecting Annotations

```bash
# Using crane (from go-containerregistry)
crane manifest oci://registry.example.com/charts/mychart:1.2.3 | jq '.annotations'

# Using oras (OCI Registry As Storage)
oras manifest fetch registry.example.com/charts/mychart:1.2.3 | jq '.annotations'
```

## OCI Artifacts vs OCI Images

Helm charts are stored as **OCI Artifacts**, not container images. The distinction matters:

| Aspect | OCI Image (Container) | OCI Artifact (Helm Chart) |
|---|---|---|
| **Media type** | `application/vnd.oci.image.manifest.v1+json` | `application/vnd.cncf.helm.chart.config.v1+json` |
| **Config blob** | Container runtime config (entrypoint, env, etc.) | Chart manifest with `Chart.yaml` metadata |
| **Layer 0** | Filesystem layer | Compressed chart `.tgz` |
| **Runtime** | Executed by container runtime (containerd, CRI-O) | Consumed by Helm CLI |
| **Tag convention** | Semantic version + `latest` | Prefer semantic version only; avoid `latest` |

**Media type chain:**

```
application/vnd.oci.image.manifest.v1+json
├── config:  application/vnd.cncf.helm.chart.config.v1+json
└── layers:
    └── layer: application/vnd.cncf.helm.chart.content.layer.v1+tar+gzip
```

**Note:** Because Helm charts use the standard OCI Manifest format, they are compatible with any OCI-compliant registry without modification. The registry does not need to "understand" Helm — it only sees standard OCI blobs and manifests.

## Helm Registry Configuration File

### Location

| OS | Path |
|---|---|
| Linux / macOS | `~/.config/helm/registry/config.json` |
| Windows | `%APPDATA%\helm\registry\config.json` |
| Custom | Set via `--registry-config` flag or `$HELM_REGISTRY_CONFIG` |

### Structure

```json
{
  "auths": {
    "registry.example.com": {
      "auth": "base64encoded(username:password)"
    },
    "ghcr.io": {
      "auth": "base64encoded(token)"
    }
  },
  "credHelpers": {
    "123456789012.dkr.ecr.us-east-1.amazonaws.com": "ecr-login"
  }
}
```

### Inspection

```bash
# View stored registries (decodes auth)
helm registry login --help

# View raw config
cat ~/.config/helm/registry/config.json | jq '.auths | keys'
```

**Warning:** The `auth` field in `config.json` contains base64-encoded credentials. While base64 is not encryption, treat this file as a secret. Use filesystem permissions (`chmod 600`).

## Troubleshooting

### "unauthorized: authentication required"

**Cause:** You are not logged in, or your credentials have expired.

**Fix:**

```bash
# Check current auth status
cat ~/.config/helm/registry/config.json | jq '.auths | keys'

# Re-authenticate
echo "$PASSWORD" | helm registry login <registry> --username <user> --password-stdin
```

### "manifest unknown: The named manifest is not known to the registry"

**Cause:** The chart version (tag) does not exist in the registry, or the path is incorrect.

**Fix:**

```bash
# Verify the exact reference
helm show chart oci://<registry>/<path> --version <version>

# List tags (use registry-specific CLI or oras/crane)
oras repo tags <registry>/<path>
```

### "x509: certificate signed by unknown authority"

**Cause:** TLS certificate is self-signed, expired, or issued by an untrusted CA.

**Fixes (in order of preference):**

1. **Use a valid certificate** (TLS termination via Let's Encrypt)

2. **Add CA to system trust store:**

```bash
# Copy CA cert to system trust store
sudo cp my-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

3. **Use `--ca-file` per command:**

```bash
helm pull oci://myregistry:443/charts/mychart --version 1.2.3 --ca-file ./my-ca.crt
```

4. **Use `HELM_REGISTRY_INSECURE=1` (dev only):**

```bash
export HELM_REGISTRY_INSECURE=1
helm pull oci://myregistry:443/charts/mychart --version 1.2.3
```

### "EOF" or "unexpected EOF"

**Cause:** Network issue, registry not reachable, or proxy misconfiguration.

**Fix:**

```bash
# Validate connectivity
curl -v https://<registry>/v2/

# Check proxy settings
env | grep -i proxy

# Set explicit proxy for Helm
export HTTP_PROXY=http://proxy:8080
export HTTPS_PROXY=http://proxy:8080
export NO_PROXY=localhost,127.0.0.1,.local,.internal
```

### "Requested access to the resource is denied" (Docker Hub)

**Cause:** The target repository does not exist on Docker Hub. Docker Hub requires explicit repository creation before pushing OCI artifacts.

**Fix:** Create the repository via Docker Hub web UI, or push a dummy container image first to auto-create the repository.

## Best Practices for OCI Chart Storage

1. **Use immutable tags.** Pin to digest-based references in production deployment pipelines:

   ```bash
   # After pushing, capture the digest
   helm pull oci://registry.example.com/charts/mychart --version 1.2.3
   # The digest is logged; store it and use it for production installs
   ```

2. **Namespace by team or project** to avoid naming collisions:
   ```
   oci://registry.example.com/team-alpha/mychart    # team-scoped
   oci://registry.example.com/product-x/mychart     # product-scoped
   ```

3. **Authenticate via credential helpers** for cloud registries instead of static credentials. Credential helpers handle automatic token refresh and are less likely to leak.

4. **Never commit `config.json`** to version control. The registry config file contains base64-encoded credentials. Add it to `.gitignore`:
   ```
   ~/.config/helm/registry/config.json
   ```

5. **Use `--password-stdin`** everywhere. Never use `--password` in scripts, CI, or interactive terminals. The stdin approach prevents shell history leakage.

6. **Set up registry retention policies** to prune old and unused chart versions. Most registries support tag-based or age-based cleanup rules.

7. **Sign charts for production.** Use Cosign or Notation to sign OCI chart artifacts:
   ```bash
   cosign sign --key cosign.key registry.example.com/charts/mychart:1.2.3
   cosign verify --key cosign.pub registry.example.com/charts/mychart:1.2.3
   ```

8. **Mirror external charts** to your internal registry. Pull public charts once and push them to your private OCI registry to avoid external rate limits and ensure availability.

9. **Verify connectivity in CI.** Add a pre-flight step before `helm push`:
   ```bash
   helm registry login <registry> --username <user> --password-stdin <<< "$PASSWORD"
   if ! helm show chart oci://<registry>/<path> --version <version> 2>/dev/null; then
     echo "Registry not accessible or version not found"
     exit 1
   fi
   ```

10. **Document registry path conventions.** Establish and document a team-wide standard for OCI chart paths so every engineer knows where to find charts and how to name new ones.
