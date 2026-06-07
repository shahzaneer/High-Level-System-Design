# Pod Security Standards

Pod Security Standards (PSS) define three policy levels for pod security: Privileged (unrestricted), Baseline (minimally restrictive, prevents known privilege escalations), and Restricted (heavily restricted, follows hardening best practices). PSS replaced the deprecated PodSecurityPolicy (PSP) in Kubernetes 1.25.

PSS are enforced at the namespace level via labels. Pods that violate the policy are rejected by the admission controller.

## Three Policy Levels

| Level | Description | Examples |
|-------|-------------|---------|
| Privileged | Unrestricted; anything allowed | System pods, CI/CD builders, security tools |
| Baseline | Prevents known privilege escalations | Most application workloads; default minimum |
| Restricted | Hardened best practices | Multi-tenant clusters, high-security environments |

## Namespace Enforcement

```bash
# Imperative: label namespace for pod security
kubectl label ns production pod-security.kubernetes.io/enforce=restricted
kubectl label ns production pod-security.kubernetes.io/audit=restricted
kubectl label ns production pod-security.kubernetes.io/warn=restricted
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## Restricted Profile Requirements

```yaml
# This pod passes Restricted:
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      readOnlyRootFilesystem: true
```

Key Restricted requirements:
- Must NOT run as root (`runAsNonRoot: true`)
- Must drop ALL capabilities
- Cannot use `privileged: true`
- Cannot use hostNetwork, hostPID, hostIPC
- Must have seccomp profile (RuntimeDefault or Localhost)
- Should use read-only root filesystem
- Cannot use hostPath volumes

## Warning and Audit Modes

```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted  # REJECTS violating pods
  pod-security.kubernetes.io/warn: restricted     # WARNS but allows
  pod-security.kubernetes.io/audit: restricted    # AUDIT LOGS violations
```

Use `warn` during migration to avoid breaking workloads. Promote to `enforce` after verifying compatibility.

## Exemptions

```bash
# Exempt specific users, namespaces, or runtime classes
# Configured in the admission controller configuration
```

## Imperative vs Declarative

Namespace labels are imperative. Pod security contexts are declarative in pod specs. Production should enforce at minimum Baseline; Restricted is recommended for multi-tenant or high-security environments.
