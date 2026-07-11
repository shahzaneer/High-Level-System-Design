# Chapter 16: Security

Helm security spans chart signing, secrets management, repository authentication, RBAC, supply chain integrity, vulnerability scanning, and compliance. This chapter covers every security dimension of the Helm ecosystem.

---

## 16.1 Chart Signing — PGP/GPG Provenance

Helm supports cryptographically signing chart packages using PGP/GPG. A signed chart produces an associated **provenance file** (`.prov`) that verifies the chart's origin and integrity — essential for supply chain security in regulated environments.

### 16.1.1 How Chart Signing Works

```
1. Chart author generates a GPG key pair (public + private)
2. Chart author signs the .tgz package with helm package --sign
3. A .prov file is generated alongside the .tgz
4. Chart consumer imports the author's public key into their keyring
5. Chart consumer runs helm verify to validate the signature
6. Helm confirms: the chart was signed by the trusted key AND the chart has not been modified
```

The provenance file is a detached signature. Helm uses OpenPGP's cleartext signature framework — the `.prov` file contains the SHA-256 digest of the chart package plus the PGP signature of that digest.

**Warning:** Chart signing only verifies the integrity and origin of a chart *package*. It does NOT verify the security of the chart's content, its templates, or the images it references. Combine signing with vulnerability scanning and policy enforcement.

### 16.1.2 Key Pair Generation

```bash
gpg --full-generate-key
```

Interactive prompts:
- Kind of key: `RSA and RSA` (default)
- Key size: `4096` (minimum recommended; 2048 acceptable)
- Expiration: `0` (does not expire) or a specific period like `2y`
- Real name: `helm-charts@example.com`
- Email: `helm-charts@example.com`
- Comment: `Helm chart signing key for Example Corp`
- Passphrase: enter a strong passphrase (required for signing)

**Production Note:** Store the private key in a secure, access-controlled location (hardware security module, secrets manager, or encrypted vault). Never commit private keys to version control. Rotate signing keys annually.

**Exam Tip:** The CKA does not test GPG key generation. Understand the concepts — what `.prov` files are, how `helm verify` works, and why signing matters.

### 16.1.3 Listing and Exporting Keys

```bash
gpg --list-keys                          # List public keys
gpg --list-secret-keys                   # List private keys
gpg --export --armor helm-charts@example.com > public-key.asc
gpg --export-secret-key --armor helm-charts@example.com > private-key.asc  # NEVER share
```

Distribute the public key to all users who need to verify charts. Import it on their machines:

```bash
gpg --import public-key.asc
```

### 16.1.4 Creating a Signed Package — `helm package --sign`

| Flag | Type | Description |
|---|---|---|
| `--sign` | bool | Sign the chart package with GPG |
| `--key` | string | The name (email or key ID) of the GPG key to use for signing |
| `--keyring` | string | Path to the GPG keyring file (default `~/.gnupg/secring.gpg` on older GPG; `~/.gnupg/pubring.kbx` on GPG 2.1+) |
| `--passphrase-file` | string | Path to a file containing the GPG passphrase (non-interactive; use in CI/CD) |

**Basic signing:**

```bash
helm package --sign --key 'helm-charts@example.com' --keyring ~/.gnupg/pubring.kbx ./mychart
```

Output:
```
Successfully packaged chart and saved it to: /path/to/mychart-1.0.0.tgz
Signed by: helm-charts@example.com
```

This creates two files:
- `mychart-1.0.0.tgz` — the chart package
- `mychart-1.0.0.tgz.prov` — the provenance file with the digital signature

**CI/CD non-interactive signing:**

```bash
echo "$GPG_PASSPHRASE" > /tmp/passphrase.txt
helm package --sign \
  --key 'helm-charts@example.com' \
  --keyring /secure/path/pubring.kbx \
  --passphrase-file /tmp/passphrase.txt \
  ./mychart
rm -f /tmp/passphrase.txt
```

**Note:** On GPG 2.1+, the keyring format changed from `pubring.gpg` / `secring.gpg` to `.kbx` files. If `--keyring` is not specified, Helm uses GPG's default keyring. GPG 2.1+ may require setting `--pinentry-mode loopback` via environment variable `GPG_TTY` and `GPG_OPTS` for non-interactive use.

---

## 16.2 Verifying Signed Charts — `helm verify`

