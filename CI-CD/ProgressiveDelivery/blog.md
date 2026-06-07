# Progressive Delivery

## Introduction
Progressive Delivery is the evolution of continuous delivery that gives operators fine-grained control over how new software versions are rolled out to users. Instead of a binary switch (old version → new version, all users at once), progressive delivery exposes the new version gradually: 1% of users → if metrics are good → 5% → 25% → 50% → 100%. At each step, if error rates spike or latency degrades, the rollout automatically halts or rolls back.

Popularized by platforms like Netflix (Spinnaker), Google, and Amazon, progressive delivery combines deployment strategies (canary, blue-green, rolling update) with automated analysis and automated rollback. It transforms deployments from high-risk events requiring manual monitoring into automated, self-verifying processes that are safe enough to run unattended—even on a Friday afternoon.

## Definition

**Progressive Delivery** is a set of deployment practices that expose new software versions to users in a controlled, incremental manner, using automated analysis to verify each step before proceeding, and automatically rolling back if issues are detected.

**Key strategies**:
- **Canary Deployment**: Route a small percentage of traffic to the new version; increase gradually
- **Blue/Green Deployment**: Deploy new version alongside old; switch all traffic at once after verification
- **Rolling Update**: Replace old instances with new ones, one at a time
- **A/B Testing**: Route subsets of users to different versions for comparison (often longer-term)
- **Feature Flags**: Decouple deployment from release—deploy code, but enable/disable features independently

## Concept Explanation

### Deployment Strategies Compared

```
ROLLING UPDATE:
  Old: [v1] [v1] [v1]
  Step: [v1] [v1] [v2]  ← One pod replaced
  Step: [v1] [v2] [v2]
  Done: [v2] [v2] [v2]
  
  Pros: No extra infrastructure, gradual rollout
  Cons: Mixed versions during rollout, no easy rollback (must roll forward or undo)


BLUE/GREEN:
  Blue: [v1] [v1] [v1]  (production)
  Green: [v2] [v2] [v2]  (staged, not serving)
  
  Switch: All traffic → Green
  Rollback: All traffic → Blue (instant)
  
  Pros: Instant rollback, no mixed versions
  Cons: 2x infrastructure during deployment


CANARY:
  Production: [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1]
  Canary:                                [v2] [v2]
  
  Step 1: 20% traffic to canary, analyze
  Step 2: 50% traffic to canary, analyze  
  Step 3: 100% traffic to canary
  
  Pros: Gradual exposure, automated analysis between steps
  Cons: Complex traffic routing, requires service mesh or smart load balancer
```

### Canary Deployment with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: order-service-canary   # Service for canary pods
      stableService: order-service-stable   # Service for stable pods
      
      steps:
      - setWeight: 10    # 10% to canary
      - pause: {duration: 120s}  # Wait 2 minutes, analyze metrics
      
      - setWeight: 30    # 30% to canary
      - pause: {duration: 120s}
      
      - setWeight: 50
      - pause: {duration: 300s}  # Wait 5 minutes at 50%
      
      - setWeight: 100   # Full promotion
      
      # Automated analysis during canary
      analysis:
        templates:
        - templateName: error-rate-check
        - templateName: latency-check
        args:
        - name: service-name
          value: order-service
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
  - name: error-rate
    interval: 30s
    successCondition: result[0] <= 0.01  # Error rate must stay < 1%
    provider:
      prometheus:
        address: http://prometheus.monitoring:9090
        query: |
          sum(rate(http_requests_total{service="order-service",status=~"5.."}[2m]))
          /
          sum(rate(http_requests_total{service="order-service"}[2m]))
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency-check
spec:
  metrics:
  - name: p99-latency
    interval: 30s
    successCondition: result[0] <= 0.5  # p99 < 500ms
    provider:
      prometheus:
        address: http://prometheus.monitoring:9090
        query: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket{service="order-service"}[2m])) 
            by (le))
```

### Automated Rollback

```yaml
# If analysis fails at any step, automatically rollback
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 120s}
      - setWeight: 100
      
      analysis:
        templates:
        - templateName: error-rate-check
        - templateName: latency-check
        
      # Automatic rollback if analysis fails
      abortScaleDownDelaySeconds: 600  # Keep canary pods for 10 min for debugging
```

### Feature Flags (Decouple Deploy from Release)

```python
# LaunchDarkly SDK: separate code deployment from feature release
import ldclient

ld_client = ldclient.LDClient("sdk-key")

def create_order(request):
    user = get_current_user()
    
    # Feature flag: is this user seeing the new checkout flow?
    if ld_client.variation('new-checkout-flow', user.key, False):
        return new_checkout_flow(request)
    else:
        return legacy_checkout_flow(request)

# Deploy code with new checkout → feature flag off → zero users see it
# Enable for 1% of users → monitor → 10% → 50% → 100%
# Problem found? → Turn flag off instantly (no redeploy needed)
# The "kill switch" is the most powerful risk mitigation in progressive delivery
```

```python
# OpenFeature: vendor-neutral feature flagging standard
from openfeature import api
from openfeature.provider import launchdarkly_provider

api.set_provider(launchdarkly_provider.LaunchDarklyProvider("sdk-key"))
client = api.get_client()

if client.get_boolean_value("new-checkout-flow", False):
    return new_checkout(request)
else:
    return legacy_checkout(request)
