# Self-Service Infrastructure

Self-service infrastructure is the operational model where developers provision their own resources (databases, caches, message queues, DNS) through automated interfaces without manual approval from operations teams. It's enabled by infrastructure-as-code, policy-as-code, and platform abstractions. The goal is NOT to eliminate operations—it's to automate the routine 90% so ops can focus on the complex 10%.

The key enabler: guardrails, not gates. Operations defines what's allowed (policies) and provides self-service tools that enforce those policies automatically. Developers operate within the guardrails without needing to understand them in detail.

## Policy-as-Code (OPA/Rego)

```rego
# OPA policy: enforce production deployment standards
package platform.production

deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    not container.resources.requests.cpu
    msg = sprintf("%s: CPU request is required", [input.metadata.name])
}

deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    container.resources.requests.memory == container.resources.limits.memory
    msg = sprintf("%s: Memory requests must differ from limits", [input.metadata.name])
}

deny[msg] {
    input.kind == "Deployment"
    not input.spec.template.spec.containers[0].securityContext.runAsNonRoot
    msg = sprintf("%s: Container must not run as root", [input.metadata.name])
}
```

## Self-Service Database Provisioning

```yaml
# Developer declares: "I need a PostgreSQL database"
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: orders-db
  namespace: production
spec:
  version: "16"
  size: medium          # small=2vCPU/8GB, medium=4vCPU/16GB, large=8vCPU/32GB
  storage: 100Gi
  ha: true              # Multi-AZ with standby
  backup:
    retention: 30       # days
    schedule: "0 2 * * *"
---
# Crossplane composition translates to cloud resources automatically:
# - AWS RDS instance provisioned
# - Security group created
# - Subnet group configured
# - Secrets stored in AWS Secrets Manager
# - Connection string injected into app
```

## Tiered Access Model

```yaml
# Tiered self-service: what you can do depends on environment
tiers:
  sandbox:
    who: any developer
    limits:
      max_instances: 3
      max_cost_per_month: $50
      auto_cleanup: 7 days idle → delete
    approval: none
    
  development:
    who: any developer
    limits:
      max_instances: 10
      max_cost_per_month: $500
    approval: none
    
  staging:
    who: team lead
    limits:
      max_instances: 20
    approval: team lead
    
  production:
    who: senior engineer + on-call
    limits:
      requires: change request
      windows: business hours only
    approval: tech lead + SRE
    audit: all changes logged to SIEM
```

## Repository Templates

```bash
# Self-service service creation
toolkit service create order-service \
  --template python-backend \    # Golden path template
  --team order-engineering \
  --port 8080 \
  --public-endpoint api.example.com/orders

# What happens automatically:
# 1. GitHub repo created from template
# 2. CI/CD pipeline configured (GitHub Actions)
# 3. Terraform workspace provisioned
# 4. Kubernetes namespace + RBAC created
# 5. Grafana dashboard provisioned
# 6. PagerDuty service created
# 7. Backstage catalog registered
# 8. README with team info, links, badges
```

## Summary

Self-service infrastructure shifts the operations bottleneck from "manual approval" to "automated policy enforcement." Developers get what they need in minutes, not days. Platform teams encode their expertise into policies and templates that scale across the entire organization. The key: guardrails (policy-as-code), not gates (manual approvals).