`helm verify` validates the provenance (`.prov`) file against the chart package and its signing key.

### 16.2.1 Verification Process

```
1. Helm reads the .prov file
2. Helm computes SHA-256 of the .tgz chart package
3. Helm verifies the PGP signature in .prov against:
   a. The computed hash
   b. The public key(s) in the keyring
4. If signature is valid and hash matches → chart is verified
5. If signature is invalid, expired, or key is not trusted → verification fails
```

### 16.2.2 Command and Flags

| Flag | Type | Description |
|---|---|---|
| `--keyring` | string | Path to the GPG keyring containing trusted public keys (default `~/.gnupg/pubring.gpg`) |

```bash
helm verify mychart-1.0.0.tgz
```

Successful output:
```
Signed by: helm-charts@example.com
Using Key with fingerprint: AAAA1111BBBB2222CCCC3333DDDD4444EEEE5555
Chart Hash Verified: sha256:abc123def456...
```

Failed output (tampered chart):
```
Error: Failed to verify chart: chart has been modified and the signature is no longer valid
```

Failed output (untrusted key):
```
Error: Failed to verify chart: no public key found for helm-charts@example.com
```

### 16.2.3 Anatomy of a `.prov` File

A provenance file contains:

```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA256

name: mychart
version: 1.0.0
description: A sample Helm chart
apiVersion: v2
...
files:
  mychart-1.0.0.tgz: sha256:abc123...
-----BEGIN PGP SIGNATURE-----
... (binary PGP signature data) ...
-----END PGP SIGNATURE-----
```

| Section | Content |
|---|---|
| Header | Hash algorithm (`SHA256`) |
| Metadata block | Chart name, version, description, apiVersion |
| Files block | Filename and SHA-256 hash of every file in the chart |
| Signature block | PGP detached signature over the metadata + files block |

**Production Note:** Always verify signed charts before deployment. Integrate `helm verify` into your CI/CD pipeline as a gate before `helm install` or `helm upgrade`. Reject any chart that fails verification.

---

## 16.3 Key Management

### 16.3.1 Key Generation Best Practices

| Practice | Recommendation |
|---|---|
| Key algorithm | RSA 4096-bit or Ed25519 (ECDSA) |
| Passphrase | Strong (20+ characters, mix of types) or no passphrase with HSM-backed keys |
| Expiration | 1–2 years with rotation policy |
| Subkeys | Use separate signing subkey (not master key) for daily signing |
| Storage | Store private key in Vault / AWS KMS / HSM; never on developer laptops |
| Keyring file | Distribute only the public keyring; never share the secret keyring |

### 16.3.2 Key Distribution and Trust

For organizations, maintain a central public keyring:

```bash
gpg --export --armor helm-charts@example.com > helm-public-key.asc
gpg --export --armor helm-charts-backup@example.com >> helm-public-key.asc
```

Distribute `helm-public-key.asc` to all CI/CD runners and developer machines. Import:

```bash
gpg --import helm-public-key.asc
```

Store the public keyring in a Git repository (it is public, non-sensitive). Pin the expected key fingerprint in your CI/CD pipeline to prevent key substitution attacks.

---

## 16.4 Secrets in Helm Charts

### 16.4.1 The Golden Rule

**Warning:** NEVER put secrets in `values.yaml`. NEVER hardcode credentials in templates. NEVER commit secrets to Git. Helm values files are plaintext YAML — anyone with repository access can read them. Release secrets are stored unencrypted in Kubernetes Secrets (base64 is encoding, not encryption).

### 16.4.2 What NOT to Do

```yaml
# ✗ DO NOT DO THIS — values.yaml
database:
  password: "SuperSecret123"
apiKey: "sk-abc123def456"
tls:
  cert: |
    -----BEGIN CERTIFICATE-----
    ...
```

```yaml
# ✗ DO NOT DO THIS — templates
apiVersion: v1
kind: Secret
data:
  password: {{ .Values.database.password | b64enc }}
```

Both patterns embed secrets in Git history forever, even if you later remove them.

---

## 16.5 Secrets Management Strategies

### 16.5.1 Strategy Comparison

