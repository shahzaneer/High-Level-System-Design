# Pipeline Security & DevSecOps

## Introduction
Pipeline Security (often called DevSecOps) integrates security practices into the CI/CD pipeline rather than treating security as a separate phase after development. The traditional model—develop, then hand off to security for review, then deploy—creates friction, delays, and often results in security findings being ignored because "we need to ship." DevSecOps shifts security left: every commit is scanned for secrets, every build is tested for vulnerabilities, every image is signed, and every deployment is verified against policy.

The 2020 SolarWinds attack demonstrated the catastrophic consequence of pipeline compromise—attackers injected malicious code into the build system, and it was signed and distributed to 18,000 customers as a legitimate update. Pipeline security is no longer optional. It is the highest-value security investment because compromising the pipeline gives attackers access to everything the pipeline can deploy.

## Definition

**DevSecOps** is the practice of integrating security testing and controls throughout the CI/CD pipeline—from code commit through deployment to production monitoring. It ensures security is a shared responsibility across development, security, and operations teams, automated wherever possible.

**Pipeline Security** specifically protects the CI/CD pipeline itself: securing source code repositories, build systems, artifact registries, deployment mechanisms, and the credentials and secrets used throughout.

## Concept Explanation

### Security Gates in the Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                    SECURITY GATES                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Code] ──→ [SAST] ──→ [SCA] ──→ [Secret Scan] ──→ [Build]  │
│                      │                                        │
│              ┌───────▼────────┐                              │
│              │ Policy Check   │                              │
│              │ (OPA/Rego)     │                              │
│              └───────┬────────┘                              │
│                      │                                        │
│  [Build] ──→ [Image Scan] ──→ [Sign Image] ──→ [Registry]   │
│                                                        │     │
│  [Deploy] ──→ [Admission Control] ──→ [Runtime Scan] ──┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1. Secret Scanning (Pre-Commit + CI)

```yaml
# Pre-commit: stop secrets before they reach Git
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

```bash
# CI pipeline: GitHub Actions secret scanning
# Built-in secret scanning (free for public repos)
# Also use: truffleHog for deep scanning

trufflehog git file://. --only-verified
```

### 2. Static Application Security Testing (SAST)

```yaml
# GitHub Actions: CodeQL (SAST)
- name: Initialize CodeQL
  uses: github/codeql-action/init@v3
  with:
    languages: javascript, python

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v3

# Semgrep: fast, configurable SAST
- name: Semgrep
  run: |
    semgrep --config=auto --error --json -o semgrep.json .

# Bandit: Python-specific security linting
- name: Bandit
  run: bandit -r src/ -f json -o bandit.json
```

### 3. Software Composition Analysis (SCA)

```bash
# Detect vulnerable dependencies
# npm audit, pip-audit, OWASP Dependency-Check

# GitHub Dependabot: automatic PRs for vulnerable deps
# Configuration: .github/dependabot.yml

pip-audit --requirement requirements.txt --format json -o pip-audit.json

# Trivy: scan for vulnerabilities in dependencies + OS packages
trivy fs --severity CRITICAL,HIGH .
```

### 4. Container Image Scanning

```yaml
- name: Build container
  run: docker build -t order-service:${{ github.sha }} .

- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: order-service:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'  # Fail pipeline on findings

# Docker Scout (built-in vulnerability scanning)
- name: Docker Scout
  run: docker scout cves order-service:${{ github.sha }} --exit-code
```

### 5. Image Signing (Sigstore / Cosign)

```bash
# Sign container image (keyless signing with OIDC)
cosign sign \
  --oidc-issuer https://token.actions.githubusercontent.com \
  ${{ env.REGISTRY }}/order-service:${{ github.sha }}

# Verify signature before deployment
cosign verify \
  --certificate-identity https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ${{ env.REGISTRY }}/order-service:${{ github.sha }}
```

### 6. Admission Control (Policy Enforcement at Deploy)

```yaml
# OPA Gatekeeper: enforce policies at Kubernetes admission
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["team", "environment"]
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockLatestTag
metadata:
  name: block-latest-tag
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
---
# Kyverno: simpler, policy-driven admission control
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-run-as-non-root
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Running as root is not allowed"
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
```

### 7. SBOM (Software Bill of Materials)

```bash
# Generate SBOM for transparency and vulnerability tracking
syft order-service:${{ github.sha }} -o spdx-json > sbom.spdx.json

# Upload to dependency track for continuous monitoring
curl -X POST https://dependencytrack.example.com/api/v1/bom \
  -H "X-Api-Key: $DEPENDENCY_TRACK_KEY" \
  -F "project=order-service" \
  -F "bom=@sbom.spdx.json"
