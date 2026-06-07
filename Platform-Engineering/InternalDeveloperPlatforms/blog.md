# Internal Developer Platforms (IDPs)

Internal Developer Platforms are the "golden path" abstraction layer that enables developers to self-serve infrastructure without needing deep expertise in Kubernetes, Terraform, or cloud services. An IDP provides a curated set of capabilities—provision a database, deploy a service, expose an endpoint—through a developer-friendly interface (CLI, web portal, or declarative YAML).

The goal is NOT to build a "company-wide PaaS" but to provide the 80% of common functionality that every team needs (deploy, observe, secure, connect) while allowing teams to customize the 20% that's unique to their domain. Spotify's Backstage is the canonical open-source IDP, but the concept predates it (Netflix's tools, Google's Borg/Build infrastructure).

## Architecture

```
┌────────────────────────────────────────────────────────┐
│                DEVELOPER INTERFACES                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Backstage│  │   CLI    │  │  GitOps  │             │
│  │ Portal   │  │ (toolkit)│  │ (PR → TF)│             │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘             │
│       └──────────────┼─────────────┘                    │
│                      │                                  │
│   ┌──────────────────▼──────────────────────┐          │
│   │         PLATFORM ORCHESTRATOR            │          │
│   │  (Crossplane, Kratix, custom K8s ops)    │          │
│   │  • Translates dev intent → infra reality │          │
│   │  • Enforces security policies            │          │
│   │  • Manages lifecycle (create/update/del) │          │
│   └──────────────────┬──────────────────────┘          │
│                      │                                  │
│   ┌──────────────────▼──────────────────────┐          │
│   │       INFRASTRUCTURE LAYER               │          │
│   │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │          │
│   │  │  K8s │ │  DB  │ │Cache │ │DNS/  │   │          │
│   │  │      │ │  Svcs│ │ Svcs │ │ TLS  │   │          │
│   │  └──────┘ └──────┘ └──────┘ └──────┘   │          │
│   └─────────────────────────────────────────┘          │
└────────────────────────────────────────────────────────┘
```

## Key Capabilities

```yaml
# Developer self-service capabilities via the IDP:
capabilities:
  - provision_database:
      type: PostgreSQL
      size: small|medium|large
      ha: true|false
      
  - deploy_service:
      source: git_repo_url
      build: Dockerfile|buildpack
      replicas: 2
      autoscaling: true
      
  - expose_endpoint:
      domain: api.example.com
      path: /orders
      auth: OAuth2
      
  - create_topic:
      name: order-events
      partitions: 3
      retention: 7d
      
  - request_access:
      resource: production/database
      reason: "Debugging latency issue"
      duration: 4h
      approvals: [tech-lead]
```

## Developer Experience

```
BEFORE IDP:
  "I need to deploy a new service."
  1. Read internal wiki (outdated)
  2. Copy-paste Terraform from another team
  3. Discover hardcoded values for their environment
  4. Open PR → wait 2 days for platform team review
  5. PR comments: "use our standard module, not this custom one"
  6. Fix and re-submit
  7. 1 week later: service is deployed

AFTER IDP:
  "I need to deploy a new service."
  1. Run: toolkit deploy create --name order-service --type python
  2. Answer 5 questions (team, port, resources, domain)
  3. CLI generates all infrastructure code
  4. GitOps automatically provisions everything
  5. 15 minutes later: service is running
```

## Build vs Buy vs "Adopt"

| Approach | Example | Best For |
|----------|---------|----------|
| Build from scratch | Custom K8s operators, Terraform modules | 500+ engineer orgs with dedicated platform team |
| Adopt & customize | Backstage + custom plugins | 50-500 engineer orgs, want OSS foundation |
| Buy (commercial) | Humanitec, Port, Cortex, Coherence | Want results fast, smaller platform team |
| Don't build one | Vanilla Terraform + docs | < 50 engineers—IDP is overkill |

## Why This Matters for Architects

The IDP is the modern architect's deliverable. Instead of designing one-off architectures for each team, architects design the PLATFORM that enables teams to build their own architectures within guardrails. This shifts the architect's role from "approver of every design" to "designer of the system that enables design."

A good IDP encodes architectural best practices as defaults: every service gets observability, every database has backups, every endpoint has TLS. Teams don't need to know how these work—they get them by default. The architect's value scales across the entire engineering organization.
