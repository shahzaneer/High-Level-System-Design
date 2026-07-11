# Chapter 4: Repository Management

Helm repositories are the distribution mechanism for charts. This chapter covers adding, searching, maintaining, and troubleshooting repositories, including private registries, OCI support, and air-gapped environments.

---

## What Is a Helm Repository?

A Helm repository is an HTTP server that serves two things:

1. **An `index.yaml` file** — A YAML manifest listing every chart in the repository, including its name, versions, URLs to `.tgz` packages, and metadata (description, keywords, maintainers, digest).
2. **Chart `.tgz` packages** — Gzipped tar archives of chart directories.

The `index.yaml` file is the repository's table of contents. When you run `helm repo update`, Helm downloads this file and caches it locally. When you run `helm search repo`, Helm reads from the cached index.

### Anatomy of `index.yaml`

```yaml
apiVersion: v1
entries:
  nginx:
    - apiVersion: v2
      appVersion: 1.26.0
      created: "2024-01-15T10:30:00Z"
      description: NGINX web server
      digest: abc123def456...
      name: nginx
      urls:
        - https://charts.example.com/nginx-15.0.0.tgz
      version: 15.0.0
    - apiVersion: v2
      appVersion: 1.25.0
      name: nginx
      urls:
        - https://charts.example.com/nginx-14.0.0.tgz
      version: 14.0.0
```

Key observations:
- Each chart appears under its name in the `entries` map.
- Multiple versions are listed, newest first.
- The `urls` field can contain multiple URLs (mirrors).
- The `digest` is a SHA-256 hash of the chart package for integrity verification.
- The `created` timestamp is informational only.

**Exam Tip:** The `index.yaml` file lives at the root of the repository URL. If your repo is at `https://charts.example.com`, Helm fetches `https://charts.example.com/index.yaml`.

---

## `helm repo add`

Add a chart repository to your local configuration.

### Syntax

```
helm repo add [NAME] [URL] [flags]
```

### All Flags

| Flag | Type | Description |
|---|---|---|
| `--allow-deprecated-repos` | bool | Allow adding repositories that are deprecated in the index |
| `--ca-file` | string | Path to a CA certificate file for TLS verification |
| `--cert-file` | string | Path to a client certificate file for TLS authentication |
| `--force-update` | bool | Replace (overwrite) the repository if it already exists by that name |
| `--insecure-skip-tls-verify` | bool | Skip TLS certificate verification (insecure, development only) |
| `--key-file` | string | Path to a client private key file for TLS authentication |
| `--keyring` | string | Path to a GPG keyring for verifying repository signatures (default `~/.gnupg/pubring.gpg`) |
| `--no-update` | bool | Skip fetching the repository index after adding |
| `--pass-credentials` | bool | Pass HTTP basic auth credentials to all domains (use with caution) |
| `--password` | string | HTTP basic auth password |
| `--username` | string | HTTP basic auth username |

### Examples

**Public repository (no auth):**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

**Basic authentication:**

```bash
helm repo add my-private-repo https://charts.internal.example.com \
  --username admin \
  --password "s3cret-p@ss"
```

**Note:** Avoid passing passwords on the command line, as they appear in shell history. Use environment variables or a password file instead.

**Client certificate authentication:**

```bash
helm repo add secure-repo https://charts.secure.example.com \
  --cert-file /etc/helm/certs/client.crt \
  --key-file /etc/helm/certs/client.key \
  --ca-file /etc/helm/certs/ca.crt
```

**CA certificate for self-signed/internal CAs:**

```bash
helm repo add internal https://charts.corp.internal \
  --ca-file /etc/ssl/certs/corp-ca.crt
```

