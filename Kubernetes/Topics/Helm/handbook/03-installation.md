# Chapter 3: Installation

This chapter covers every supported method for installing Helm on macOS, Linux, and Windows, plus post-installation configuration, tab completion, plugin management, and troubleshooting.

---

## Prerequisites

Before installing Helm, ensure the following are in place:

| Prerequisite | Minimum Version | Verification Command |
|---|---|---|
| Kubernetes cluster | v1.20+ | `kubectl version --short` |
| `kubectl` configured | v1.20+ | `kubectl cluster-info` |
| Valid kubeconfig | — | `kubectl config current-context` |

Helm communicates with the Kubernetes API server through your existing `kubeconfig` file (default `~/.kube/config`). The context defined in `KUBECONFIG` or the `HELM_KUBECONTEXT` environment variable determines which cluster Helm targets.

**Note:** Helm 3 no longer requires Tiller (the server-side component from Helm 2). Everything runs client-side using your kubeconfig credentials and RBAC permissions.

---

## macOS Installation

### Homebrew (Recommended)

The simplest and most maintainable method on macOS:

```bash
brew install helm
```

Homebrew keeps Helm updated alongside your other packages:

```bash
brew update && brew upgrade helm
```

To install a specific version:

```bash
brew install helm@3.14
```

### Manual Binary Download

1. Determine your system architecture:

```bash
uname -m
# Output: arm64 (Apple Silicon) or x86_64 (Intel)
```

2. Download the latest release:

```bash
# For Apple Silicon (M1/M2/M3)
curl -fsSL -o helm.tar.gz \
  https://get.helm.sh/helm-v3.16.0-darwin-arm64.tar.gz

# For Intel
curl -fsSL -o helm.tar.gz \
  https://get.helm.sh/helm-v3.16.0-darwin-amd64.tar.gz
```

3. Extract and install:

```bash
tar -xzf helm.tar.gz
sudo mv darwin-arm64/helm /usr/local/bin/helm
rm -rf helm.tar.gz darwin-arm64
```

4. Verify:

```bash
helm version
```

**Production Note:** Pinning a specific version via direct download is preferred in CI/CD environments and for reproducible infrastructure. Avoid `curl | bash` patterns in production pipelines.

---

## Linux Installation

### Snap (Ubuntu, Debian, Fedora, Arch)

```bash
sudo snap install helm --classic
```

The `--classic` flag is required because Helm needs access to files outside the Snap sandbox (kubeconfig, charts, plugins).

### apt (Debian/Ubuntu)

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | \
  sudo tee /usr/share/keyrings/helm.gpg > /dev/null

sudo apt-get install apt-transport-https --yes

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/helm.gpg] \
  https://baltocdn.com/helm/stable/debian/ all main" | \
  sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update
sudo apt-get install helm
```

### yum/dnf (RHEL, CentOS, Fedora, Amazon Linux)

```bash
# RHEL/CentOS 7 (yum)
sudo yum install -y helm

# Fedora / RHEL 8+ (dnf)
sudo dnf install -y helm
```

If the package is not in default repositories, add the Helm repository:

```bash
# Fedora
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo \
  https://baltocdn.com/helm/stable/rpm/helmshim.repo
sudo dnf install -y helm
```

### Binary Download (All Linux Distributions)

```bash
# Detect architecture
ARCH=$(uname -m)
case $ARCH in
  x86_64)  ARCH_SUFFIX="amd64" ;;
  aarch64) ARCH_SUFFIX="arm64"  ;;
  armv7l)  ARCH_SUFFIX="arm"    ;;
  *)       echo "Unsupported architecture: $ARCH"; exit 1 ;;
esac

HELM_VERSION="v3.16.0"
curl -fsSL -o helm.tar.gz \
  https://get.helm.sh/helm-${HELM_VERSION}-linux-${ARCH_SUFFIX}.tar.gz

