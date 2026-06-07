# Logging Architecture

## Introduction
Logging is the oldest and most universal observability signal. Before metrics dashboards and distributed traces, there were log files—and they remain the most detailed and flexible source of system intelligence. While metrics tell you WHAT is happening (error rate is 3%) and traces tell you WHERE it's happening (the payment service call is slow), logs tell you WHY (the payment service returned "connection timeout after 30s to stripe.com").

Modern logging architecture has evolved from grepping /var/log/messages on a single server to centralized, structured, and searchable log platforms handling terabytes per day. The ELK stack (Elasticsearch, Logstash, Kibana) pioneered this, followed by cloud-native alternatives (Grafana Loki, AWS CloudWatch Logs, Azure Log Analytics). The challenge for architects is designing a logging pipeline that is comprehensive enough for debugging, fast enough for incident response, and cost-effective enough for the volume of logs modern systems generate.

## Definition

**Logging** is the practice of recording discrete, timestamped events from applications and infrastructure into a structured or semi-structured format for later analysis, debugging, auditing, and compliance.

**Structured Logging** is the practice of emitting logs in a machine-parseable format (JSON) with consistent field names, enabling automated querying and analysis without regex parsing.

## Concept Explanation

### Structured vs Unstructured Logging

```python
# BAD: Unstructured log (requires regex to parse)
# 2024-06-15 10:30:15 ERROR OrderService: Failed to create order for user 42: Database timeout
logger.error(f"Failed to create order for user {user_id}: {error}")

# GOOD: Structured log (immediately queryable)
logger.error("Order creation failed", extra={
    "event": "order_creation_failed",
    "user_id": 42,
    "order_id": None,
    "error_type": "DatabaseTimeout",
    "error_message": str(error),
    "service": "order-service",
    "trace_id": "0af7651916cd43dd8448eb211c80319c",
    "span_id": "b7ad6b7169203331",
    "duration_ms": 1450
})

# Output (JSON):
# {
#   "timestamp": "2024-06-15T10:30:15.123Z",
#   "level": "ERROR",
#   "message": "Order creation failed",
#   "event": "order_creation_failed",
#   "user_id": 42,
#   "error_type": "DatabaseTimeout",
#   "trace_id": "0af7651916cd43dd8448eb211c80319c",
#   ...
# }
```

### Log Levels and Their Semantic Meaning

```python
import logging

# FATAL/CRITICAL: Service cannot continue; operator must intervene
logging.critical("Database corruption detected; shutting down to prevent data loss")
# → Page on-call immediately

# ERROR: Operation failed, but service continues
logging.error("Payment processing failed", extra={
    "order_id": "ORD-123", "error": "Stripe API timeout", "retry_count": 3
})
# → Alert if error rate exceeds threshold

# WARNING: Something unexpected but non-fatal
logging.warning("Cache hit rate dropped to 45% (baseline: 95%)")
# → Investigate during business hours

# INFO: Significant business events
logging.info("Order created", extra={
    "order_id": "ORD-123", "user_id": 42, "total": 99.99, "items_count": 3
})
# → Business analytics, audit trail

# DEBUG: Detailed diagnostic information
logging.debug("Query executed: SELECT * FROM orders WHERE user_id=42 (took 45ms)")
# → Enabled temporarily for debugging; not in production by default

# TRACE: Extremely detailed (function entry/exit, variable values)
logging.debug("Function validate_order entered with params: %s", params)
# → Almost never in production
```

### Log Aggregation Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                    LOG SOURCES                               │
│  App Logs │ Container Stdout │ Systemd │ Audit │ K8s Events │
└──────────┬──────────────────────────────────────────────────┘
           │
    ┌──────▼──────────┐
    │  LOG COLLECTOR   │  (Fluentd, Fluent Bit, Logstash, Vector)
    │  • Tail files    │
    │  • Parse JSON    │
    │  • Enrich (add   │
    │    pod name,     │
    │    node, cluster)│
    │  • Buffer to disk│
    └──────┬──────────┘
           │
    ┌──────▼──────────┐
    │  LOG AGGREGATOR  │  (Elasticsearch, Loki, S3 + Athena)
    │  • Index / Store │
    │  • Retention     │
    │  • Access control│
    └──────┬──────────┘
           │
    ┌──────▼──────────┐
    │  QUERY / VISUAL  │  (Kibana, Grafana, CloudWatch Console)
    │  • Search        │
    │  • Dashboards    │
    │  • Alerting      │
    └─────────────────┘
