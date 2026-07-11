# Helm — Package Manager for Kubernetes

Helm bundles related Kubernetes resources into **charts** (templates + values), enabling versioned, reusable, and configurable deployments. Helm is to Kubernetes what `apt` is to Ubuntu, `brew` is to macOS, or `pip` is to Python.

**Key Concepts:** Chart · Repository · Release · Revision · Values · Templates · Hooks · OCI Registry

---

## Quick Reference

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo postgresql
helm install my-release bitnami/postgresql -n production -f values-prod.yaml
helm upgrade my-release bitnami/postgresql -n production --set persistence.size=200Gi
helm rollback my-release 1 -n production
helm uninstall my-release -n production
helm list -n production
helm history my-release -n production
helm template my-release ./chart -f values-prod.yaml
helm lint ./chart
helm package ./chart
helm dependency update ./chart
```

---

## The Handbook

For complete mastery, see the full handbook:

| # | Chapter | Description |
|---|---------|-------------|
| 0 | [Frontmatter](handbook/00-frontmatter.md) | Title page, table of contents |
| 1 | [What is Helm?](handbook/01-what-is-helm.md) | Why Helm exists, architecture, key concepts |
| 2 | [Architecture](handbook/02-architecture.md) | Internal flow, release storage, rendering pipeline |
| 3 | [Installation](handbook/03-installation.md) | Every install method, upgrade/downgrade, plugins |
| 4 | [Repository Management](handbook/04-repository-management.md) | Repos, OCI, private repos, caching, indexing |
| 5 | [Chart Management](handbook/05-chart-management.md) | Chart structure, every file explained, packaging |
| 6 | [Release Management](handbook/06-release-management.md) | Install/upgrade/rollback lifecycle, revisions |
| 7 | [Values](handbook/07-values.md) | Override precedence, merge behavior, inline values |
| 8 | [Templating](handbook/08-templating.md) | Go templates, Sprig, named templates, whitespace |
| 9 | [Helm Commands](handbook/09-helm-commands.md) | Every command, every flag, complete reference |
| 10 | [OCI Registries](handbook/10-oci-registries.md) | Login, push/pull, ECR/ACR/Harbor/GHCR |
| 11 | [Helm Hooks](handbook/11-helm-hooks.md) | All hook types, weights, execution order |
| 12 | [Helm Tests](handbook/12-helm-tests.md) | Test pods, cleanup, production validation |
| 13 | [Debugging](handbook/13-debugging.md) | Lint, template, dry-run, kubectl integration |
| 14 | [Rollbacks](handbook/14-rollbacks.md) | Revision history, storage, recovery strategies |
| 15 | [Chart Development](handbook/15-chart-development.md) | Subcharts, dependencies, library charts, best practices |
| 16 | [Security](handbook/16-security.md) | Signing, provenance, RBAC, supply chain |
| 17 | [CI/CD](handbook/17-cicd.md) | GitHub Actions, ArgoCD, Flux, atomic upgrades |
| 18 | [Helm with K8s Resources](handbook/18-kubernetes-resources.md) | Managing every resource type via Helm |
| 19 | [Production Best Practices](handbook/19-production-best-practices.md) | Versioning, secrets, GitOps, monitoring, DR |
| 20 | [CKA Exam Focus](handbook/20-cka-exam-focus.md) | Commands to memorize, time-saving, pitfalls |
| 21 | [Cheat Sheets](handbook/21-helm-cheat-sheets.md) | Quick reference tables for every scenario |
| 22 | [Practice Scenarios](handbook/22-practice-scenarios.md) | 100+ exercises with solutions |
| 23 | [Interview Questions](handbook/23-interview-questions.md) | 100+ questions: easy, medium, hard, scenario |
| 24 | [Command Reference](handbook/24-complete-command-reference.md) | Alphabetical reference, every flag, every example |