tar -xzf helm.tar.gz
sudo mv linux-${ARCH_SUFFIX}/helm /usr/local/bin/helm
rm -rf helm.tar.gz linux-${ARCH_SUFFIX}
```

### Install Script (Convenience)

Helm provides an official install script. Always inspect scripts before running them:

```bash
curl -fsSL -o get_helm.sh \
  https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

# Inspect the script
less get_helm.sh

# Run it
bash get_helm.sh

# With specific version
bash get_helm.sh --version v3.16.0
```

**Warning:** Piping `curl` directly to `bash` (`curl ... | bash`) is a security risk. Always download and inspect the script first, especially in production environments.

---

## Windows Installation

### Chocolatey

```powershell
choco install kubernetes-helm
```

Upgrade:

```powershell
choco upgrade kubernetes-helm
```

### Scoop

```powershell
scoop install helm
```

### Manual Download

1. Download the Windows amd64 zip from [https://github.com/helm/helm/releases](https://github.com/helm/helm/releases)
2. Extract the zip file
3. Add the directory containing `helm.exe` to your `PATH`:

```powershell
# System-wide (Administrator)
[Environment]::SetEnvironmentVariable(
  "Path",
  [Environment]::GetEnvironmentVariable("Path", "Machine") + ";C:\helm",
  "Machine"
)

