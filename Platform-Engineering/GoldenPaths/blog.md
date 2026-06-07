# Golden Paths

Golden Paths are opinionated, supported, and well-documented workflows for common development tasks. Instead of giving developers unlimited choice ("use any language, any framework, any deployment strategy"), Golden Paths provide a curated, supported path that handles 90% of use cases. Need something outside the path? Go ahead—but you're on your own for support.

The concept originated at Spotify with Backstage: "Golden Paths are to platform engineering what design systems are to UI development." They reduce cognitive load (developers don't need to know every option), improve consistency (all services look similar), and enable platform teams to support developers at scale (support 5 paths, not 500 unique architectures).

## Golden Path Design

```yaml
# Golden Path: "Deploy a Backend Service"
golden_path:
  name: backend-service
  description: Standard backend service deployment
  when_to_use: |
    Building a new HTTP/gRPC service. Works for Python, Go, or Java.
  
  steps:
    - scaffold:
        command: toolkit create service --name <name> --language python|go|java
        generates: repo structure, Dockerfile, Helm chart, CI/CD config
    
    - develop:
        tools: [VSCode, IntelliJ, devcontainer config provided]
        local_run: docker-compose up (includes emulators for dependencies)
    
    - test:
        unit: pytest, go test, JUnit (run automatically in CI)
        integration: docker-compose.test.yml
    
    - deploy:
        staging: auto-deploy on merge to main
        production: auto-deploy after staging smoke tests pass
    
    - observe:
        dashboards: auto-provisioned Grafana dashboard
        alerts: auto-provisioned (high error rate, high latency)
        logs: auto-collected to Loki
    
    - database:
        option: managed-postgres (RDS, Cloud SQL)
        migration: flyway (auto-run in CI)
        backup: daily snapshots (30-day retention)
  
  supported_languages:
    python:
      framework: FastAPI
      package_manager: poetry
      lint: ruff
    go:
      framework: chi or gin
      lint: golangci-lint
    java:
      framework: Spring Boot
      build: Gradle
  
  not_supported:
    - Rust (no golden path yet; community support only)
    - PHP (legacy; migration path to Go/Python)
```

## Golden Path vs Freedom

```
Spectrum of constraint:

NO SUPPORT (Wild West):
  "Use whatever you want. Figure it out yourself."
  → Maximum freedom, zero consistency, high cognitive load

GOLDEN PATHS:
  "Here are 5 supported ways to do things. Follow a path and we support you. 
   Go off-path and you're on your own."
  → High consistency for 90% of use cases, freedom for the 10%

TOTAL PLATFORM:
  "You can ONLY use these specific tools and patterns."
  → Maximum consistency, zero freedom, stifles innovation
```

## Implementation with Backstage

```yaml
# Backstage software template (scaffolds a golden path)
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: python-backend-service
  title: Python Backend Service
  description: Golden path for Python backend services
spec:
  owner: platform-engineering
  type: service
  
  parameters:
    - title: Service Configuration
      required: [name, owner, port]
      properties:
        name:
          title: Service Name
          type: string
          pattern: '^[a-z][a-z0-9-]*$'
        owner:
          title: Owner
          type: string
          ui:field: OwnerPicker
        port:
          title: Port
          type: integer
          default: 8080
        database:
          title: Needs Database?
          type: boolean
          default: false
        public_endpoint:
          title: Public Endpoint?
          type: boolean
          default: false
  
  steps:
    - action: fetch:template
      input:
        url: ./template
        values:
          name: ${{ parameters.name }}
          port: ${{ parameters.port }}
    
    - action: publish:github
      input:
        repoUrl: github.com?owner=myorg&repo=${{ parameters.name }}
    
    - action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish'].output.repoContentsUrl }}
```

## Measuring Golden Path Adoption

```python
# Key metrics for golden path success
metrics = {
    'adoption_rate': 'Percentage of new services following a golden path',
    'time_to_first_deploy': 'Days from repo creation to first production deploy',
    'off_path_count': 'Number of services NOT on any golden path',
    'path_satisfaction': 'NPS/CSAT for each golden path',
    'pr_frequency': 'Platform team PR review load (should decrease with golden paths)',
}

# Goal: 80%+ of services on golden paths
#       Time to first deploy < 1 day
#       PR review load decreasing over time
```

## Summary

Golden Paths are the bridge between "use whatever you want" (chaos at scale) and "you must use exactly this" (stifling). They encode architectural best practices as the default, supported option while preserving escape hatches for special cases. A well-designed golden path makes the right thing the easy thing.
