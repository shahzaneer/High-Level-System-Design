# Alerting & On-Call

## Introduction
Alerting and on-call are the operational interfaces between your monitoring systems and your engineering team. A perfectly instrumented system with beautiful dashboards is worthless at 3 AM if nobody knows there's a problem. But the opposite—alerting on every minor fluctuation—creates alert fatigue, where engineers learn to ignore alerts because "it's always nothing." The art of alerting is designing signals that are actionable, reliable, and infrequent enough to maintain responder attention.

Google's SRE practices introduced key concepts: alert only on symptoms (user-visible impact), not causes (high CPU, memory pressure). Every alert should require human intelligence to resolve—otherwise it should be automated. And on-call rotations must be sustainable: no single person should be the only one who knows how to respond. The solution architect designs not just the monitoring infrastructure but the entire incident response system.

## Definition

**Alerting** is the automated detection of conditions that require human attention and the notification of responsible responders through appropriate channels.

**On-Call** is the practice of designating engineers who are responsible for responding to incidents during a specific time period, including the rotation schedule, escalation policies, and operational procedures.

**Key metrics for alerting quality**:
- **MTTA (Mean Time to Acknowledge)**: How quickly an alert is picked up
- **MTTR (Mean Time to Resolve)**: How quickly an incident is resolved
- **Alert-to-Signal Ratio**: Percentage of alerts that were actionable (target: >80%)
- **False Positive Rate**: Alerts that fired but shouldn't have (target: <10%)

## Concept Explanation

### Alert Design Principles

```
GOOD ALERT:
  ✓ Indicates a user-visible problem
  ✓ Requires human intelligence to resolve
  ✓ Contains a clear symptom description
  ✓ Links to a runbook with resolution steps
  ✓ Fires infrequently (< 1 per shift)
  ✓ Has a tested, working notification path

BAD ALERT:
  ✗ Fires on a metric threshold without user impact
  ✗ Could be resolved by auto-scaling
  ✗ Contains "CPU > 80% on server-14" with no context
  ✗ No runbook; responder must debug from scratch
  ✗ Fires 50 times per shift (crying wolf)
  ✗ Goes to an email nobody reads
```

### Alerting on Symptoms, Not Causes

```python
# BAD: Alert on cause
# ALERT: CPU utilization > 90% on order-service
# Problem: CPU spikes can be normal (batch processing, traffic surge)
#          Responder can't tell if users are impacted

# GOOD: Alert on symptom
# ALERT: Order API p99 latency > 2 seconds AND error rate > 1%
# Problem: Users ARE experiencing slowness and errors
#          Responder knows this is real user impact

# Example Prometheus alerting rules
groups:
  - name: symptom-based-alerts
    rules:
      # BAD - alerts on cause
      - alert: HighCPU
        expr: cpu_utilization > 0.9
        # Don't do this! CPU can be high while service is fine
      
      # GOOD - alerts on symptom (user experience)
      - alert: OrderAPIDegraded
        expr: |
          (
            histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{endpoint="/api/orders"}[5m])) > 2
            AND
            rate(http_requests_total{endpoint="/api/orders",status=~"5.."}[5m]) / rate(http_requests_total{endpoint="/api/orders"}[5m]) > 0.01
          )
        for: 5m
        labels:
          severity: page
        annotations:
          summary: "Order API is degraded"
          description: "p99 latency > 2s AND error rate > 1% for 5 minutes"
          runbook_url: "https://wiki.internal.com/runbooks/order-api-degraded"
```

### Severity Levels

```
SEV-1 (Critical): Service completely down or major data loss
  → Page on-call engineer immediately (PagerDuty, OpsGenie)
  → Wake them up at 3 AM—this is worth it
  → Incident Commander activated
  Example: 100% of users seeing 500 errors, database corruption

SEV-2 (Major): Service degraded but functional
  → Page on-call engineer (may wait for business hours if at night)
  → Workaround available
  Example: Search is slow (5s response time), payment retries needed

SEV-3 (Minor): Partial degradation, low impact
  → Create ticket; respond during business hours
  → No immediate page
  Example: Admin dashboard not loading, non-critical batch job failed

SEV-4 (Cosmetic): No user impact
  → Backlog ticket; fix when convenient
  Example: Typo in UI, non-critical log warning
```

### On-Call Rotation Design

