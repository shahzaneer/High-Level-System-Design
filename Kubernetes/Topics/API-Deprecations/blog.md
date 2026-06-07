# API Deprecations & Upgrades

Kubernetes APIs evolve, and deprecated API versions are eventually removed. Running a cluster with workloads using deprecated APIs that get removed in the next version upgrade causes: deployments rejected, monitoring broken, and worst case—invisible failures until the upgrade when everything breaks simultaneously.

The Kubernetes deprecation policy: API versions are supported for 3 releases (9-12 months) after deprecation announcement. For example, `extensions/v1beta1` Ingress was deprecated in v1.14, removed in v1.22. Teams that didn't migrate were broken.

## Detecting Deprecated APIs

```bash
# kubectl deprecations (built-in since v1.26)
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# Pluto (Fairwinds): scan cluster for deprecated API usage
pluto detect-all-in-namespace production
pluto detect-helm -o wide

# Kube-no-trouble (kubent): detect deprecated APIs
kubent

# kubeconform (Yann Hamon): validate manifests
kubeconform -kubernetes-version 1.30.0 manifests/
```

## Common Deprecations

| Deprecated API | Removed In | Replacement |
|---------------|-----------|-------------|
| `extensions/v1beta1` Ingress | v1.22 | `networking.k8s.io/v1` Ingress |
| `extensions/v1beta1` PodSecurityPolicy | v1.25 | Pod Security Standards (built-in) |
| `autoscaling/v2beta2` HPA | v1.26 | `autoscaling/v2` |
| `policy/v1beta1` PodDisruptionBudget | v1.25 | `policy/v1` |
| `batch/v1beta1` CronJob | v1.25 | `batch/v1` |

## Pre-Upgrade Checklist

```bash
# 1. Check current version
kubectl version --short

# 2. Check all target version deprecations
# Read Kubernetes changelog: https://kubernetes.io/releases/
# Check: deprecated API removals section

# 3. Scan cluster for deprecated API usage
pluto detect-all-in-cluster

# 4. Scan all IaC repos
grep -r "extensions/v1beta1\|apps/v1beta\|batch/v1beta1" .

# 5. Fix all findings BEFORE upgrading
# Update manifests to current API versions
# Test in staging cluster first

# 6. Check add-on compatibility
# cert-manager, ingress-nginx, monitoring stack must support target version

# 7. Backup etcd (or take EKS/GKE/AKS cluster snapshot)

# 8. Upgrade control plane first, then node groups
# Follow cloud provider's specific upgrade process

# 9. Post-upgrade: verify all workloads healthy
kubectl get pods -A | grep -v Running
```

## Automated Scanning in CI/CD

```yaml
# GitHub Actions: check for deprecated APIs on PR
- name: Check deprecated APIs
  run: |
    kubeconform -kubernetes-version 1.30.0 -summary \
      -ignore-filename-pattern 'kustomization.yaml' \
      kubernetes/
```

## Managed K8s Upgrades

```bash
# EKS: upgrade control plane + node groups
aws eks update-cluster-version --name prod --kubernetes-version 1.30
eksctl upgrade nodegroup --cluster prod --name app-nodes

# GKE: upgrade with release channels
gcloud container clusters upgrade prod --master --cluster-version 1.30

# AKS: upgrade
az aks upgrade --name prod --kubernetes-version 1.30
```

API deprecation management should be part of the CI/CD pipeline, not a pre-upgrade scramble. Automated scanning catches deprecated APIs before they break production.