**Replace an existing repository:**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami --force-update
```

Without `--force-update`, attempting to add a repo with a duplicate name results in:

```
Error: repository name (bitnami) already exists, please specify a different name
```

**Skip repository index fetch (useful for large repos):**

```bash
helm repo add large-repo https://charts.large-org.com --no-update
```

**Production Note:** Always use `--ca-file` for internal repositories with custom CAs. Never use `--insecure-skip-tls-verify` in production. Rotate repository credentials regularly, using a secrets manager to inject them at runtime.

---

## `helm repo list`

Display all configured chart repositories.

### Syntax

```
helm repo list [flags]
```

### Flags

| Flag | Type | Description |
|---|---|---|
| `-o`, `--output` | string | Output format: `table` (default), `json`, `yaml` |

### Examples

**Default table output:**

```bash
helm repo list
```

```
NAME            URL
bitnami         https://charts.bitnami.com/bitnami
prometheus      https://prometheus-community.github.io/helm-charts
stable          https://charts.helm.sh/stable
```

**JSON output (for scripting):**

```bash
helm repo list -o json
```

```json
[
  {
    "name": "bitnami",
    "url": "https://charts.bitnami.com/bitnami"
  },
  {
    "name": "prometheus",
    "url": "https://prometheus-community.github.io/helm-charts"
  }
]
```

**YAML output:**

```bash
helm repo list -o yaml
```

---

## `helm repo remove`

Remove a repository from your local configuration.

```bash
helm repo remove bitnami
```

Output:

```
"bitnami" has been removed from your repositories
```

This removes the entry from `$HELM_REPOSITORY_CONFIG` (`repositories.yaml`). It does NOT delete cached index files; use `helm repo update` after removal to clean up.

---

## `helm repo update`

Fetch the latest `index.yaml` from all configured repositories and update the local cache.

### When to Use

- After adding a new repository
- Before searching for charts (`helm search repo`) to get the latest versions
- Before installing or upgrading a chart to ensure you have the latest metadata
- Periodically to refresh the local cache

### What Happens Internally

1. Helm reads `$HELM_REPOSITORY_CONFIG` (`repositories.yaml`) for all configured repositories.
2. For each repository, it sends an HTTP GET request to `<repo-url>/index.yaml`.
3. The response is cached to `$HELM_REPOSITORY_CACHE/<repo-name>-index.yaml`.
4. If a repository is unreachable, Helm prints a warning and continues to the next one.
5. Charts are NOT downloaded — only the index file is fetched.

### Flags

| Flag | Type | Description |
|---|---|---|
| `--fail-on-repo-update-fail` | bool | Exit with error if any repository update fails |

### Examples

```bash
# Update all repositories
helm repo update

# Update and fail on any errors
helm repo update --fail-on-repo-update-fail