| Strategy | Encrypt at Rest? | Git-Safe? | Complexity | Best For |
|---|---|---|---|---|
| External Secrets Operator | Yes (in provider) | Yes | Medium | AWS/GCP/Azure-native shops |
| Sealed Secrets | Yes | Yes | Low | Simple GitOps workflows |
| SOPS + helm-secrets | Yes | Yes | Medium | Multi-cloud, age/GPG/KMS |
| HashiCorp Vault (sidecar) | Yes | Yes | High | Enterprise with existing Vault |
| Cloud Secrets Manager direct | Yes (in provider) | Yes | Medium | Single-cloud deployments |
| External Secrets (initContainer) | Yes | Yes | Low | Legacy apps, no operator allowed |

### 16.5.2 External Secrets Operator (ESO)

ESO synchronizes secrets from external providers into Kubernetes Secrets.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-secret
  data:
    - secretKey: password
      remoteRef:
        key: prod/database
        property: password
```

The Helm chart references the Kubernetes Secret normally — ESO keeps it synchronized:

```yaml
# templates/deployment.yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

**Production Note:** ESO requires cluster-wide privileges to create `SecretStore` and `ClusterSecretStore` resources. This is appropriate — you want a single, auditable secrets controller rather than granting every Helm release secret-access permissions.

### 16.5.3 Sealed Secrets

Sealed Secrets encrypt a Kubernetes Secret into a `SealedSecret` CRD that is safe to commit to Git. Only the cluster-side controller can decrypt it.

```bash
kubeseal --controller-name sealed-secrets --controller-namespace kube-system \
  --format yaml < raw-secret.yaml > sealed-secret.yaml
```

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
spec:
  encryptedData:
    password: AgBy8hCkJiTqF...long encrypted blob...
```

Deploy the `SealedSecret` alongside your Helm chart (as a pre-install hook or separate manifest). The controller creates the corresponding Kubernetes Secret automatically.

**Exam Tip:** Know that Sealed Secrets are cluster-specific. A `SealedSecret` sealed on cluster A cannot be decrypted on cluster B unless you share the sealing key.

### 16.5.4 SOPS + helm-secrets Plugin

`helm-secrets` is the most widely used plugin for secret management in Helm. It integrates with [Mozilla SOPS](https://github.com/getsops/sops) to encrypt entire values files.

#### Installation

```bash
helm plugin install https://github.com/jkroepke/helm-secrets
```

SOPS must also be installed:

```bash
# macOS
brew install sops

# Linux
curl -LO https://github.com/getsops/sops/releases/download/v3.9.0/sops-v3.9.0.linux.amd64
sudo mv sops-v3.9.0.linux.amd64 /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops
```

#### SOPS Configuration

`.sops.yaml` (at repo root):

```yaml
creation_rules:
  - path_regex: .*secrets.*\.yaml$
    age: age1abc123... # or pgp: FINGERPRINT, or use KMS ARN
  - path_regex: .*staging.*
    kms: arn:aws:kms:us-east-1:123456789012:key/abc-def-123
  - path_regex: .*prod.*
    kms: arn:aws:kms:us-east-1:123456789012:key/prod-key-id
```

SOPS supports multiple encryption backends:
| Backend | Description |
|---|---|
| `age` | Modern, simple encryption (recommended for new setups) |
| `pgp` | GPG key encryption |
| `aws_kms` | AWS KMS key encryption |
| `gcp_kms` | GCP Cloud KMS encryption |
| `azure_kv` | Azure Key Vault encryption |
| `hc_vault` | HashiCorp Vault transit encryption |

#### Encrypting and Decrypting Values

```bash
# Create a secret values file, edit it, and encrypt it
helm secrets enc secrets.yaml

# Edit an already encrypted file (decrypt → edit → re-encrypt)
helm secrets edit secrets.yaml

# Decrypt (for inspection; be careful not to commit decrypted output)
helm secrets dec secrets.yaml

# View decrypted content without writing to disk
helm secrets view secrets.yaml
```

#### Using Encrypted Values with Helm

```bash
helm secrets upgrade --install myapp ./mychart -f secrets.yaml -f values.yaml
```

The `helm secrets` wrapper command temporarily decrypts the secrets file in memory, merges it with other values, passes the merged values to Helm, and never writes decrypted content to disk.

**Typical values file structure:**

```
values.yaml          # non-sensitive defaults (safe to commit)
secrets.yaml         # encrypted secrets (safe to commit after encryption)
values-prod.yaml     # environment-specific overrides (non-sensitive)
secrets-prod.yaml    # environment-specific secrets (encrypted)
```

**Note:** `helm secrets` does not integrate directly with Helm's `--set` flag for secret values. Use encrypted values files exclusively for secrets — never pass secrets via `--set`.

#### Workflow Example

```bash
# 1. Developer creates a new secret
helm secrets edit secrets.yaml
#   (editor opens; developer adds the secret value in plaintext)
#   (SOPS encrypts on save)

