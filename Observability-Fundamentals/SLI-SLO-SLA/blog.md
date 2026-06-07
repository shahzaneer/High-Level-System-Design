# SLI, SLO & SLA

## Introduction
SLIs, SLOs, and SLAs form the quantitative foundation for service reliability. Popularized by Google's Site Reliability Engineering (SRE) book, this framework replaces subjective "the system feels slow" with objective, measurable reliability targets that drive operational decisions. The key insight is that 100% reliability is neither achievable nor desirable—at some point, the cost of additional reliability exceeds the cost of the downtime it prevents.

The SLI/SLO/SLA framework creates a shared language between engineering (SLOs define reliability budgets), product (trade off features vs reliability), and business (SLAs define contractual commitments). Error budgets—the amount of acceptable unreliability before action is required—transform reliability from a binary "system is up or down" to a continuous "system is 99.95% reliable, 0.05% budget remaining this month."

## Definition

**SLI (Service Level Indicator)**: A quantitative measure of some aspect of the service's behavior. Examples:
- Availability: `successful_requests / total_requests`
- Latency: `requests_completed_faster_than_500ms / total_requests`
- Throughput: `requests_per_second`
- Durability: `successfully_stored_bytes / total_bytes_written`
- Freshness: `time_since_last_successful_data_pipeline_run`

**SLO (Service Level Objective)**: A target value or range for an SLI, measured over a time window. Example: "99.9% availability over a rolling 30-day window"

**SLA (Service Level Agreement)**: A contractual agreement with users/customers about what happens if the SLO is violated. Example: "If availability drops below 99.5%, customer receives 10% service credit"

## Concept Explanation

### The SLI/SLO/SLA Relationship

```
                    __________________________________
                    |          SLA (Contract)          |
                    |  "99.5% uptime or 10% credit"    |
                    |  ┌──────────────────────────┐   |
                    |  │     SLO (Internal)        │   |
                    |  │   "99.9% availability"     │   |
                    |  │  ┌──────────────────┐     │   |
                    |  │  │  SLI (Measure)    │     │   |
                    |  │  │  success / total  │     │   |
                    |  │  └──────────────────┘     │   |
                    |  └──────────────────────────┘   |
                    └__________________________________┘

Key principle: SLO > SLA (SLO is STRICTER than SLA)
If SLO = 99.9% and SLA = 99.5%, you have a 0.4% "buffer"
SLO violation → engineering action
SLA violation → customer compensation + engineering action
```

### SLO Calculation

```python
class SLOCalculator:
    def __init__(self, slo_target, window_days=30):
        self.target = slo_target  # e.g., 0.999 (99.9%)
        self.window = timedelta(days=window_days)
    
    def calculate_sli(self, good_events, total_events):
        """SLI = good events / total events"""
        if total_events == 0:
            return 1.0  # No traffic = not violating
        return good_events / total_events
    
    def calculate_error_budget(self, total_events):
        """Error budget = total events × (1 - SLO target)"""
        return int(total_events * (1 - self.target))
    
    def calculate_error_budget_remaining(self, total_events, error_events):
        budget = self.calculate_error_budget(total_events)
        remaining = budget - error_events
        return max(0, remaining)
    
    def should_freeze_releases(self, total_events_this_month, 
                                error_events_this_month):
        """If error budget is exhausted, freeze feature releases"""
        remaining = self.calculate_error_budget_remaining(
            total_events_this_month, error_events_this_month
        )
        return remaining == 0

# Example: Order API
slo = SLOCalculator(slo_target=0.999)  # 99.9% availability

total_requests = 10_000_000
failed_requests = 15_000  # 0.15% failure rate

sli = slo.calculate_sli(
    total_requests - failed_requests, total_requests
)
# SLI = 99.85% — NOT meeting 99.9% SLO

error_budget = slo.calculate_error_budget(total_requests)
# Budget: 10,000,000 × 0.001 = 10,000 allowed failures
# Used: 15,000 failures
# Budget exhausted! Freeze releases and focus on reliability.
```

### Choosing the Right SLIs

```python
# The "Four Golden Signals" (Google SRE book)
class GoldenSignals:
    
    @staticmethod
    def latency(p50_target_ms, p99_target_ms):
        """
        Proportion of requests faster than target
        SLI = requests_faster_than_target / total_requests
        
        Don't use averages—they hide tail latency!
        """
        return {
            'p50_latency': {
                'target_ms': p50_target_ms,
                'slo': '99% of requests < 50ms'  # Example
            },
            'p99_latency': {
                'target_ms': p99_target_ms,
                'slo': '99% of requests < 500ms'
            }
        }
    
    @staticmethod
    def traffic():
        """Requests per second—helps identify if SLO violations 
           correlate with traffic spikes"""
        return 'http_requests_total{status!~"5.."}'
    
    @staticmethod
    def errors():
        """Proportion of failed requests
           SLI = (total - 5xx) / total"""
        return 'rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])'
    
    @staticmethod
    def saturation():
        """How "full" the service is
           e.g., CPU utilization, memory, queue depth"""
        return {
            'cpu': 'container_cpu_usage_seconds_total',
            'memory': 'container_memory_working_set_bytes',
            'goroutines': 'go_goroutines'
        }
```