```

### Logging in Containers (Kubernetes)

```yaml
# Application logs to stdout/stderr (not files)
# Kubernetes captures container stdout/stderr automatically

apiVersion: v1
kind: Pod
spec:
  containers:
  - name: order-service
    image: order-service:latest
    # Application logs to stdout → Docker JSON driver → node file
    # Fluent Bit DaemonSet tails /var/log/containers/*.log
    # Ships to centralized logging backend

---
# Fluent Bit DaemonSet: log collector on every node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
spec:
  template:
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        config:
          [OUTPUT]
            Name  loki
            Match *
            host  loki.monitoring.svc
            port  3100
            labels job=fluentbit,cluster=production
```

### Log Query Languages

```sql
-- Kibana Query Language (KQL)
service: "order-service" AND level: "ERROR" AND error_type: "DatabaseTimeout"
AND @timestamp > now-1h

-- LogQL (Grafana Loki - PromQL-inspired for logs)
{service="order-service"} 
  |= "error"            -- contains "error"
  | json                -- parse as JSON
  | error_type="DatabaseTimeout"
  | duration_ms > 1000  -- slow errors (took > 1 second)
  | line_format "{{.user_id}} - {{.error_message}}"

-- CloudWatch Logs Insights
fields @timestamp, @message
| filter @message like /ERROR/
| filter service = 'order-service'
| stats count(*) by error_type, bin(5m)
| sort count desc

-- Kusto Query Language (Azure Log Analytics)
AppTraces
| where SeverityLevel == 3  -- Error
| where Properties.service == "order-service"
| summarize Count = count() by tostring(Properties.error_type), bin(TimeGenerated, 5m)
| order by Count desc
```

### Log Sampling and Cost Control

```python
class LogSampler:
    """
    Not every log needs to be stored forever.
    Sampling controls costs while preserving debugging capability.
    """
    
    def __init__(self):
        self.rules = {
            'ERROR': {'sample_rate': 1.0, 'retention': '90_days'},       # Keep all errors
            'WARNING': {'sample_rate': 1.0, 'retention': '30_days'},     # Keep all warnings
            'INFO': {'sample_rate': 0.10, 'retention': '14_days'},       # Sample 10% of info
            'DEBUG': {'sample_rate': 0.0, 'retention': '0_days'}         # Drop all debug
        }
    
    def should_log(self, level, event_type):
        rule = self.rules.get(level, {'sample_rate': 0.01})
        
        # Always log business-critical events regardless of level
        if event_type in ['order_created', 'payment_completed', 'user_login']:
            return True
        
        return random.random() < rule['sample_rate']
```

### Correlation: Logs + Traces + Metrics

```python
# The three pillars of observability, correlated:

# In your logs:
logger.info("Order processing started", extra={
    "trace_id": "0af7651916cd43dd8448eb211c80319c",  # Trace correlation
    "span_id": "b7ad6b7169203331",
    "order_id": "ORD-123",
    "user_id": 42
})

logger.info("Payment charged", extra={
    "trace_id": "0af7651916cd43dd8448eb211c80319c",  # Same trace_id
    "span_id": "e8cd7b6b9203331a",
    "payment_amount": 99.99,
    "payment_provider": "stripe"
})