```python
from datetime import datetime, timedelta

class OnCallRotation:
    def __init__(self, team_members, rotation_period='weekly'):
        self.members = team_members
        self.period = rotation_period
        self.schedule = {}
        self.override = {}  # Vacation overrides
    
    def get_on_call(self, date=None):
        """Determine who is on call for a given date"""
        if date is None:
            date = datetime.now()
        
        # Check for manual overrides first (vacations, swaps)
        for override_date, member in self.override.items():
            if override_date.date() == date.date():
                return member
        
        # Calculate from rotation
        week_number = date.isocalendar()[1]
        member_index = week_number % len(self.members)
        primary = self.members[member_index]
        secondary = self.members[(member_index + 1) % len(self.members)]
        
        return {
            'primary': primary,
            'secondary': secondary,
            'manager': self.escalation_manager
        }
    
    def find_available_responder(self):
        """If primary doesn't acknowledge, escalate to secondary"""
        # PagerDuty/OpsGenie handles this automatically
        # This is the logic they implement
        escalation_chain = [
            self.get_on_call()['primary'],
            self.get_on_call()['secondary'],
            self.escalation_manager
        ]
        
        for responder in escalation_chain:
            if notify_and_wait(responder, timeout_minutes=5):
                return responder
        
        return None  # Nobody available—major incident
```

### Runbook Structure

```markdown
# Runbook: Order API Degraded

## Symptoms (What triggered this alert?)
- Order API p99 latency > 2 seconds for 5+ minutes
- Order API error rate > 1% for 5+ minutes

## Impact
- Customers unable to place or view orders
- Revenue impact: ~$5,000/minute during peak hours

## Immediate Triage (do these within 2 minutes)
1. Check dashboard: https://grafana.internal.com/d/orders
2. Check recent deployments: `kubectl rollout history deployment/order-service`
3. Check dependency status:
   - Payment gateway status: https://status.stripe.com
   - Database status: AWS RDS console → orders-db

## Common Causes and Fixes

### Cause 1: Database connection pool exhausted
  Symptom: pg_stat_activity count > 100
  Fix: `kubectl rollout restart deployment/order-service`
  Rollback: None needed (rolling restart)

### Cause 2: Downstream payment service timeout
  Symptom: Most latency in "charge_payment" span (check Jaeger)
  Fix: If Stripe status page shows incident → wait for resolution
        Enable degraded mode: `kubectl set env deployment/order-service PAYMENT_MODE=deferred`
  Rollback: `kubectl set env deployment/order-service PAYMENT_MODE=live`

### Cause 3: Cache eviction cascade
  Symptom: Redis hit rate dropped from 95% → 30%
  Fix: Pre-warm cache: `kubectl exec -it deploy/cache-warmer -- python warm_cache.py`
  Rollback: N/A

## Escalation
If unresolved after 30 minutes → Escalate to SRE Manager
If unresolved after 60 minutes → Escalate to VP Engineering
If revenue impact > $50K → Escalate to CTO

## Post-Incident
Link to post-mortem template: https://wiki.internal.com/postmortems/template
```

### Automated Runbooks (Self-Healing)

```python
class AutoRemediator:
    """
    For well-known issues, automate the fix.
    Don't page a human for something a script can fix.
    """
    
    def __init__(self):
        self.rules = {
            'database_connection_exhausted': self.restart_service,
            'disk_space_low': self.cleanup_logs,
            'stuck_deployment': self.rollback_deployment,
            'memory_leak': self.restart_service
        }
    
    def evaluate_and_act(self, alert):
        if alert.name in self.rules:
            action = self.rules[alert.name]
            success = action(alert)
            
            if success:
                # Auto-resolved—no human needed
                self._close_alert(alert, resolution='auto-remediated')
            else:
                # Auto-remediation failed—escalate to human
                self._escalate(alert, note='Auto-remediation attempted but failed')
    
    def restart_service(self, alert):
        """Restart the affected service"""
        service_name = alert.labels.get('service')
        try:
            subprocess.run(['kubectl', 'rollout', 'restart', 
                          f'deployment/{service_name}'], 
                          timeout=60, check=True)
            return True
        except:
            return False
```

## Layman's Explanation

### The Fire Station
Your monitoring system is a fire station's alarm board. The on-call engineer is the firefighter on duty:

**Good Alerting**: When a smoke detector goes off (symptom of fire), the alarm sounds, lights flash, and the firefighter (on-call) responds. The alarm board shows which building, which floor, and a map of hydrant locations (runbook).

**Bad Alerting**: The alarm goes off every time someone burns toast (CPU > 80% but no user impact). After the 50th burnt-toast alarm, the firefighter starts ignoring the alarm entirely. When a real fire happens, nobody responds.

**On-Call Rotation**: Firefighters work in shifts. Nobody works 24/7/365. When shift A ends, shift B takes over. If the primary firefighter doesn't confirm they're responding within 5 minutes, the alarm escalates to the secondary, then the chief.

**Automated Response**: The sprinkler system (auto-remediation) handles small fires automatically—no firefighter needed. Only fires that need human judgment (complex, dangerous, novel) trigger a human response.