### SLO-Based Alerting

```python
# PROMETHEUS ALERTING RULES FOR SLI/SLO

# Burn rate: how fast you're consuming error budget
# 1x burn rate: consuming budget at normal rate (will exhaust in 30 days)
# 10x burn rate: consuming budget 10x faster (will exhaust in 3 days)
# 100x burn rate: consuming budget 100x faster (will exhaust in 7.2 hours)

# Alert: high burn rate (consuming budget fast)
groups:
  - name: slo-alerts
    rules:
      - alert: HighErrorBurnRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h]))
            /
            sum(rate(http_requests_total[1h]))
          ) > (14.4 * (1 - 0.999))
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error budget burning at high rate"
          description: "SLO: 99.9%. Current error rate consuming budget {{ $value | humanizePercentage }}x faster than normal."
      
      - alert: SLOAlmostExhausted
        expr: |
          (1 - (
            sum(rate(http_requests_total{status!~"5.."}[30d]))
            /
            sum(rate(http_requests_total[30d]))
          )) > 0.0009
        labels:
          severity: warning
        annotations:
          summary: "Error budget 90% exhausted"
```

### Multi-Window, Multi-Burn-Rate Alerts

```python
class SLIAlertDesign:
    """
    Google SRE's recommended alerting design:
    - Short window (1h): detects fast burns (major incidents)
    - Long window (6h): detects slow burns (steady degradation)
    - Different burn rates for different severity levels
    """
    
    ALERT_CONFIGS = {
        'critical': {
            'short_window': {'hours': 1, 'burn_rate': 14.4},  # 2% budget in 1h
            'long_window': {'hours': 6, 'burn_rate': 6.0}      # 5% budget in 6h
        },
        'warning': {
            'short_window': {'hours': 5, 'burn_rate': 6.0},    # 1% budget in 5h
            'long_window': {'hours': 72, 'burn_rate': 3.0}     # 3% budget in 3 days
        }
    }
    
    def calculate_threshold(self, slo, window_hours, burn_rate):
        """Error rate threshold that triggers alert"""
        error_budget_share = burn_rate * window_hours / 720  # 720h = 30 days
        return error_budget_share * (1 - slo)
```

## Layman's Explanation

### The Monthly Delivery Budget
Your company promises customers "99% of packages delivered on time." This is the SLA—the external, contractual promise.

Internally, you set a stricter target: "99.5% on-time delivery." This is the SLO—the internal objective that gives you a buffer before customers notice.

Each month, you have a certain number of packages to deliver. If you deliver 10,000 packages, your error budget is 10,000 × (1 - 0.995) = 50 packages. You can have up to 50 late deliveries this month and still meet your SLO.

**Error Budget Use Cases**:
- Week 1: 5 late deliveries. Budget: 45 remaining. Business as usual.
- Week 2: 30 late deliveries (carrier strike). Budget: 15 remaining. Warning: stop risky changes.
- Week 3: 10 late deliveries. Budget: 5 remaining. Critical: freeze all non-critical deploys.
- Week 4: 0 late deliveries. Budget: 5 remaining. End month with SLO met (99.95%).

If you had 55 late deliveries (budget exhausted), you compensate customers (SLA violation) AND freeze feature development until reliability is restored.

### The Bowling Bumper Analogy
Setting an SLO is like putting up bowling bumpers:
- No bumpers (no SLO): balls go everywhere, nobody knows what "good" looks like
- Bumpers at the edge (SLA): at least you don't gutter every time
- Bumpers in the middle (SLO): you're being held to a higher standard than the absolute minimum

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **SLO is a Business Decision, Not a Technical One**: 99.9% availability costs 3-5x more than 99% availability. The SLO should be set by the business, based on user expectations and revenue impact of downtime. Engineers implement it; product owners define it.
- **Error Budget Governs Release Velocity**: When error budget is healthy, teams can deploy faster, experiment more, and take risks. When budget is exhausted, all non-critical changes freeze. This creates a natural feedback loop between reliability and velocity.
- **Too Many SLOs Is Worse Than None**: A service should have 2-5 key SLOs—typically availability and latency. Employees can remember 3-5 numbers. 30 SLOs across 10 services means nobody pays attention to any of them. Focus on what matters to users.
- **SLIs Must Be Measurable from the User's Perspective**: Database uptime is not a user-facing SLI. Users experience API availability and latency. Measure SLIs at the point where users interact with the system—the load balancer or API gateway.