# Update a specific repository by name (Helm 3.15+)
helm repo update bitnami
```

**Note:** `helm repo update` does not affect installed releases. It only refreshes the searchable index cache.

---

## `helm repo index`

Generate an `index.yaml` file from a directory of packaged charts (`.tgz` files). Used when hosting your own chart repository.

### Syntax

```
helm repo index [DIR] [flags]
```

### Flags

| Flag | Type | Description |
|---|---|---|
| `--merge` | string | Merge the generated index into the given index file (preserves existing entries) |
| `--url` | string | Base URL for chart download links in the generated index |

### Examples

**Generate an index from a directory of charts:**

```bash
mkdir -p ./charts
cp ~/my-charts/*.tgz ./charts/
helm repo index ./charts --url https://charts.example.com
```

This creates `./charts/index.yaml` with download URLs like:

```yaml
urls:
  - https://charts.example.com/nginx-15.0.0.tgz
```

**Merge with an existing index (add new versions without removing old ones):**

```bash
helm repo index ./charts \
  --url https://charts.example.com \
  --merge ./existing-index.yaml
```

**Production Note:** When hosting a repository on a static file server (S3, GCS, Nginx), run `helm repo index` after adding or updating chart packages. The `--url` must be the publicly accessible base URL where the `.tgz` files are served.

---

## `helm search repo`

Search the local repository cache for charts matching a keyword.

### Syntax

```
helm search repo [keyword] [flags]
```

### All Flags

| Flag | Type | Description |
|---|---|---|
| `--devel` | bool | Include pre-release versions (versions with a hyphen, e.g., `1.0.0-rc.1`) |
| `--max-col-width` | uint | Maximum column width for table output (default 50) |
| `-o`, `--output` | string | Output format: `table` (default), `json`, `yaml` |
| `-r`, `--regexp` | bool | Treat the search keyword as a regular expression |
| `-l`, `--versions` | bool | Show all available versions for each chart, not just the latest |
| `--version` | string | Filter results by a specific chart version (supports semver constraints like `>=1.0.0,<2.0.0`) |

### Examples

**Basic search:**

```bash
helm search repo nginx
```

```
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/nginx           15.0.0          1.26.0          NGINX Open Source web server
bitnami/nginx-ingress   10.0.0          1.10.0          NGINX Ingress Controller
```

**Regex search:**

```bash
helm search repo --regexp "^(bitnami|prometheus)/nginx"
```

**Show all versions:**

```bash
helm search repo nginx --versions
```

```
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/nginx           15.0.0          1.26.0          NGINX...
bitnami/nginx           14.0.0          1.25.0          NGINX...
bitnami/nginx           13.0.0          1.24.0          NGINX...
```

**Include pre-releases:**

```bash
helm search repo cert-manager --devel
```

**JSON output:**

```bash
helm search repo nginx -o json | jq '.[] | {name: .name, version: .version}'
```

**Filter by semver constraint:**

```bash
helm search repo nginx --version ">=14.0.0,<15.0.0"
```

---

## `helm search hub`

Search Artifact Hub for charts, bypassing the local repository cache.

### Syntax

```
helm search hub [keyword] [flags]
```

### Flags

| Flag | Type | Description |
|---|---|---|
| `--endpoint` | string | Artifact Hub API endpoint (default `https://artifacthub.io`) |
| `--list-repo-url` | bool | Display each chart's repository URL in the results |
| `--max-col-width` | uint | Maximum column width for table output (default 50) |
| `-o`, `--output` | string | Output format: `table` (default), `json`, `yaml` |

### Examples

```bash
# Search Artifact Hub directly
helm search hub nginx

# Show repository URLs
helm search hub nginx --list-repo-url

# JSON output for scripting
helm search hub grafana -o json | jq '.[].url'
```

**Note:** `helm search hub` queries the public Artifact Hub API. It does not require a local repository to be added and does not use cached index files. Results include charts from thousands of repositories.

---

## Custom Repositories

Any HTTP server that serves static files can function as a Helm repository. The server must serve two things:

1. `index.yaml` at the root
2. Chart `.tgz` files at the paths referenced in the index

### GitHub Pages

A common choice for public chart repositories:

1. Create a GitHub repository (e.g., `my-org/helm-charts`)
2. Enable GitHub Pages in repository settings (branch: `gh-pages`)
3. Package charts and commit them to the `gh-pages` branch or a `docs/` directory:

```bash
# Package a chart
helm package ./my-chart

# Create/update the index
helm repo index . --url https://my-org.github.io/helm-charts

# Commit and push
git checkout gh-pages
cp *.tgz index.yaml .
git add *.tgz index.yaml
git commit -m "Release my-chart 0.1.0"
git push origin gh-pages
```

4. Users add the repository:

```bash
helm repo add my-org https://my-org.github.io/helm-charts
```

### AWS S3

Use the `helm-s3` plugin for native S3 support:

```bash
helm plugin install https://github.com/hypnoglow/helm-s3

# Initialize S3 bucket as a Helm repository
helm s3 init s3://my-helm-charts

# Upload a chart
helm s3 push ./my-chart-0.1.0.tgz my-repo
```

Alternatively, use S3 static website hosting:

```bash
# Upload chart and index
aws s3 cp my-chart-0.1.0.tgz s3://my-helm-charts/
aws s3 cp index.yaml s3://my-helm-charts/

# Enable static website hosting on the bucket
# Add repository:
helm repo add my-repo http://my-helm-charts.s3-website-us-east-1.amazonaws.com
```

### Google Cloud Storage (GCS)

```bash
# Upload chart and index
gsutil cp my-chart-0.1.0.tgz gs://my-helm-charts/
gsutil cp index.yaml gs://my-helm-charts/

# Make bucket publicly readable
gsutil iam ch allUsers:objectViewer gs://my-helm-charts

# Add repository:
helm repo add my-repo https://storage.googleapis.com/my-helm-charts
```

### Any HTTP Server (Nginx Example)

```nginx
server {
    listen 80;
    server_name charts.example.com;
    root /var/www/helm-charts;

    location / {
        autoindex on;
    }

    location /index.yaml {
        add_header Content-Type "application/x-yaml; charset=utf-8";
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }
}
```

Place `index.yaml` and `.tgz` chart packages in `/var/www/helm-charts/`.

**Production Note:** The `index.yaml` file should NOT be cached aggressively. Use `Cache-Control: no-cache` or a short TTL so clients pick up new chart versions. Chart packages (`.tgz`) can be cached indefinitely since they are immutable.

---

## ChartMuseum and Harbor

### ChartMuseum

ChartMuseum is a dedicated, open-source Helm chart repository server written in Go. It supports multiple storage backends.

**Deploy with Helm:**

```bash
helm repo add chartmuseum https://chartmuseum.github.io/charts
helm install chartmuseum chartmuseum/chartmuseum \
  --set env.open.DISABLE_API=false
```

**Storage backends:** Local filesystem, AWS S3, GCS, Azure Blob Storage, OpenStack Swift, Oracle OCI Object Storage, Baidu BOS, Alibaba OSS, Tencent COS, DigitalOcean Spaces, MinIO.

**Push a chart:**

```bash
curl --data-binary "@my-chart-0.1.0.tgz" \
  http://localhost:8080/api/charts
```

Or use the `helm-push` plugin.

### Harbor

Harbor is an open-source container and Helm chart registry. It supports both OCI and traditional chart repositories natively.

- **Traditional repo URL:** `https://harbor.example.com/chartrepo/my-project`
- **OCI repo URL:** `harbor.example.com/my-project/chart-name`

Add as a traditional repository:

```bash
helm repo add harbor https://harbor.example.com/chartrepo/my-project \
  --username admin --password "password"
```

Add as an OCI registry:

```bash
helm registry login harbor.example.com
helm pull oci://harbor.example.com/my-project/nginx --version 15.0.0
```

---

## Private Repositories: Authentication Methods

### Basic Authentication

Add with username and password:

```bash
helm repo add private-repo https://charts.internal.example.com \
  --username myuser \
  --password "$MY_PASSWORD"
```

Credentials are stored in `$HELM_REPOSITORY_CONFIG` (`repositories.yaml`):

```yaml
apiVersion: ""
generated: "2024-01-01T00:00:00Z"
repositories:
  - name: private-repo
    url: https://charts.internal.example.com
    username: myuser
    password: c2VjcmV0  # base64-encoded
```

**Warning:** Passwords in `repositories.yaml` are base64-encoded, not encrypted. Anyone with read access to this file can decode the password.

### Bearer Token Authentication

Pass tokens via the `Authorization` header by using a proxy or setting environment variables. Helm does not natively support bearer token authentication for repositories outside of OCI registries. Use a reverse proxy that injects the token:

```nginx
location / {
    proxy_set_header Authorization "Bearer $TOKEN";
    proxy_pass http://backend-chart-server;
}
```

### Client Certificate (mTLS)

```bash
helm repo add secure-repo https://charts.mtls.example.com \
  --cert-file /etc/helm/tls/client.crt \
  --key-file /etc/helm/tls/client.key \
  --ca-file /etc/helm/tls/ca.crt
```

### OCI Registry Authentication

```bash
helm registry login registry.example.com \
  --username myuser \
  --password-stdin
```

Credentials are stored in `$HELM_REGISTRY_CONFIG` (default `~/.config/helm/registry/config.json`), following the Docker credential format.

---

## OCI Repositories

Helm 3.8.0+ supports OCI (Open Container Initiative) registries for storing and distributing charts. OCI registries replace the traditional `index.yaml` + HTTP server model.

### Traditional Repositories vs. OCI

| Feature | Traditional Repository | OCI Registry |
|---|---|---|
| Index mechanism | `index.yaml` file | Registry API (tag listing) |
| Storage | `.tgz` files on HTTP server | OCI-compliant registry (layers) |
| Authentication | Basic auth stored in `repositories.yaml` | Docker credential store |
| Version listing | `helm search repo` (index-based) | `helm show chart` (tag-based) |
| Push mechanism | `helm package` + upload to server | `helm push` |
| Standards | Custom Helm protocol | OCI distribution spec |
| Registries | Any HTTP server, S3, GCS | Docker Hub, ECR, GCR, ACR, Harbor |

### OCI Operations

**Login:**

```bash
helm registry login registry.example.com
```

**Push a chart to OCI:**

```bash
helm package ./my-chart
helm push ./my-chart-0.1.0.tgz oci://registry.example.com/charts
```

**Pull from OCI:**

```bash
helm pull oci://registry.example.com/charts/my-chart --version 0.1.0
```

**Install from OCI:**

```bash
helm install my-release oci://registry.example.com/charts/my-chart \
  --version 0.1.0
```

**List tags/versions of an OCI chart:**

```bash
# Requires crane or skopeo (not built into Helm)
crane ls registry.example.com/charts/my-chart
```

**Logout:**

```bash
helm registry logout registry.example.com
```

**Production Note:** OCI registries are the future of Helm chart distribution. Many organizations are migrating from traditional repositories to OCI for unified artifact management (containers + charts). OCI registries provide better security through native authentication and access control.

---

## Repository Caching

### Cache Location

Repository index files are cached at:

```
$HELM_REPOSITORY_CACHE/<repo-name>-index.yaml
```

Default: `~/.cache/helm/repository/`

### Cache Contents

```
~/.cache/helm/repository/
├── bitnami-index.yaml
├── prometheus-community-index.yaml
├── stable-index.yaml
└── my-private-repo-index.yaml
```

Each file is a complete copy of the repository's `index.yaml`.

### Cleanup

To clear the cache and force a full refresh:

```bash
rm -rf ~/.cache/helm/repository/
helm repo update
```

### Force Update

Overwrite the cached index even if Helm considers it fresh:

There is no explicit `--force` flag on `helm repo update`. To force a refresh, delete the cache first:

```bash
rm ~/.cache/helm/repository/bitnami-index.yaml
helm repo update
```

Or remove and re-add the repository:

```bash
helm repo remove bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami
```

---

## Offline / Air-Gapped Repositories

### Strategy 1: Mirror Charts with a Script

```bash
#!/bin/bash
# mirror-charts.sh
CHART_DIR="./offline-charts"
mkdir -p "$CHART_DIR"

# Pull charts you need
helm pull bitnami/nginx --version 15.0.0 -d "$CHART_DIR"
helm pull bitnami/postgresql --version 15.0.0 -d "$CHART_DIR"
helm pull prometheus-community/kube-prometheus-stack -d "$CHART_DIR"

# Generate index
helm repo index "$CHART_DIR" --url http://offline-repo.local

# Now serve $CHART_DIR via an internal HTTP server
```

### Strategy 2: Full Repository Mirror

Use tools like `charts-syncer` to mirror entire repositories:

```bash
# Install charts-syncer
wget https://github.com/bitnami/charts-syncer/releases/latest/download/charts-syncer_linux_amd64.tar.gz
tar -xzf charts-syncer_linux_amd64.tar.gz

# Configure mirroring (charts-syncer.yaml)
# source:
#   repo:
#     kind: HELM
#     url: https://charts.bitnami.com/bitnami
# target:
#   repo:
#     kind: OCI
#     url: https://harbor.internal.example.com/charts

./charts-syncer sync
```

### Strategy 3: OCI Registry Mirror

Push all required charts to an internal OCI registry:

```bash
# On internet-connected machine
helm pull bitnami/nginx --version 15.0.0
helm push nginx-15.0.0.tgz oci://harbor.internal.example.com/charts

# On air-gapped machine
helm pull oci://harbor.internal.example.com/charts/nginx --version 15.0.0
```

### Strategy 4: USB / Physical Transport

```bash
# Export: save all charts and index to a USB drive
mkdir -p /media/usb/helm-charts
helm pull bitnami/nginx -d /media/usb/helm-charts
helm repo index /media/usb/helm-charts --url file:///media/usb/helm-charts

# Import: on the air-gapped machine
# Serve the directory via a simple HTTP server:
python3 -m http.server 8080 --directory /media/usb/helm-charts &

helm repo add local http://localhost:8080
helm repo update
helm install nginx local/nginx
```

**Production Note:** Test chart mirroring pipelines regularly. Chart dependencies (subcharts listed in `Chart.yaml` dependencies) must also be mirrored or resolvable from the air-gapped environment.

---

## Repository Best Practices

### Structure

- Maintain a single repository per organization or team (not one per chart).
- Use semantic versioning for all charts.
- Never delete old chart versions from a production repository; deprecate them instead.
- Include a `README.md` and `values.yaml` in every chart to make `helm show` useful.

### `index.yaml` Management

- Always merge with `--merge` when adding new chart versions to preserve existing entries.
- Run `helm repo index` after every chart push.
- Set the `--url` flag to the publicly accessible base URL, not a local path.
- Validate the generated `index.yaml` before serving:

```bash
helm repo index ./charts --url https://charts.example.com
# Verify YAML is valid
python3 -c "import yaml; yaml.safe_load(open('./charts/index.yaml'))"
```

### Security

- Use HTTPS for all repositories in production.
- Require authentication for private repositories.
- Rotate repository credentials regularly.
- Do not store raw passwords in CI/CD scripts; use environment variables or secret managers.
- Sign charts and verify signatures with `helm verify`.
- Use OCI registries with vulnerability scanning when possible.

### CI/CD Integration

```yaml
# GitHub Actions example: publish chart on release
name: Publish Helm Chart
on:
  push:
    tags: ["chart-*"]
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Package and push
        run: |
          helm package ./charts/my-app
          helm repo index ./charts --url https://charts.example.com
          # Push chart and index to S3/GCS/GitHub Pages
```

---

## Repository Troubleshooting

### Certificate Errors

**Error:**

```
Error: looks like "https://charts.example.com" is not a valid chart repository
or cannot be reached: Get "https://charts.example.com/index.yaml":
x509: certificate signed by unknown authority
```

**Solutions:**

```bash
# Solution 1: Provide the CA certificate
helm repo add internal https://charts.example.com \
  --ca-file /path/to/internal-ca.crt

# Solution 2: For corporate proxies with SSL inspection,
# add the corporate CA to the system trust store:
# Linux:
sudo cp corp-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# macOS:
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain corp-ca.crt
```

**Warning:** Never use `--insecure-skip-tls-verify` in production. It disables all TLS verification and exposes you to man-in-the-middle attacks.

### 404 Not Found

**Error:**

```
Error: looks like "https://charts.example.com" is not a valid chart repository
or cannot be reached: failed to fetch https://charts.example.com/index.yaml : 404 Not Found
```

**Solutions:**

- Verify the URL is correct (ensure no trailing path that shouldn't be there).
- Check that `index.yaml` exists at the root of the URL:

```bash
curl -I https://charts.example.com/index.yaml
```

- For GitHub Pages, ensure the branch configured for Pages contains `index.yaml`.
- For ChartMuseum, verify the service is running and accessible.

### Authentication Failures

**Error:**

```
Error: looks like "https://private.example.com" is not a valid chart repository
or cannot be reached: failed to fetch https://private.example.com/index.yaml : 401 Unauthorized
```

**Solutions:**

- Verify credentials:

```bash
cat ~/.config/helm/repositories.yaml | grep -A5 "private.example.com"
```

- Re-add with correct credentials:

```bash
helm repo remove private-repo
helm repo add private-repo https://private.example.com \
  --username correct-user --password correct-password
```

- For OCI registries, verify login status:

```bash
cat ~/.config/helm/registry/config.json
```

- Re-authenticate:

```bash
helm registry login private.example.com
```

### Repository Index Stale / Corrupted

**Error:** `helm search repo` shows outdated versions, or chart pull fails with a 404.

**Solution:**

```bash
# Clear cache and refresh
rm ~/.cache/helm/repository/<repo-name>-index.yaml
helm repo update
```

### Slow Repository Updates

Large repositories (e.g., Bitnami with thousands of charts) can be slow to update.

**Solutions:**

- Use a subset of charts and host them in your own repository.
- Use an OCI registry for faster, tag-based operations.
- Set up a local caching proxy (e.g., Squid, Varnish) for frequently accessed repositories.

---

## Complete Command Reference

| Command | Purpose | Key Flags |
|---|---|---|
| `helm repo add [NAME] [URL]` | Add a chart repository | `--username`, `--password`, `--ca-file`, `--cert-file`, `--key-file`, `--force-update`, `--no-update`, `--insecure-skip-tls-verify`, `--pass-credentials` |
| `helm repo list` | List all configured repositories | `-o table\|json\|yaml` |
| `helm repo remove [NAME]` | Remove a repository | (none) |
| `helm repo update` | Update local index cache from all repositories | `--fail-on-repo-update-fail` |
| `helm repo index [DIR]` | Generate an index.yaml for a local directory of charts | `--url`, `--merge` |
| `helm search repo [KEYWORD]` | Search repository cache for charts | `--devel`, `--versions`, `-r`/`--regexp`, `-o`, `--version`, `--max-col-width` |
| `helm search hub [KEYWORD]` | Search Artifact Hub for charts | `--endpoint`, `--list-repo-url`, `-o`, `--max-col-width` |
| `helm registry login [HOST]` | Authenticate with an OCI registry | `--username`, `--password`, `--password-stdin` |
| `helm registry logout [HOST]` | Log out of an OCI registry | (none) |