# 2. Developer commits the encrypted file
git add secrets.yaml
git commit -m "Add database credentials"

# 3. CI/CD deploys using helm-secrets
helm secrets upgrade --install myapp ./mychart -f values.yaml -f secrets.yaml

# 4. Secrets are decrypted in-memory, passed to Helm, and never touch disk on CI runner
```

**Production Note:** In CI/CD, SOPS needs access to the decryption key. Use cloud KMS (AWS/GCP/Azure) or a CI/CD secret variable containing the age/GPG private key. Never use `--backend=pgp` with a key that has a passphrase unless you have automated passphrase injection.

---

## 16.6 Private Repository Authentication

### 16.6.1 Authentication Methods

| Method | Flag / Mechanism | Use Case |
|---|---|---|
| HTTP Basic Auth | `--username`, `--password` | Simple internal repos |
| TLS Client Certificates | `--cert-file`, `--key-file` | Mutual TLS (mTLS) |
| Bearer Token | `--ca-file` with token injection | Service-account-based access |
| OCI Registry Auth | `helm registry login` | OCI-compliant registries |
| AWS ECR Token | `aws ecr get-login-password \| helm registry login` | ECR |
| GCP Artifact Registry | `gcloud auth print-access-token \| helm registry login` | GAR |
| Azure ACR | `helm registry login --username 00000000-...` | ACR |

### 16.6.2 HTTP Basic Auth

```bash
# Preferred: environment variable (avoids shell history leak)
export HELM_REPO_USERNAME="admin"
export HELM_REPO_PASSWORD="s3cret-p@ss"
helm repo add myrepo https://charts.internal.example.com

# CI/CD: inject credentials from secret variables
helm repo add myrepo https://charts.internal.example.com \
  --username "$REPO_USERNAME" \
  --password "$REPO_PASSWORD"
```

### 16.6.3 mTLS Client Certificate Auth

```bash
helm repo add secure-repo https://charts.secure.example.com \
  --cert-file /etc/helm/certs/client.crt \
  --key-file /etc/helm/certs/client.key \
  --ca-file /etc/helm/certs/ca.crt
```

### 16.6.4 OCI Registry Authentication

```bash
helm registry login registry.example.com \
  --username myuser \
  --password "$REGISTRY_PASSWORD"

helm registry login oci://registry.example.com
```

All OCI operations (`push`, `pull`, `install` from OCI) use the credentials stored by `helm registry login`. Credentials are stored in the credential store configured by the `HELM_REGISTRY_CONFIG` environment variable (default `~/.config/helm/registry/config.json`).

**Multi-registry authentication:**

```bash
helm registry login registry1.example.com --username u1 --password "$P1"
helm registry login registry2.example.com --username u2 --password "$P2"
```

Helm matches credentials by registry hostname. No additional configuration is needed.

**Credential helpers (ECR, GCR, ACR):**

```bash
# Docker credential helpers work transparently
# ~/.docker/config.json with credHelpers:
{
  "credHelpers": {
    "123456789012.dkr.ecr.us-east-1.amazonaws.com": "ecr-login",
    "us-central1-docker.pkg.dev": "gcloud"
  }
}