### Business Impact
- **Data-Driven Reliability Decisions**: Instead of "the VP says we need five nines," SLOs create a data-driven discussion: "Our users tolerate 99.5% availability before abandoning carts. 99.9% would cost $500K/year more. Is the incremental 0.4% worth it?"
- **Reduced Operational Toil**: Alerting on SLO burn rates eliminates noisy alerts. Instead of "CPU > 80% on node 14" (which users don't care about), alert when error budget is burning faster than acceptable. Fewer, more meaningful alerts.
- **Contractual Protection**: SLAs with customers are legally binding. The 0.4% buffer between SLO (99.9%) and SLA (99.5%) protects the business from compensating customers for minor incidents. Without this buffer, every SLO violation is also an SLA violation.

## On-Premises Examples

### Prometheus SLO Rules
```yaml
groups:
  - name: order-service-slo
    interval: 30s
    rules:
      # SLI: Availability
      - record: job:http_requests_total:sum
        expr: sum(rate(http_requests_total[5m])) by (job)
      
      - record: job:http_errors_total:sum
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
      
      # Error budget burn rate (percentage per hour)
      - record: slo:error_budget_burn_rate
        expr: |
          (job:http_errors_total:sum / job:http_requests_total:sum) 
          / (1 - 0.999) * 100
```

### Google's SLO Dashboard (Grafana)
```json
{
  "dashboard": {
    "title": "Order Service - SLO Dashboard",
    "panels": [
      {
        "title": "Current SLI (30-day rolling)",
        "targets": [{
          "expr": "1 - (sum(increase(http_errors_total[30d])) / sum(increase(http_requests_total[30d])))"
        }]
      },
      {
        "title": "Error Budget Remaining",
        "targets": [{
          "expr": "1 - ((1 - sli_current) / (1 - 0.999))"
        }]
      },
      {
        "title": "Burn Rate (last 1 hour)",
        "targets": [{
          "expr": "slo:error_budget_burn_rate"
        }]
      }
    ]
  }
}
```

## AWS Examples

### CloudWatch Composite Alarms (SLO Alerts)
```hcl
resource "aws_cloudwatch_composite_alarm" "slo_burn_rate" {
  alarm_name        = "orders-slo-burn-rate-critical"
  alarm_description = "Error budget burning at critical rate"
  alarm_actions     = [aws_sns_topic.alerts.arn]

  alarm_rule = "ALARM(short-window-burn) AND ALARM(long-window-burn)"

  depends_on = [
    aws_cloudwatch_metric_alarm.short_window_burn,
    aws_cloudwatch_metric_alarm.long_window_burn
  ]
}
```

### CloudWatch ServiceLens (SLI Tracking)
```bash
# ServiceLens integrates X-Ray traces with CloudWatch metrics
# Automatically calculates availability and latency SLIs
# Visualizes service map with SLO status overlay
```

## GCP Examples

### Cloud Monitoring SLOs
```bash
# Create SLO for order service
gcloud monitoring service-level-objectives create \
  --service=order-service \
  --slo-id=availability \
  --goal=0.999 \
  --calendar-period=month \
  --display-name="Order API Availability"

# Create alert on SLO burn rate
gcloud monitoring policies create \
  --notification-channels=opsgenie-channel \
  --condition-filter='metric.type="monitoring.googleapis.com/slo/error_budget_burn_rate"' \
  --condition-threshold-value=10
```

### Error Budget Policy
```bash
gcloud monitoring error-budget-policies create \
  --service=order-service \
  --slo-id=availability \
  --measurement-window=3600s \
  --threshold=0.02  # Alert at 2% budget consumed in 1 hour
```

## Azure Examples

### Application Insights SLO
```python
# Application Insights automatically calculates:
# - Availability: failed_requests / total_requests
# - Server response time: p50, p95, p99
# - Failed requests by exception type

# Custom SLO alert with KQL
"""
let SLO = 0.999;
let errorBudget = 1 - SLO;
requests
| where timestamp > ago(30d)
| summarize 
    total = count(),
    failed = countif(success == false)
| extend sli = todouble(total - failed) / total
| extend budgetRemaining = (SLO - sli) / errorBudget
| where budgetRemaining < 0.2
"""
```

## Summary

| Concept | Who Sets It | Example | What Happens on Violation |
|---------|------------|---------|--------------------------|
| SLI | Engineering (measurement) | 99.95% of requests succeed | Nothing (it's just a measurement) |
| SLO | Engineering + Product | 99.9% availability over 30 days | Freeze releases, focus on reliability |
| SLA | Legal + Business + Product | 99.5% or 10% monthly credit | Customer compensation + engineering action |

SLI/SLO/SLA is a framework for making reliability a data-driven, business-aligned practice rather than a religious argument about "five nines." The architect's role is to design systems that can MEASURE reliability (SLI instrumentation), to work with product owners to SET appropriate reliability targets (SLO), and to ensure the contractual commitments (SLA) are looser than the SLO by a safe margin. The error budget is the control mechanism that balances the tension between feature velocity and service reliability.
