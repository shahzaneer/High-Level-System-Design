# Developer Experience (DevEx)

Developer Experience (DevEx) is the practice of optimizing the tools, workflows, and environment that developers interact with daily. It treats developers as the "users" and the platform as the "product." Just as UX designers optimize customer-facing products, DevEx optimizes the engineering experience: how long does it take to set up a new service? How fast is the CI/CD pipeline? How easy is it to find documentation?

DevEx is measured by the SPACE framework (Satisfaction, Performance, Activity, Communication, Efficiency) or the simpler DORA metrics. The ROI is direct: a 10-minute improvement in build time across 100 developers who build 10 times per day saves 167 engineer-hours per week. DevEx is not about beanbag chairs and free snacks—it's about reducing friction in the engineering workflow.

## Key Metrics

```python
class DevExMetrics:
    """Measure what matters for developer productivity"""
    
    @staticmethod
    def time_to_first_commit():
        """From idea to first code commit. Includes: setup, scaffolding, permissions"""
        return metric('repo_created → first_commit')
    
    @staticmethod
    def time_to_first_deploy():
        """From repo creation to first production deployment"""
        return metric('repo_created → first_production_deploy')
    
    @staticmethod
    def build_time_p50_p95():
        """CI build time distribution"""
        return metric('ci_build_duration', percentiles=[50, 95])
    
    @staticmethod
    def pr_review_time():
        """Time from PR open to first review, and to merge"""
        return metric('pr_opened → first_review', 'pr_opened → merged')
    
    @staticmethod
    def local_dev_setup_time():
        """Time to get a working dev environment on a new laptop"""
        return survey('How long did it take to set up your dev environment?')
    
    @staticmethod
    def developer_nps():
        """Net Promoter Score: 'Would you recommend our dev tooling?'"""
        return survey('On a scale of 1-10, how likely are you to recommend...')
```

## Common Friction Points

| Friction | Symptom | Fix |
|----------|---------|-----|
| Slow CI | Build takes 20+ minutes | Parallelize tests, cache dependencies, faster runners |
| Flaky tests | Random failures, developers re-run | Quarantine flaky tests, fix or delete |
| PR review delay | PRs sit for 2+ days | Auto-assign reviewers, team SLAs |
| Poor docs | "How do I..." in Slack constantly | TechDocs in repo, Golden Paths |
| Secrets access | "Can I get prod DB access?" ticket | Self-service JIT access via IDP |
| Environment drift | "Works on my machine" | Standardized dev containers, pre-built images |

## The Platform as a Product Mindset

```
Traditional Ops                  Platform Engineering
───────────────                  ───────────────────
"You need a database?            "Here's a self-service
 File a ticket."                 DB provisioning API."

"I'll review your PR             "Here's a CI template 
 and deploy it."                 with pre-approved patterns."

"Here's the server."             "Here's a golden path that
                                 provisions, deploys, monitors."

"Read the wiki"                  "Run: toolkit explain database"
```

## Dev Containers

```json
// .devcontainer.json (standardized dev environment)
{
  "image": "mcr.microsoft.com/devcontainers/python:3.12",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "postCreateCommand": "pip install -r requirements-dev.txt && pre-commit install",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "charliermarsh.ruff"
      ]
    }
  }
}
```

## Summary

DevEx treats the engineering organization as a product to be optimized. Friction in the development workflow—slow builds, delayed reviews, confusing docs—compounds across the organization. Removing friction scales linearly with team size. The architect's role in DevEx is to design the golden paths, standardize the toolchain, and measure the metrics that drive improvements.