# Helm OCI reads these helpers automatically
helm push ./mychart-1.0.0.tgz oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/
```

---

## 16.7 RBAC for Helm

### 16.7.1 What Permissions Helm Needs

Helm's client (`helm install`, `helm upgrade`, etc.) uses the kubeconfig credentials to communicate with the Kubernetes API server. The permissions required depend on the resources being deployed.

| Operation | Minimum Permissions |
|---|---|
| `helm install` | Create access to all resource types in the chart (Deployments, Services, ConfigMaps, Secrets, etc.) |
| `helm upgrade` | Create + Update + Patch |
| `helm rollback` | Create + Update + Patch + Delete (old revision cleanup) |
| `helm uninstall` | Delete access to all resource types in the release |
| `helm list` | List access to Secrets (Helm stores release info in Secrets) |
| `helm history` | Get access to Secrets |
| `helm test` | Get + List on Pods, Jobs |

If a chart includes cluster-scoped resources (ClusterRole, ClusterRoleBinding, PersistentVolume, Namespace), the Helm service account requires cluster-scoped permissions.

### 16.7.2 Principle of Least Privilege — Namespace-Scoped Helm RBAC

**Scenario:** CI/CD needs to deploy Helm charts to the `staging` namespace only.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm-deployer
  namespace: staging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: helm-deployer
  namespace: staging
rules:
  - apiGroups: ["", "apps", "batch", "extensions", "networking.k8s.io", "autoscaling", "policy"]
    resources:
      - deployments
      - replicasets
      - pods
      - services
      - configmaps
      - secrets
      - persistentvolumeclaims
      - ingresses
      - jobs
      - cronjobs
      - horizontalpodautoscalers
      - poddisruptionbudgets
      - serviceaccounts
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: helm-deployer
  namespace: staging
subjects:
  - kind: ServiceAccount
    name: helm-deployer
    namespace: staging
roleRef:
  kind: Role
  name: helm-deployer
  apiGroup: rbac.authorization.k8s.io
```

### 16.7.3 Cluster-Scoped Helm RBAC

For charts deploying cluster-wide resources or managing releases across multiple namespaces:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm-cluster-deployer
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: helm-cluster-deployer
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: helm-cluster-deployer
subjects:
  - kind: ServiceAccount
    name: helm-cluster-deployer
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: helm-cluster-deployer
  apiGroup: rbac.authorization.k8s.io
```

**Warning:** `resources: ["*"]` and `verbs: ["*"]` grants unrestricted access. This is equivalent to cluster-admin. For production, enumerate specific resource types and restrict verbs. Use tools like `audit2rbac` or `rakkess` to discover the minimal permissions a chart needs.

### 16.7.4 Resource-Specific RBAC Example

If a chart only deploys Deployments, Services, and ConfigMaps:

```yaml
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "create"]   # Helm stores release info in Secrets
```

---

## 16.8 Supply Chain Security

### 16.8.1 Verifying Chart Sources

| Trust Signal | How to Verify |
|---|---|
| Artifact Hub Verified Publisher | Look for the blue checkmark badge on Artifact Hub |
| Official Repository | Charts from `https://helm.sh/charts` or maintainer-owned domains |
| Signed Charts | Verify with `helm verify` and a trusted keyring |
| OCI Signature (Cosign) | Verify with `cosign verify` on OCI artifacts |
| Chart Source Code | Check the `sources` field in `Chart.yaml`; verify the repo is the publisher's |
| Maintainer Email | Correlate maintainer emails with known organization domains |
| Provenance File | Check that every `helm package --sign` produces a `.prov` file |

**Production Note:** Always prefer charts from verified publishers on Artifact Hub. Pin chart versions with explicit SemVer — never use version ranges or `latest`. Mirror third-party charts to your internal registry to prevent supply-chain changes from breaking deployments.

### 16.8.2 OCI Artifact Signing with Cosign / Sigstore

Cosign signs OCI artifacts (including Helm charts stored in OCI registries) using keyless signing (Sigstore Fulcio + Rekor) or key-based signing.

**Key-based signing:**

```bash
# Generate a key pair
cosign generate-key-pair

# Push the chart to OCI
helm push ./mychart-1.0.0.tgz oci://registry.example.com/charts

# Sign the pushed chart
cosign sign --key cosign.key registry.example.com/charts/mychart:1.0.0

# Verify the signed chart
cosign verify --key cosign.pub registry.example.com/charts/mychart:1.0.0
```

**Keyless signing (OIDC-based):**

```bash
cosign sign registry.example.com/charts/mychart:1.0.0
# Redirects to browser for OIDC authentication
# Signature is stored in Rekor transparency log
```

**Verification in CI/CD:**

```bash
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity "https://github.com/myorg/myrepo/.github/workflows/release.yml@refs/heads/main" \
  registry.example.com/charts/mychart:1.0.0
```

### 16.8.3 Security Scanning

**Scanning chart packages for vulnerabilities:**

```bash
# Trivy — scan a chart archive
trivy fs ./mychart-1.0.0.tgz

# Trivy — scan a rendered manifest
helm template ./mychart | trivy config -
```