```

### Automated Analysis (Metrics-Driven Promotion)

```python
class AutomatedPromotion:
    """
    Automatically decide whether to promote, pause, or rollback
    based on comparing canary metrics against baseline.
    """
    
    def evaluate_canary(self, canary_metrics, baseline_metrics):
        analysis = {
            'promote': True,
            'findings': []
        }
        
        # Error rate check
        if canary_metrics.error_rate > baseline_metrics.error_rate * 2:
            analysis['promote'] = False
            analysis['findings'].append({
                'severity': 'CRITICAL',
                'metric': 'error_rate',
                'canary': f"{canary_metrics.error_rate:.2%}",
                'baseline': f"{baseline_metrics.error_rate:.2%}",
                'message': 'Error rate more than 2x baseline'
            })
        
        # Latency check
        if canary_metrics.p99_latency > baseline_metrics.p99_latency * 1.5:
            analysis['promote'] = False
            analysis['findings'].append({
                'severity': 'HIGH',
                'metric': 'p99_latency',
                'canary': f"{canary_metrics.p99_latency:.0f}ms",
                'baseline': f"{baseline_metrics.p99_latency:.0f}ms",
                'message': 'p99 latency more than 1.5x baseline'
            })
        
        # Saturation check (did we cause CPU spike?)
        if canary_metrics.cpu_utilization > 80:
            analysis['findings'].append({
                'severity': 'WARNING',
                'metric': 'cpu',
                'message': 'Canary CPU utilization above 80%'
            })
        
        return analysis
```

## Layman's Explanation

### The Restaurant New Menu Rollout
A restaurant chain wants to introduce a new menu item (software version). Progressive delivery approaches:

**Big Bang**: Replace all menus in all 200 locations overnight. If the new item causes food poisoning (critical bug), all 200 locations are affected simultaneously. Disaster.

**Canary (the namesake)**: The term comes from coal mining—miners brought canaries into mines. If the canary died (toxic gas), miners evacuated. In software: introduce the new menu in 2 locations (10%). Monitor for 2 hours—any food poisoning reports? Customer complaints about taste? If all clear, expand to 10 locations (50%). Still good? All 200 locations.

**Blue/Green**: Prepare all ingredients for both old and new menus in every kitchen (2x infrastructure). At 2 PM, swap all menus simultaneously. If there's a problem, swap back immediately (instant rollback). No mixed menus, no confusion.

**Feature Flags**: Ship the new menu ingredients to every location (deploy the code). But the menu still lists only old items ("feature flag off"). When ready, update the menu to include the new item for 10% of tables first. Problem? Remove the item from the menu instantly (turn feature flag off). No need to recall ingredients or change recipes.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Canary Analysis Metrics**: What metrics determine if a canary is healthy? Error rate and latency are universal, but domain-specific checks matter: "order completion rate" for e-commerce, "trade execution latency" for finance. Choose metrics that detect real user impact, not infrastructure telemetry.
- **Traffic Routing Implementation**: Service mesh (Istio, Linkerd) provides the most flexible traffic routing with weight-based splitting and header-based routing. API Gateway with weighted target groups is simpler but less granular. DNS-based routing is too coarse for canary (DNS caching, TTLs).
- **State and Schema Compatibility**: During canary, both old and new versions run simultaneously. Database schema changes must be backward-compatible (new version reads old data, old version ignores new columns). API changes must be compatible (new version can be called by old clients). This is the "expand-contract" pattern: add new column → deploy → migrate data → remove old column.
- **Feature Flag Technical Debt**: Feature flags accumulate. After the new checkout flow is 100% rolled out and stable, REMOVE the old code path and the feature flag. Old flags that are always-on become dead code and technical debt.

### Business Impact
- **Reduced Deployment Risk**: Deployments go from "major events requiring all-hands monitoring" to "automated processes that happen during lunch." Netflix deploys thousands of times per day because progressive delivery makes each individual deployment low-risk.
- **Faster Recovery**: A bad deployment caught at 10% canary affects 10% of users. Automatic rollback limits damage to minutes. A bad big-bang deployment affects 100% of users and may take hours to rollback (database schema reversions, etc.).
- **Experimentation Culture**: When deployments are low-risk, teams deploy more frequently. When teams deploy more frequently, they experiment more. Progressive delivery enables the experimentation culture that drives product innovation.

## On-Prem, AWS, GCP, Azure Examples

```yaml
# Istio VirtualService: traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: order-service
        subset: v2
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 90
    - destination:
        host: order-service
        subset: v2
      weight: 10
```

```hcl
# AWS CodeDeploy: canary deployment for Lambda/ECS
resource "aws_codedeploy_deployment_group" "canary" {
  deployment_config_name = "CodeDeployDefault.ECSLinear10PercentEvery1Minute"
  # Shifts 10% traffic every 1 minute with automated rollback
}
```

```bash
# GCP Cloud Run: traffic splitting
gcloud run services update-traffic order-service \
  --to-revisions LATEST=10,PREVIOUS=90

# Azure: App Service deployment slots (Blue/Green)
az webapp deployment slot swap \
  --name order-service \
  --slot staging \
  --target-slot production
```

## Summary

| Strategy | Rollback Speed | Infrastructure Cost | User Impact During Deploy | Best For |
|----------|---------------|--------------------|--------------------------|----------|
| Rolling Update | Slow (roll forward) | 1x | Mixed versions | Stateless apps, Kubernetes Deployments |
| Blue/Green | Instant | 2x | None (atomic switch) | Critical systems, zero-downtime |
| Canary | Automated (minutes) | 1.1-2x | Gradual (small %) | User-facing services, automated analysis |
| Feature Flags | Instant (no redeploy) | 1x | None (per-user toggle) | UI changes, experimental features |

Progressive delivery makes deployments boring—and that's the goal. A deployment should be as uneventful as saving a file. Canary deployments with automated metric analysis and automatic rollback transform deployment from a source of anxiety into a routine operation. The architect's role is to design the traffic routing, metric analysis, and rollback mechanisms that make this possible, and to ensure database schema changes, API compatibility, and feature flag hygiene are managed throughout the delivery lifecycle.