```

### Pipeline Access Control

```yaml
# GitHub Actions: OIDC federation (no long-lived cloud credentials)
# Trust GitHub's OIDC provider → assume IAM role

# AWS IAM trust policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:org/order-service:*"
      }
    }
  }]
}
```

## Layman's Explanation

### The Airport Security Model
Traditional security: "Build the plane (develop), then the TSA inspects it at the gate (security review), then it takes off (deploy)." Problems: The TSA finds a critical issue, and the flight is delayed by hours. Sometimes, under pressure, the TSA waves it through.

DevSecOps: Security is built into every step:
- **Secret Scanning**: A metal detector at the factory entrance—catches weapons before they enter the building
- **SAST**: X-ray scanning during assembly—finds structural flaws inside components
- **SCA**: Vet the suppliers—verify every part that goes into the plane isn't recalled or counterfeit
- **Image Scanning**: Final inspection of the complete aircraft before it leaves the hangar
- **Image Signing**: The seal on the cockpit door—verifies nobody tampered with it after inspection
- **Admission Control**: Air traffic control—only cleared, signed planes are allowed on the runway

### The Supply Chain Analogy
The SolarWinds attack is like a food manufacturer having their production line compromised. The attacker didn't attack the restaurant—they poisoned the ingredients at the factory. Every restaurant that used those ingredients served poisoned food, even though the restaurant itself was secure. Pipeline security is about securing the factory (the build pipeline) so that everything it produces can be trusted.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **OIDC Federation vs Long-Lived Credentials**: CI/CD systems need cloud access to deploy. Traditional approach: create an IAM user with access keys, store them as GitHub Secrets. Problem: long-lived credentials that can leak. Modern approach: OIDC federation—the pipeline assumes a role via trust, no persistent credentials exist to steal.
- **Policy as Code (OPA/Kyverno)**: Security policies must be version-controlled, tested, and automatically enforced. Manual security review is neither scalable nor reliable. OPA policies define what's allowed; the admission controller enforces it automatically.
- **SBOM Strategy**: Executive Order 14028 mandates SBOMs for software sold to the U.S. government. Even without regulatory pressure, SBOMs enable rapid response to new CVEs—when Log4Shell hit, organizations with SBOMs knew within hours which applications were vulnerable; others spent weeks manually checking.
- **Immutable Artifacts**: Container images must be immutable (no `:latest` tags in production). Every production deployment references a specific SHA256 digest. This ensures that "the image that passed security scanning" is "the image that's running in production."

### Business Impact
- **Supply Chain Attack Prevention**: The average supply chain attack costs $4.33 million. Pipeline security—signed commits, signed images, SBOM—prevents or detects these attacks before they reach production.
- **Compliance**: SOC 2 CC6.1 requires logical access controls. SSDF (Secure Software Development Framework) from NIST (SP 800-218) codifies many DevSecOps practices. FedRAMP requires code scanning and vulnerability management.
- **Developer Trust**: When developers trust the pipeline—that their code will be scanned and only secure builds reach production—they deploy more frequently and with more confidence.

## AWS, GCP, Azure Examples

```yaml
# AWS Inspector: continuous vulnerability scanning
resource "aws_inspector2_enabler" "main" {
  account_ids    = [data.aws_caller_identity.current.account_id]
  resource_types = ["EC2", "ECR", "LAMBDA"]
}
```

```bash
# GCP Binary Authorization: only allow signed images
gcloud container clusters update production-cluster \
  --enable-binary-authorization

# Azure Defender for Containers
az aks enable-addons --addons azure-defender
```

## Summary

| Pipeline Security Layer | Tool |
|------------------------|------|
| Secret Scanning | gitleaks, truffleHog, GitHub Secret Scanning |
| SAST | CodeQL, Semgrep, SonarQube, Checkmarx |
| SCA | Dependabot, Snyk, Trivy, OWASP Dependency-Check |
| Image Scanning | Trivy, Grype, Docker Scout, Clair |
| Image Signing | Cosign (Sigstore), Notary |
| Admission Control | OPA/Gatekeeper, Kyverno |
| SBOM | Syft, CycloneDX, SPDX |

Pipeline security is the highest-leverage security investment an organization can make. Compromise the pipeline, and you compromise everything it deploys. Secure the pipeline—shift left with SAST/SCA/secret scanning, sign artifacts, enforce admission control—and you create a secure supply chain that produces trusted, verifiable software. The architect's role is to design the pipeline with security gates at every stage, automate enforcement via policy-as-code, and ensure the pipeline itself is protected with short-lived credentials, signed commits, and immutable artifacts.