```bash
# Grype — scan a chart
grype dir:./mychart

# Grype — scan a rendered manifest
helm template ./mychart > rendered.yaml && grype rendered.yaml
```

**What scanners check:**
- Container image references in templates (vulnerable base images)
- Misconfigurations in rendered manifests (privileged containers, hostPath mounts, missing security contexts)
- Use of deprecated or vulnerable Kubernetes API versions
- Hardcoded credentials in templates (pattern matching)

**Production Note:** Integrate scanning into your CI/CD pipeline. Reject charts that contain CRITICAL or HIGH vulnerabilities. Use `trivy config --severity CRITICAL,HIGH --exit-code 1` to fail the pipeline.

---

## 16.9 Helm Tiller Security (Helm 2 Historical Context)

### 16.9.1 Why Tiller Was Removed

In Helm 2, the architecture required a server-side component called **Tiller** running inside the cluster. Tiller:
- Ran as a Deployment with a ServiceAccount
- Had full cluster-admin privileges by default
- Accepted gRPC requests from any authenticated Helm client
- Managed release lifecycle (install, upgrade, rollback, delete)
- Stored release history in ConfigMaps (Helm 2) or Secrets

**Critical security problems with Tiller:**
1. **Broad attack surface:** Tiller exposed a gRPC port inside the cluster. Any pod or compromised workload could communicate with Tiller and deploy arbitrary resources.
2. **No fine-grained RBAC:** Tiller's single ServiceAccount determined permissions for ALL releases. If Tiller had cluster-admin, any Helm user could deploy cluster-admin resources.
3. **No authentication between Helm client and Tiller:** By default, any client that could reach the Tiller endpoint could deploy.
4. **TLS configuration was opt-in and complex:** Mutual TLS between client and Tiller was available but rarely configured correctly.

### 16.9.2 Helm 3 Fixes

Helm 3 removed Tiller entirely. The Helm client now communicates directly with the Kubernetes API server, using the user's kubeconfig credentials. This means:

- Helm inherits the user's Kubernetes RBAC permissions — no extra privileged service account
- No server-side component to attack
- No gRPC port exposed inside the cluster
- No additional TLS configuration needed
- Release history is stored as Secrets (encrypted at rest in etcd)
- Audit logging captures Helm operations as standard Kubernetes API calls

**Exam Tip:** CKA exams use Helm 3 exclusively. Understand that Tiller was the server-side component in Helm 2 and was removed for security reasons. The CKA may ask about Helm 3's client-only architecture as a security improvement.

---

## 16.10 Network Security

Helm communicates exclusively with the Kubernetes API server over HTTPS. No additional ports or protocols are involved.

```
Helm Client → (HTTPS over TCP 6443) → Kubernetes API Server
```