# In Grafana:
# 1. See spike in HTTP 500 errors (metrics)
# 2. Click through to trace exemplar → full trace in Jaeger (traces)
# 3. Click on slow span → correlated log entries (logs)
# This is the "single pane of glass" for observability
```

## Layman's Explanation

### The Ship's Log
A ship's captain maintains a logbook (application log). Every significant event is recorded: "08:00 Departed port. Weather clear. 12:00 Slight swell. 14:30 Engine room reports temperature increase."

The log serves three purposes:
- **Navigation (Debugging)**: "The ship started veering at 14:30. Let me check what happened around that time."
- **Audit (Compliance)**: "The insurance company wants to verify we followed proper procedures during the storm."
- **Analysis (Optimization)**: "Looking at logs from the last 100 voyages, we consistently slow down at this latitude—the current is stronger than we thought."

### Structured Logging: The Spreadsheet vs The Paragraph
Unstructured logging is like writing the logbook in flowing prose: "At two-thirty PM on June 15th, the order service failed to create an order for user forty-two due to a database timeout error." You'd have to read every sentence to find all database timeout errors.

Structured logging is like keeping the logbook in a spreadsheet:
| Timestamp | Service | Event | User | Error |
|-----------|---------|-------|------|-------|
| 14:30:15 | order-service | order_failed | 42 | DatabaseTimeout |

Now you can instantly: "Show me all DatabaseTimeout errors in the last hour" or "Show me all events for user 42 today."

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Log Volume Estimation**: A mid-size microservices environment can generate 1TB+ of logs per day. The architect must estimate log volume and design a pipeline that can handle peak load (which is often 3-5x average). Under-provisioned log pipelines drop logs silently—the worst failure mode because you don't know what you're missing.
- **Structured Logging Schema**: Define a company-wide logging schema. Every service logs the same field names: `trace_id`, `user_id`, `service`, `event`, `error_type`. This enables cross-service querying. Without a standard schema, every debugging session starts with "what fields does this service log?"
- **Log Retention Architecture**: Hot storage (Elasticsearch, last 7 days) for fast queries. Warm storage (S3 with Athena, last 90 days) for slower queries. Cold storage (Glacier, 7+ years) for compliance. Each tier has different cost and query performance characteristics.
- **PII in Logs**: The #1 data privacy violation in logs. Credit card numbers, SSNs, email addresses, and auth tokens accidentally logged. Architecture must include log scrubbing (regex/pattern matching) BEFORE logs leave the application boundary, not after they're in the centralized store.

### Business Impact
- **Debugging Time**: Structured, centralized logging reduces debugging time from hours (SSH to each server, grep /var/log) to seconds (centralized query). For a team of 10 engineers, saving 30 minutes per incident × 20 incidents/month = 100 hours/month saved.
- **Compliance**: SOC 2, PCI DSS, HIPAA all require audit logging and log retention. Centralized, immutable logging with defined retention periods is a direct compliance deliverable.
- **Security Incident Investigation**: Without centralized logging, investigating a security incident requires accessing dozens of individual servers (which may have been compromised and had their logs wiped). Centralized, immutable log storage is essential for forensics.

## On-Premises Examples

### EFK Stack (Elasticsearch + Fluentd + Kibana)
```yaml
# Fluentd configuration
<source>
  @type tail
  path /var/log/app/*.log
  pos_file /var/log/td-agent/app.pos
  tag app.*
  <parse>
    @type json
  </parse>
</source>

<filter app.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    environment "production"
  </record>
</filter>

<match app.**>
  @type elasticsearch
  host elasticsearch.monitoring.svc
  port 9200
  logstash_format true
  logstash_prefix app-logs
  include_tag_key true
  flush_interval 5s
</match>
```

### Grafana Loki (Lightweight, Prometheus-Compatible)
```yaml
# Loki is cheaper than Elasticsearch—only indexes metadata, not full text
# promtail config (agent)
server:
  http_listen_port: 9080

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*.log
  
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
```

## AWS Examples

### CloudWatch Logs
```python
import boto3
import watchtower
import logging

# CloudWatch Logs handler for Python
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
cw_handler = watchtower.CloudWatchLogHandler(
    log_group='order-service',
    stream_name='production',
    send_interval=5  # Batch send every 5 seconds
)
logger.addHandler(cw_handler)

logger.info("Order created", extra={
    'order_id': 'ORD-123',
    'user_id': 42
})
```

```hcl
resource "aws_cloudwatch_log_group" "order_service" {
  name              = "order-service"
  retention_in_days = 30

  kms_key_id = aws_kms_key.logs.arn  # Encrypt logs at rest
}

resource "aws_cloudwatch_log_subscription_filter" "export" {
  name            = "export-to-s3"
  log_group_name  = aws_cloudwatch_log_group.order_service.name
  filter_pattern  = ""  # All logs
  destination_arn = aws_kinesis_firehose_delivery_stream.logs.arn
  role_arn        = aws_iam_role.log_export.arn
}

# Kinesis Firehose → S3 (long-term, queryable with Athena)
resource "aws_kinesis_firehose_delivery_stream" "logs" {
  destination = "extended_s3"
  # Compress and batch to S3
  # Convert to Parquet for Athena querying
}
```

### CloudWatch Logs Insights
```sql
-- Find slow order creations
fields @timestamp, @message
| filter event = "order_created"
| filter duration_ms > 1000
| stats count() as slow_orders,
    avg(duration_ms) as avg_duration,
    max(duration_ms) as max_duration
  by bin(1h)
| sort @timestamp desc
```

## GCP Examples

### Cloud Logging (formerly Stackdriver)
```python
from google.cloud import logging

client = logging.Client()
logger = client.logger('order-service')

# Structured log entry
logger.log_struct({
    'event': 'order_created',
    'order_id': 'ORD-123',
    'user_id': 42,
    'total': 99.99,
    'trace': 'projects/my-project/traces/0af7651916cd43dd8448eb211c80319c',
    'severity': 'INFO'
})

# Query in Logs Explorer:
# resource.type="k8s_container"
# severity>=ERROR
# jsonPayload.event="order_creation_failed"
```

### Log Router (Sink to BigQuery)
```bash
# Sink all logs to BigQuery for analytics
gcloud logging sinks create bigquery-sink \
  bigquery.googleapis.com/projects/my-project/datasets/logs \
  --log-filter='severity>=INFO'

# Sink only security logs to Pub/Sub for SIEM
gcloud logging sinks create security-sink \
  pubsub.googleapis.com/projects/my-project/topics/security-logs \
  --log-filter='logName="projects/my-project/logs/cloudaudit.googleapis.com"'
```

## Azure Examples

### Log Analytics
```kusto
// KQL: Find all errors grouped by type
AppTraces
| where TimeGenerated > ago(1h)
| where SeverityLevel == 3
| extend errorType = tostring(Properties.error_type)
| summarize Count = count() by errorType, bin(TimeGenerated, 5m)
| render timechart

// KQL: Correlate logs with traces
AppDependencies
| where TimeGenerated > ago(1h)
| where Name == "POST /api/orders"
| where Duration > 1000  // > 1 second
| project TimeGenerated, Name, Duration, OperationId
| join kind=inner (
    AppTraces
    | extend OperationId = tostring(Properties.OperationId)
) on OperationId
| project TimeGenerated, Name, Duration, Message, OperationId
```

### Azure Monitor Logs
```bash
az monitor log-analytics workspace create \
  --workspace-name app-logs \
  --resource-group myResourceGroup

# Query with Azure CLI
az monitor log-analytics query \
  --workspace app-logs \
  --analytics-query 'AppTraces | where TimeGenerated > ago(1h) | take 100'
```

## Summary

| Logging Platform | Best For | Storage Cost | Query Language |
|-----------------|----------|-------------|----------------|
| Elasticsearch + Kibana | Full-text search, large deployments | High (indexes everything) | KQL, Lucene |
| Grafana Loki | Kubernetes, Prometheus users | Low (indexes only labels) | LogQL (PromQL-like) |
| CloudWatch Logs | AWS-native | Medium | CloudWatch Logs Insights |
| Cloud Logging | GCP-native | Medium | Logs Explorer |
| Azure Log Analytics | Azure-native, KQL power | Medium | KQL |

Logging is the "why" of observability. When metrics show a spike in errors and traces show the slow span, logs reveal the root cause: "connection timeout after 30s to stripe.com." Modern logging architecture requires structured formats (JSON), centralized aggregation, cost-optimized retention tiers, and correlation with traces and metrics. The architect must design a logging pipeline that scales with traffic, protects sensitive data through scrubbing, and provides fast query capabilities for incident response while controlling storage costs through sampling and tiered retention.
