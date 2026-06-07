# Backstage (Service Catalog)

Backstage is Spotify's open-source platform for building developer portals. Its core is the Software Catalog—a centralized registry of all services, APIs, websites, libraries, and data pipelines in the organization, with ownership metadata, documentation, and operational status. Built as a plugin-based React/Node.js application, Backstage has become the de facto standard for Internal Developer Portals since its CNCF donation in 2020.

Backstage's killer feature: it connects disparate tools into a unified developer experience. Instead of visiting 10 different UIs (GitHub for code, PagerDuty for alerts, Grafana for metrics, Confluence for docs), Backstage surfaces relevant information from all tools in one place, contextualized to the service you're looking at.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                  BACKSTAGE UI                         │
│  ┌────────────────────────────────────────────────┐  │
│  │              Plugin Architecture                │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐       │  │
│  │  │Catalog   │ │TechDocs  │ │Kubernetes│       │  │
│  │  │Plugin    │ │Plugin    │ │Plugin    │       │  │
│  │  └──────────┘ └──────────┘ └──────────┘       │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐       │  │
│  │  │Grafana   │ │PagerDuty │ │GitHub    │       │  │
│  │  │Plugin    │ │Plugin    │ │Plugin    │       │  │
│  │  └──────────┘ └──────────┘ └──────────┘       │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────┬───────────────────────────────┘
                       │ APIs
┌──────────────────────▼───────────────────────────────┐
│                  BACKSTAGE BACKEND                     │
│  ┌────────────────────────────────────────────────┐  │
│  │  Catalog Database  │  Search  │  Auth (OIDC)   │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────┬───────────────────────────────┘
                       │ Integrations
┌──────────────────────▼───────────────────────────────┐
│  GitHub │ PagerDuty │ Kubernetes │ Grafana │ Jenkins │
└──────────────────────────────────────────────────────┘
```

## Catalog Entity Definition

```yaml
# catalog-info.yaml (lives in each service's repo)
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-service
  description: Order processing microservice
  annotations:
    github.com/project-slug: myorg/order-service
    backstage.io/techdocs-ref: dir:.
    pagerduty.com/service-id: P123ABC
    grafana/dashboard-selector: "service=order-service"
  tags:
    - python
    - microservice
    - pci
  links:
    - url: https://wiki.internal.com/order-service
      title: Team Wiki
spec:
  type: service
  lifecycle: production
  owner: order-engineering
  system: ecommerce-platform
  dependsOn:
    - component:payment-service
    - resource:orders-database
  providesApis:
    - order-api
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: order-api
spec:
  type: openapi
  lifecycle: production
  owner: order-engineering
  definition:
    $text: https://github.com/myorg/order-service/blob/main/openapi.yaml
---
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: orders-database
spec:
  type: database
  owner: order-engineering
  system: ecommerce-platform
```

## Key Plugins for Architects

| Plugin | Purpose |
|--------|---------|
| Catalog | Service registry, ownership, dependencies |
| TechDocs | Documentation from Markdown in repo (MkDocs) |
| Kubernetes | View deployments, pods, logs within Backstage |
| Grafana | Embed dashboards per service |
| PagerDuty/OpsGenie | On-call status, alert history |
| Cloud Cost | Per-service cloud spend |
| Tech Radar | Technology adoption guidance |
| Scaffolder | Software templates—golden path creation |

## Catalog Discovery (Auto-Registration)

```yaml
# Backstage discovers entities automatically from:
# 1. Static catalog-info.yaml files in repos
# 2. GitHub/GitLab org discovery (find all repos with catalog-info.yaml)
# 3. LDAP/AD (users, groups)
# 4. Kubernetes API (deployments, pods)
# 5. Cloud providers (AWS resources, GCP projects)
```

```yaml
# app-config.yaml: GitHub discovery
catalog:
  providers:
    github:
      organization:
        target: https://github.com/myorg
        schedule:
          frequency: { minutes: 30 }
      filters:
        repository: 'catalog-info.yaml'
```

## Summary

Backstage provides the "single pane of glass" for developer experience. The catalog is the source of truth for "what services exist, who owns them, and how they're doing." For architects, Backstage surfaces the actual architecture—not what's drawn in diagrams, but what's running in production. It's simultaneously a documentation tool, an operational dashboard, and a service creation workflow.