### The 3 AM Test
Before creating an alert, ask: "Is this problem worth waking someone up at 3 AM?" If not, it should be a ticket, not a page.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Alert Routing**: Which alerts go to which team? Service-specific alerts go to the owning team. Platform/infrastructure alerts go to SRE. Never route every alert to every team. Each team should only receive alerts for systems they own and can fix.
- **Escalation Policy**: Primary on-call → secondary → manager → director → VP. Escalation policies must be defined and enforced in the paging tool (PagerDuty/OpsGenie). Manual escalation (Slack DMs, phone calls) fails when the person calling is also stressed.
- **On-Call Load**: Sustainable on-call means <25% of work time spent on incidents. If on-call is a full-time job, you need a dedicated SRE team. If it's 80% of an engineer's time, they can't do feature work and they'll burn out.
- **Alerting During Business Hours Only**: Most alerts don't need 3 AM response. Consider: "page 24/7" for SEV-1 only, "page during business hours" for SEV-2, "create ticket" for everything else.

### Business Impact
- **Incident Resolution Speed**: A well-designed alert with a runbook link reduces MTTR from 2 hours (investigating from scratch) to 20 minutes (following the runbook). For a revenue-critical service losing $5,000/minute, that's a $500,000 difference.
- **Engineer Burnout**: Poor on-call practices cause burnout and attrition. Engineers leave teams with unsustainable on-call more than any other factor except compensation. Good on-call design is a retention strategy.
- **Customer Trust**: Customers notice the difference between "we know about the outage and are fixing it" (alert fired, runbook followed) and "what outage? let me check" (no alerting, first learning from customer complaint).

## On-Premises Examples

### Alertmanager (Prometheus)
```yaml
# alertmanager.yml
global:
  slack_api_url: 'https://hooks.slack.com/services/...'

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    
    - match:
        severity: warning
      receiver: 'slack-warnings'
    
    - match:
        team: sre
      receiver: 'sre-team'
      routes:
        - match:
            service: order-service
          receiver: 'order-team'

receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'PD_ROUTING_KEY'
        severity: critical
  
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warnings'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ .CommonAnnotations.description }}'
  
  - name: 'order-team'
    webhook_configs:
      - url: 'http://opsgenie-webhook/order-team'
```

### PagerDuty + Runbook Integration
```python
import pdpyras

session = pdpyras.EventsAPISession('PD_ROUTING_KEY')

# Trigger alert
dedup_key = session.trigger(
    summary='Order API p99 latency exceeded threshold',
    source='prometheus',
    severity='critical',
    component='order-service',
    group='order-api',
    class_type='latency',
    custom_details={
        'p99_latency': '2.3s',
        'error_rate': '2.1%',
        'dashboard_url': 'https://grafana.internal.com/d/orders',
        'runbook_url': 'https://wiki.internal.com/runbooks/order-api-degraded',
        'recent_deployments': 'order-service:v2.4.1 (deployed 10 min ago)'
    },
    links=[
        {'href': 'https://grafana.internal.com/d/orders', 'text': 'Grafana Dashboard'},
        {'href': 'https://wiki.internal.com/runbooks/order-api-degraded', 'text': 'Runbook'}
    ]
)

# Resolve alert
session.resolve(dedup_key)
```

## AWS + GCP + Azure Examples

### AWS: CloudWatch → SNS → PagerDuty
```hcl
resource "aws_sns_topic" "alerts" {
  name = "cloudwatch-alerts"
}

resource "aws_sns_topic_subscription" "pagerduty" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "https"
  endpoint  = "https://events.pagerduty.com/integration/XXX/enqueue"
}
```

### GCP: Cloud Monitoring → PagerDuty
```bash
gcloud monitoring channels create \
  --display-name="PagerDuty" \
  --type=pagerduty \
  --channel-labels=service_key=PD_INTEGRATION_KEY

gcloud monitoring policies create \
  --notification-channels=PAGERDUTY_CHANNEL_ID \
  --condition-filter='...'
```

### Azure: Monitor → ITSM (ServiceNow/PagerDuty)
```bash
az monitor action-group create \
  --name on-call \
  --resource-group myResourceGroup \
  --action itsm ITSMConnection \
  --itsm-receiver-name ServiceNow
```

## Summary

| Alerting Aspect | Recommendation |
|----------------|----------------|
| What to alert on | Symptoms (user impact), not causes (CPU, memory) |
| Alert frequency | < 2 per on-call shift (target) |
| Auto-remediation | Automate before alerting a human |
| Escalation | Primary → Secondary → Manager → Director |
| On-call rotation | Weekly; avoid shifts > 1 week |
| Runbooks | Every alert must link to a runbook |
| Post-incident | Blameless post-mortem within 48 hours |
| Alert fatigue prevention | > 50% of alerts should result in action (not snoozed/ignored) |

Alerting and on-call are the human-facing side of reliability. The best monitoring infrastructure in the world provides zero value if alerts are ignored due to fatigue or if nobody knows how to respond. Design alerting as a product—its users are the on-call engineers. Make it actionable, respectful of their time, and continuously improved through post-incident reviews. A system that wakes someone up at 3 AM should be so well-designed that the responder knows within 30 seconds: what's wrong, how severe it is, and exactly where to find the runbook to fix it.