**Recommendations:**
- Always use HTTPS (never `--insecure-skip-tls-verify` in production)
- Use a properly signed TLS certificate on the API server (internal CA or Let's Encrypt)
- Restrict API server access with network policies or firewall rules
- Use `--kube-apiserver` flag to pin to a specific API server endpoint
- In air-gapped environments, Helm works entirely offline once charts are fetched

---

## 16.11 Audit Logging

Helm operations appear in Kubernetes audit logs as standard API server requests.

| Helm Command | Audit Event Type | Resources Audited |
|---|---|---|
| `helm install` | `create` events | All resources in the release (Deployment, Service, Secret, etc.) |
| `helm upgrade` | `create` + `update` + `patch` events | Modified resources |
| `helm rollback` | `update` + `delete` events | Reverted resources |
| `helm uninstall` | `delete` events | All resources in the release |
| `helm list` | `get` + `list` events | Secrets (release storage) |
| `helm history` | `get` events | Secrets |
| `helm get values` | `get` events | Secrets |

The audit log user identity matches the kubeconfig user (ServiceAccount, human user, or OIDC identity). This means Helm operations are traceable to specific users or CI/CD systems.

**Production Note:** Enable Kubernetes audit logging with at least the `Metadata` level. Use an audit policy that logs all `create`, `update`, `delete`, and `patch` operations on `secrets` (where Helm stores release metadata). Forward audit logs to a SIEM for long-term retention and alerting.

---

## 16.12 Compliance Considerations

### 16.12.1 SOC2 / HIPAA / PCI-DSS

Helm itself is not subject to compliance frameworks — it is a deployment tool. However, the *way* you use Helm affects compliance posture.

| Requirement | Helm-Specific Consideration |
|---|---|
| **Access Control** | Use RBAC scoped to namespaces; audit user and ServiceAccount access |
| **Change Management** | All deployments through version-controlled charts and values; use pull requests |
| **Audit Trail** | Kubernetes audit logs track every `helm install`/`upgrade`/`rollback`; retain logs per compliance period |
| **Secrets Management** | Never store secrets in chart values; use External Secrets, Sealed Secrets, or SOPS |
| **Integrity** | Sign charts with GPG or Cosign; verify signatures before deployment |
| **Vulnerability Management** | Scan charts and rendered manifests with Trivy/Grype; remediate within SLA |
| **Least Privilege** | Create namespace-scoped ServiceAccounts for Helm in CI/CD with minimal verbs |
| **Separation of Duties** | Separate chart development permissions from deployment permissions |
| **Configuration Hardening** | Use Pod Security Standards (restricted); enforce via Kyverno or OPA Gatekeeper on rendered manifests |

### 16.12.2 Vulnerability Disclosure

To report security vulnerabilities in Helm itself:
- Email: `cncf-helm-security@lists.cncf.io`
- Do NOT open a public GitHub issue for security vulnerabilities
- Helm follows the CNCF coordinated vulnerability disclosure process
- Security advisories are published at https://github.com/helm/helm/security/advisories

For vulnerabilities in a specific Helm chart:
- Contact the chart maintainer (listed in `Chart.yaml` under `maintainers`)
- Use the repository's security disclosure process (e.g., GitHub Security Advisories)

---

## 16.13 Security Best Practices Checklist

```
☐ 1.  RUN HELM 3 ONLY — Helm 2 is end-of-life; Tiller is a security risk
☐ 2.  SIGN ALL PRODUCTION CHARTS with GPG or Cosign
☐ 3.  VERIFY SIGNATURES before every deployment (helm verify or cosign verify)
☐ 4.  NEVER PUT SECRETS IN values.yaml
☐ 5.  NEVER HARDCODE CREDENTIALS in templates
☐ 6.  NEVER USE --set FOR SECRET VALUES
☐ 7.  USE EXTERNAL SECRETS OPERATOR, Sealed Secrets, or SOPS/helm-secrets for secrets
☐ 8.  CREATE DEDICATED SERVICE ACCOUNTS for Helm in CI/CD (not cluster-admin)
☐ 9.  SCOPE RBAC TO NAMESPACES when possible; use resource-specific rules
☐ 10. USE OCI REPOSITORIES for chart storage (better auth, signing, and immutability)
☐ 11. AUTHENTICATE TO PRIVATE REPOSITORIES with TLS client certs or OCI login
☐ 12. NEVER USE --insecure-skip-tls-verify in production
☐ 13. PIN CHART VERSIONS to explicit SemVer in requirements and CI/CD
☐ 14. MIRROR THIRD-PARTY CHARTS to your internal registry
☐ 15. PREFER VERIFIED PUBLISHERS on Artifact Hub
☐ 16. SCAN CHARTS AND RENDERED MANIFESTS with Trivy or Grype in CI/CD
☐ 17. FAIL PIPELINES on CRITICAL and HIGH severity findings
☐ 18. ENABLE KUBERNETES AUDIT LOGGING at Metadata level or higher
☐ 19. STORE RELEASE SECRETS in etcd encryption at rest (Kubernetes feature)
☐ 20. USE Pod Security Standards (restricted) on Helm-managed namespaces
☐ 21. SEPARATE DUTIES — chart authors do not deploy to production
☐ 22. ROTATE SIGNING KEYS annually; revoke compromised keys immediately
☐ 23. USE READ-ONLY kubeconfig for lint/template/dry-run operations in CI/CD
☐ 24. NEVER STORE PRIVATE GPG KEYS on CI/CD runners; use KMS or Vault-backed signing
☐ 25. ENFORCE POLICIES on rendered manifests (OPA/Kyverno admission webhooks)
☐ 26. REVIEW Artifact Hub security reports for third-party charts before adoption
☐ 27. FOLLOW VULNERABILITY DISCLOSURE process; do not open public issues for security bugs
☐ 28. REGULARLY UPDATE Helm CLI to the latest patch version
```