# Current user
[Environment]::SetEnvironmentVariable(
  "Path",
  [Environment]::GetEnvironmentVariable("Path", "User") + ";C:\helm",
  "User"
)
```

4. Restart PowerShell and verify:

```powershell
helm version
```

### WSL2 (Windows Subsystem for Linux)

Follow the Linux installation instructions inside your WSL2 distribution. The `kubectl` and `helm` binaries run natively within WSL2 and access the same kubeconfig.

**Note:** When using WSL2, store your kubeconfig in the WSL2 filesystem (`~/.kube/config`) rather than on a Windows-mounted drive (`/mnt/c/...`) to avoid permission issues.

---

## Binary Installation from GitHub Releases (Step by Step, All OS)

This method works identically across all platforms and is ideal for CI/CD and air-gapped environments.

### Step 1: Identify Your Platform

| OS | Architecture | Release Suffix |
|---|---|---|
| macOS Intel | amd64 | `darwin-amd64` |
| macOS Apple Silicon | arm64 | `darwin-arm64` |
| Linux (x86_64) | amd64 | `linux-amd64` |
| Linux (aarch64) | arm64 | `linux-arm64` |
| Linux (armv7) | arm | `linux-arm` |
| Windows | amd64 | `windows-amd64` |

### Step 2: Determine the Latest Version

```bash
# Using GitHub API
LATEST=$(curl -s https://api.github.com/repos/helm/helm/releases/latest | \
  grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
echo "Latest: $LATEST"

# Or visit: https://github.com/helm/helm/releases
```

### Step 3: Download the Archive

```bash
VERSION="v3.16.0"
OS="linux"        # darwin, linux, windows
ARCH="amd64"      # amd64, arm64, arm

curl -fsSL -o helm-${VERSION}-${OS}-${ARCH}.tar.gz \
  https://get.helm.sh/helm-${VERSION}-${OS}-${ARCH}.tar.gz
```

### Step 4: Verify the Checksum

Always verify the downloaded binary against the official checksums:

```bash
# Download checksum file
curl -fsSL -o helm-${VERSION}-${OS}-${ARCH}.tar.gz.sha256 \
  https://get.helm.sh/helm-${VERSION}-${OS}-${ARCH}.tar.gz.sha256

# Verify
echo "$(cat helm-${VERSION}-${OS}-${ARCH}.tar.gz.sha256)  helm-${VERSION}-${OS}-${ARCH}.tar.gz" | \
  sha256sum -c -
```

Expected output: `helm-...tar.gz: OK`

### Step 5: Extract and Install

```bash
tar -xzf helm-${VERSION}-${OS}-${ARCH}.tar.gz
sudo mv ${OS}-${ARCH}/helm /usr/local/bin/helm
rm -rf helm-${VERSION}-${OS}-${ARCH}.tar.gz ${OS}-${ARCH}
```

### Step 6: Verify Installation

```bash
helm version
```

---

## Verify Installation

### `helm version`

Check the client version and verify it connects to a cluster:

```bash
helm version
```

Example output:

```
version.BuildInfo{Version:"v3.16.0", GitCommit:"abc123...",
  GitTreeState:"clean", GoVersion:"go1.22.0"}
```

**Note:** Helm 3 is client-only. `helm version` shows only the client version. There is no server-side component to query.

### `helm env`

Display all Helm environment variables and their current values:

```bash
helm env
```

Example output:

```
HELM_BIN="helm"
HELM_CACHE_HOME="/home/user/.cache/helm"
HELM_CONFIG_HOME="/home/user/.config/helm"
HELM_DATA_HOME="/home/user/.local/share/helm"
HELM_DRIVER="secret"
HELM_KUBECONTEXT=""
HELM_NAMESPACE="default"
HELM_PLUGINS="/home/user/.local/share/helm/plugins"
HELM_REGISTRY_CONFIG="/home/user/.config/helm/registry/config.json"
HELM_REPOSITORY_CACHE="/home/user/.cache/helm/repository"
HELM_REPOSITORY_CONFIG="/home/user/.config/helm/repositories.yaml"
```

---

## Upgrade Helm

### macOS (Homebrew)

```bash
brew upgrade helm
```

### Linux (Snap)

```bash
sudo snap refresh helm
```

### Linux (apt)

```bash
sudo apt-get update && sudo apt-get install --only-upgrade helm
```

### Linux (yum/dnf)

```bash
sudo yum update helm
# or
sudo dnf upgrade helm
```

### Windows (Chocolatey)

```powershell
choco upgrade kubernetes-helm
```

### Windows (Scoop)

```powershell
scoop update helm
```

### Manual (All Platforms)

Download the new binary from GitHub Releases and replace the existing one:

```bash
curl -fsSL https://get.helm.sh/helm-v3.16.0-linux-amd64.tar.gz | tar -xz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

**Production Note:** Test upgrades in a non-production cluster first. Chart API versions (`apiVersion` in `Chart.yaml`) may change between Helm releases.

---

## Downgrade Helm

### Explicit Steps

1. Determine your current version:

```bash
helm version --short
# Output: v3.16.0
```

2. Download the desired older version from GitHub Releases:

```bash
curl -fsSL -o helm.tar.gz \
  https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
tar -xzf helm.tar.gz
```

3. Replace the existing binary:

```bash
sudo mv /usr/local/bin/helm /usr/local/bin/helm.v3.16.0.bak
sudo mv linux-amd64/helm /usr/local/bin/helm
```

4. Verify:

```bash
helm version --short
# Output: v3.14.0
```

### Warnings About Compatibility

**Warning:** Downgrading Helm can cause the following issues:

- **Release storage format changes:** Helm stores release information in Kubernetes Secrets (default) or ConfigMaps. If the older version uses a different storage schema, releases may become unreadable.
- **Chart API version incompatibility:** If you have upgraded charts to a newer `apiVersion` in `Chart.yaml`, an older Helm client may refuse to process them.
- **Plugin compatibility:** Plugins compiled against a newer Helm may break.
- **OCI registry support:** Older Helm versions may lack OCI support (`helm push`, `helm pull` to/from OCI registries).

**Exam Tip:** For CKA/CKAD exams, stick to the version pre-installed in the exam environment. Do not upgrade or downgrade.

---

## Environment Variables

Helm uses the following environment variables. Each overrides the default paths or behavior.

### Core Variables

| Variable | Default | Description |
|---|---|---|
| `HELM_BIN` | `helm` | Name or path of the Helm binary. Used by plugins that shell out to Helm. |
| `HELM_CACHE_HOME` | `~/.cache/helm` | Cache directory for repository indexes and downloaded charts. |
| `HELM_CONFIG_HOME` | `~/.config/helm` | Configuration directory containing `repositories.yaml` and registry config. |
| `HELM_DATA_HOME` | `~/.local/share/helm` | Data directory for plugins, starter packs, and release storage caches. |
| `HELM_PLUGINS` | `$HELM_DATA_HOME/plugins` | Directory where Helm plugins are installed. |
| `HELM_REGISTRY_CONFIG` | `$HELM_CONFIG_HOME/registry/config.json` | Path to the Docker/OCI registry config file (auth credentials). |
| `HELM_REPOSITORY_CACHE` | `$HELM_CACHE_HOME/repository` | Directory where repository index files are cached. |
| `HELM_REPOSITORY_CONFIG` | `$HELM_CONFIG_HOME/repositories.yaml` | Path to the file listing all added repositories. |

### Kubernetes Connectivity Variables

| Variable | Default | Description |
|---|---|---|
| `HELM_KUBECONTEXT` | (current context) | Name of the kubeconfig context Helm should use. Overrides `kubectl config current-context`. |
| `HELM_NAMESPACE` | `default` | Default namespace for Helm operations. Equivalent to `--namespace` flag. |
| `KUBECONFIG` | `~/.kube/config` | Path to the kubeconfig file (inherited from kubectl, not set by Helm). |

### Storage Driver Variables

| Variable | Default | Description |
|---|---|---|
| `HELM_DRIVER` | `secret` | Release storage backend. Valid values: `secret`, `configmap`, `memory`, `sql`. |
| `HELM_DRIVER_SQL_CONNECTION_STRING` | (none) | PostgreSQL connection string when `HELM_DRIVER=sql`. Example: `postgresql://user:pass@host:5432/helm?sslmode=disable`. |

### Detailed Explanations

**`HELM_DRIVER`** — Controls where Helm stores release information. Options:

- `secret` (default): Stores each release revision as a base64-encoded gzipped JSON blob in a Kubernetes Secret. This is the most secure option.
- `configmap`: Stores release data in ConfigMaps. Less secure (no encryption at rest), but useful for debugging since ConfigMap data is visible in plain text.
- `memory`: Stores release data in memory only. All data is lost when the Helm process exits. Useful only for testing and CI pipelines where persistence is not needed.
- `sql`: Stores release data in a PostgreSQL database. Requires `HELM_DRIVER_SQL_CONNECTION_STRING`.

**`HELM_DRIVER_SQL_CONNECTION_STRING`** — Required when `HELM_DRIVER=sql`. Example:

```bash
export HELM_DRIVER=sql
export HELM_DRIVER_SQL_CONNECTION_STRING="postgresql://helm:password@postgres.example.com:5432/helm_releases?sslmode=require"
```

The target database must exist before Helm uses it. The SQL driver creates the necessary tables automatically on first use.

**`HELM_KUBECONTEXT`** — Isolates Helm operations to a specific cluster context. Useful in multi-cluster workflows:

```bash
export HELM_KUBECONTEXT=prod-us-east
helm list --all-namespaces  # Operates only on the prod-us-east context
```

**`HELM_NAMESPACE`** — Sets a persistent default namespace to avoid repetitive `--namespace` flags:

```bash
export HELM_NAMESPACE=monitoring
helm install prometheus prometheus-community/kube-prometheus-stack
# Equivalent to: helm install --namespace monitoring prometheus ...
```

**Warning:** Setting `HELM_NAMESPACE` permanently can cause unintended operations in the wrong namespace. Consider using shell aliases or wrapper scripts instead of global environment variables for namespace management.

---

## Tab Completion Setup

Helm supports shell auto-completion for commands, flags, release names, and chart names.

### Bash

```bash
# One-time setup
helm completion bash | sudo tee /etc/bash_completion.d/helm > /dev/null

# Or per-user (add to ~/.bashrc)
source <(helm completion bash)
```

### Zsh

```bash
# One-time (if /usr/local/share/zsh/site-functions exists)
helm completion zsh | sudo tee /usr/local/share/zsh/site-functions/_helm > /dev/null

# Or per-user (add to ~/.zshrc)
source <(helm completion zsh)
```

If you get `command not found: compdef`, add this before the source line in `~/.zshrc`:

```bash
autoload -Uz compinit && compinit
```

### Fish

```bash
helm completion fish | source

# Permanent (add to ~/.config/fish/config.fish)
helm completion fish > ~/.config/fish/completions/helm.fish
```

### PowerShell

```powershell
helm completion powershell | Out-String | Invoke-Expression

# Permanent (add to $PROFILE)
helm completion powershell >> $PROFILE
```

Restart your shell or source the profile after adding completion.

---

## Plugin Installation

### Installing a Plugin

```bash
helm plugin install <url|path>
```

Examples:

```bash
# From a Git repository
helm plugin install https://github.com/databus23/helm-diff

# From a local directory
helm plugin install ./my-custom-plugin
```

### Listing Installed Plugins

```bash
helm plugin list
```

Example output:

```
NAME            VERSION     DESCRIPTION
diff            3.9.0       Preview helm upgrade changes as a diff
secrets         4.6.0       Manage secrets with helm
```

### Updating Plugins

```bash
helm plugin update <plugin-name>
```

### Removing a Plugin

```bash
helm plugin uninstall <plugin-name>
```

### Popular Helm Plugins

| Plugin | Purpose | Install Command |
|---|---|---|
| `helm-diff` | Shows a colored diff of what a `helm upgrade` would change before applying it | `helm plugin install https://github.com/databus23/helm-diff` |
| `helm-secrets` | Encrypt/decrypt `values.yaml` files using SOPS, AWS KMS, GCP KMS, or age | `helm plugin install https://github.com/jkroepke/helm-secrets` |
| `helmfile` | Declarative spec for deploying Helm charts across environments | `helm plugin install https://github.com/helmfile/helmfile` |
| `helm-unittest` | Unit tests for Helm chart templates using YAML test definitions | `helm plugin install https://github.com/helm-unittest/helm-unittest` |
| `helm-mapkubeapis` | Maps deprecated Kubernetes APIs in release manifests to supported versions | `helm plugin install https://github.com/helm/helm-mapkubeapis` |
| `helm-push` | Push charts to OCI registries, ChartMuseum, and S3 | `helm plugin install https://github.com/chartmuseum/helm-push` |
| `helm-s3` | Manage Helm repositories on AWS S3 | `helm plugin install https://github.com/hypnoglow/helm-s3` |

**Production Note:** Verify plugin authenticity before installation. Plugins run with the same privileges as the Helm binary and have access to your kubeconfig.

---

## Managing Multiple Helm Versions

### Using asdf (Recommended)

Install asdf and the Helm plugin:

```bash
# Install asdf (if not installed)
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0

# Add Helm plugin
asdf plugin add helm
```

Install and switch versions:

```bash
# List available versions
asdf list all helm

# Install a specific version
asdf install helm 3.16.0

# Set global default
asdf global helm 3.16.0

# Set per-project (creates .tool-versions)
asdf local helm 3.14.0
```

### Using a Version Manager Wrapper

Create a simple wrapper script:

```bash
#!/bin/bash
# ~/bin/helm-wrapper
VERSION_FILE=".helm-version"

if [ -f "$VERSION_FILE" ]; then
    VERSION=$(cat "$VERSION_FILE")
    exec "/usr/local/bin/helm-${VERSION}" "$@"
else
    exec /usr/local/bin/helm "$@"
fi
```

Keep multiple binaries:

```bash
/usr/local/bin/helm         # latest
/usr/local/bin/helm-v3.14.0 # pinned older version
/usr/local/bin/helm-v3.16.0 # newer version
```

---

## Troubleshooting Installation Issues

### `helm: command not found`

The binary is not in your `PATH`. Verify the installation:

```bash
which helm
echo $PATH
ls -la /usr/local/bin/helm
```

Fix: Ensure `/usr/local/bin` is in `PATH`, or symlink the binary.

### `Error: Kubernetes cluster unreachable`

Helm cannot connect to the cluster. Check:

```bash
kubectl cluster-info
kubectl config current-context
echo $KUBECONFIG
```

**Common causes:**
- kubeconfig file is missing or misconfigured
- VPN is disconnected (for remote clusters)
- Cluster API server is down
- Context points to a deleted cluster

### `Error: could not find tiller`

You are trying to use Helm 2 commands with Helm 3, or your `kubectl` is configured for a Helm 2 cluster. Helm 3 does not use Tiller. Ensure you are running Helm 3:

```bash
helm version
```

### Permission Denied When Moving Binary

```bash
# Use sudo or ensure you own the target directory
sudo mv linux-amd64/helm /usr/local/bin/helm
sudo chmod +x /usr/local/bin/helm
```

### macOS Gatekeeper Blocks Binary

If macOS blocks the downloaded binary:

```bash
# Remove quarantine attribute
xattr -d com.apple.quarantine /usr/local/bin/helm
```

Or allow it in **System Preferences > Security & Privacy > General**.

### Checksum Verification Failure

If `sha256sum -c` fails, the download is corrupted or tampered with:

```bash
# Re-download
rm helm-*.tar.gz*
curl -fsSL -o helm.tar.gz https://get.helm.sh/helm-v3.16.0-linux-amd64.tar.gz
# Re-verify
```

### SSL/TLS Certificate Errors

When adding repositories with self-signed certificates:

```bash
# Option 1: Add CA certificate
helm repo add --ca-file /path/to/ca.crt myrepo https://myrepo.example.com

# Option 2: Skip TLS verification (insecure, development only)
helm repo add --insecure-skip-tls-verify myrepo https://myrepo.example.com
```

---

## Verification Checklist After Installation

Run through this checklist to confirm a working Helm installation:

| # | Check | Command | Expected Result |
|---|---|---|---|
| 1 | Binary exists | `which helm` | `/usr/local/bin/helm` |
| 2 | Binary is executable | `helm version` | Version info printed |
| 3 | Kubernetes connectivity | `helm ls` | Lists releases or is empty |
| 4 | Default repos | `helm repo list` | Shows configured repos |
| 5 | Can add a repo | `helm repo add bitnami https://charts.bitnami.com/bitnami` | `"bitnami" has been added` |
| 6 | Can search repos | `helm search repo bitnami/nginx` | Shows chart results |
| 7 | Can install a chart | `helm install test-release bitnami/nginx --dry-run` | Prints rendered manifests |
| 8 | Tab completion works | Type `helm ` and press Tab | Suggestions appear |
| 9 | Environment variables are correct | `helm env` | Defaults or custom values shown |
| 10 | Uninstall test release | `helm uninstall test-release` (if installed) | Release removed |

### Automated Verification Script

```bash
#!/bin/bash
# verify-helm.sh
set -e

echo "=== Helm Installation Verification ==="

echo -n "1. Binary location: "
which helm

echo -n "2. Version check: "
helm version --short

echo -n "3. Cluster connectivity: "
helm ls > /dev/null 2>&1 && echo "OK" || echo "FAILED"

echo -n "4. Plugin list: "
helm plugin list

echo "=== Verification Complete ==="
```

**Production Note:** Incorporate this verification into your CI/CD pipeline or infrastructure provisioning scripts to catch installation issues early.
