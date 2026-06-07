# Metrics & Monitoring

## Introduction
Metrics are the quantitative measurements that tell you how your system is performing. They are the foundation of observability—before you can debug an issue or optimize performance, you need to know what "normal" looks like. The discipline of metrics and monitoring evolved from simple SNMP polling of server CPU to sophisticated time-series databases (Prometheus, InfluxDB) and dimensional metrics systems that can answer arbitrary questions about system behavior across hundreds of dimensions.

The RED method (Rate, Errors, Duration) and USE method (Utilization, Saturation, Errors) provide systematic frameworks for what to measure. Combined with SLOs (Service Level Objectives) and dashboards, metrics transform system operations from reactive firefighting to data-driven decision making. A solution architect designs not just what to measure, but the entire metrics pipeline: collection, storage, querying, alerting, and visualization.

## Definition

**Metrics** are numeric measurements collected over time. Key types:

- **Counter**: Monotonically increasing value (total requests, total errors)
- **Gauge**: Value that can go up or down (current memory usage, active connections)
- **Histogram**: Distribution of values (request latency, response size)
- **Summary**: Similar to histogram but with client-side quantile calculation

**Monitoring** is the process of collecting, processing, aggregating, and displaying real-time quantitative data about a system.

## Concept Explanation

### The RED Method (Service-Level Metrics)

```
For EVERY service, measure:

RATE:    Number of requests per second
         → Identifies traffic spikes and patterns

ERRORS:  Number of failed requests per second
         → Identifies service degradation

DURATION: Time to process each request (p50, p95, p99)
         → Identifies performance issues
```

```python
from prometheus_client import Counter, Histogram, generate_latest
import time
import functools

# Define metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
)

ERROR_COUNT = Counter(
    'http_errors_total',
    'Total HTTP errors',
    ['method', 'endpoint', 'error_type']
)

# Decorator to instrument any endpoint
def observed(endpoint):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            start = time.time()
            try:
                result = func(*args, **kwargs)
                status = result[1] if isinstance(result, tuple) else 200
                REQUEST_COUNT.labels(
                    method='POST', endpoint=endpoint, status=status
                ).inc()
                REQUEST_DURATION.labels(
                    method='POST', endpoint=endpoint
                ).observe(time.time() - start)
                return result
            except Exception as e:
                ERROR_COUNT.labels(
                    method='POST', endpoint=endpoint, error_type=type(e).__name__
                ).inc()
                raise
        return wrapper
    return decorator

@app.route('/api/orders', methods=['POST'])
@observed('/api/orders')
def create_order():
    order = order_service.create(request.json)
    return jsonify(order.to_dict()), 201
```

### The USE Method (Resource-Level Metrics)

```
For EVERY resource (CPU, memory, disk, network):

UTILIZATION:  % of resource in use
              → CPU: 85% (approaching saturation)
              → Memory: 92% (need to scale)

SATURATION:   Queue depth, backlog
              → CPU run queue length: 12 (CPUs overloaded)
              → Network tx queue drops

ERRORS:       Resource-specific errors
              → Disk: I/O errors
              → Network: Interface errors, collisions
```

### Cardinality and Dimensionality

```python
# LOW CARDINALITY (good):
# Few unique label values
http_requests_total{method="GET", endpoint="/api/orders"}   # 5 methods × 20 endpoints = 100 series

# HIGH CARDINALITY (dangerous):
# Many unique label values
http_requests_total{user_id="abc123", request_id="req-xyz"}  # millions of users = millions of series
# Prometheus recommends < 100,000 total series
# Each unique label combination creates a new time series
```

### Aggregation and Retention

```python
# Metrics storage strategy: downsampling over time
class MetricsRetention:
    """
    Raw data: keep 2 weeks (1-minute resolution)
    1-hour rollups: keep 3 months
    1-day rollups: keep 2 years
    """
    
    def downsample(self, metric_name, source_resolution, target_resolution):
        if source_resolution == '1m' and target_resolution == '1h':
            query = f"""
                SELECT 
                    time_bucket('1 hour', time) as hour,
                    avg(value) as avg_value,
                    min(value) as min_value,
                    max(value) as max_value,
                    count(*) as sample_count
                FROM raw_metrics
                WHERE name = '{metric_name}'
                  AND time > now() - interval '2 weeks'
                  AND time < now() - interval '1 day'  -- don't downsample today
                GROUP BY hour
            """
        return query
```

### Prometheus Architecture

```yaml
# Prometheus deployment (pull-based model)
global:
  scrape_interval: 15s     # How often to collect metrics
  evaluation_interval: 15s # How often to evaluate alerting rules

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      
  - job_name: 'postgresql'
    static_configs:
      - targets: ['postgres-exporter:9187']
    
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

# Federation: Prometheus scrapes another Prometheus
# Hierarchical: per-cluster Prometheus → global Prometheus → Grafana

# Long-term storage: Thanos or Cortex
# These add: global query view, downsampling, S3/GCS backup
```

### Grafana Dashboards

```json
{
  "dashboard": {
    "title": "Order Service - RED Dashboard",
    "panels": [
      {
        "title": "Request Rate (per second)",
        "targets": [{
          "expr": "rate(http_requests_total{endpoint=~\"/api/orders.*\"}[5m])"
        }]
      },
      {
        "title": "Error Rate (%)",
        "targets": [{
          "expr": "sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m])) * 100"
        }],
        "alert": {
          "conditions": [{
            "evaluator": { "params": [5], "type": "gt" }
          }]
        }
      },
      {
        "title": "Request Duration (p99)",
        "targets": [{
          "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))"
        }]
      },
      {
        "title": "Database Connections",
        "targets": [{
          "expr": "pg_stat_database_numbackends{datname=\"orders\"}"
        }]
      }
    ]
  }
}
```

## Layman's Explanation

### The Car Dashboard
Your car's dashboard is a metrics system:

- **Speedometer (Gauge)**: Current speed—goes up and down. Like current memory usage.
- **Odometer (Counter)**: Total miles driven—only goes up. Like total requests served.
- **Trip Computer (Histogram)**: Average speed over this trip, max speed, fuel efficiency distribution. Like request latency distribution.
- **Check Engine Light (Alert)**: When a metric crosses a threshold. Like error rate > 5%.
- **Fuel Gauge (Gauge with threshold)**: 75% = fine. 10% = warning light. 0% = stranded.

Without a dashboard, you'd have no idea if the engine is overheating until steam pours from the hood. Metrics give you visibility before the breakdown.

### Why Percentiles Matter (Not Averages)
If your average response time is 100ms, that sounds great. But:
- 90% of requests: 10ms (cached)
- 9% of requests: 200ms (database query)
- 1% of requests: 5,000ms (complex report generation)

Average: (90*10 + 9*200 + 1*5000)/100 = 100ms ✓
But 1% of users wait 5 seconds—and that 1% might be your biggest customers running complex queries.

This is why we measure p50, p95, p99—not averages.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Pull vs Push Model**: Prometheus (pull: server scrapes endpoints) vs StatsD/Telegraf (push: agents send to server). Pull gives the monitoring system control over collection. Push is simpler for ephemeral workloads (Lambda, batch jobs). Modern systems often use push to a gateway, then Prometheus scrapes the gateway.
- **Cardinality Management**: Every label dimension multiplies the number of time series. `{method} × {endpoint} × {status} × {instance} = 5 × 200 × 10 × 100 = 1,000,000 series`—approaching Prometheus limits. Design label schemas before instrumenting. Never put user IDs, request IDs, or session IDs in metric labels.
- **Long-Term Storage**: Prometheus is not designed for long-term storage (> 30 days). Thanos, Cortex, or Mimir provide: downsampling, compaction, S3/GCS backing, and global querying across multiple Prometheus instances.
- **Exporter Strategy**: Every infrastructure component needs an exporter (Node Exporter, PostgreSQL Exporter, Redis Exporter, Blackbox Exporter, JMX Exporter). The architect must ensure every component in the system has a metrics endpoint available to the monitoring system.

### Business Impact
- **Capacity Planning**: Usage trends over months inform procurement decisions. "Our database grows 15%/month—we need to shard in 6 months" is actionable from metrics. Without metrics, capacity planning is guesswork.
- **Cost Allocation**: Per-service request counts × cost per request = per-service cloud cost. This enables showback/chargeback to business units and identifies cost optimization targets.
- **Incident Prevention**: Metrics-based alerting catches issues before users notice. Disk usage at 85% with a 10% weekly growth rate = disk full in 1.5 weeks. Proactive alert prevents an outage.

## On-Premises Examples

### Prometheus + Node Exporter
```bash
# Node Exporter (system metrics)
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xzf node_exporter-*.tar.gz
./node_exporter --collector.systemd --collector.processes

# Endpoint: http://localhost:9100/metrics
# Exposes: CPU, memory, disk, network, filesystem metrics

# PostgreSQL Exporter
./postgres_exporter --web.listen-address=":9187"

# Prometheus server
./prometheus --config.file=prometheus.yml --storage.tsdb.retention.time=30d
```

### Telegraf + InfluxDB (Push-Based)
```bash
# Telegraf agent pushes to InfluxDB
telegraf --config telegraf.conf
# Collects: CPU, memory, disk, network, Docker, Kafka, Redis, MySQL, etc.

# Query with InfluxQL
influx -database 'telegraf' -execute \
  "SELECT mean(usage_idle) FROM cpu WHERE time > now() - 1h GROUP BY time(1m)"
```

## AWS Examples

### CloudWatch Metrics
```python
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch')

# Publish custom metric
cloudwatch.put_metric_data(
    Namespace='OrderService',
    MetricData=[
        {
            'MetricName': 'OrdersCreated',
            'Value': 150,
            'Unit': 'Count',
            'Timestamp': datetime.utcnow(),
            'Dimensions': [
                {'Name': 'Environment', 'Value': 'production'},
                {'Name': 'Region', 'Value': 'us-east-1'}
            ]
        }
    ]
)

# Query metric
response = cloudwatch.get_metric_statistics(
    Namespace='AWS/RDS',
    MetricName='DatabaseConnections',
    Dimensions=[{'Name': 'DBInstanceIdentifier', 'Value': 'orders-db'}],
    StartTime=datetime.utcnow() - timedelta(hours=1),
    EndTime=datetime.utcnow(),
    Period=300,  # 5-minute granularity
    Statistics=['Average', 'Maximum']
)
```

### CloudWatch Alarms
```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "orders-service-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 60
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "CPU > 80% for 3 minutes"

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]

  dimensions = {
    ServiceName = "order-service"
    ClusterName = "production"
  }
}
```

### Container Insights (EKS)
```hcl
# CloudWatch agent for EKS metrics
# Automatically collects:
# - Pod/container CPU, memory, network, disk
# - Service-level metrics
# - Cluster-level aggregate metrics
```

## GCP Examples

### Cloud Monitoring (formerly Stackdriver)
```python
from google.cloud import monitoring_v3

client = monitoring_v3.MetricServiceClient()

# Create custom metric descriptor
descriptor = monitoring_v3.MetricDescriptor()
descriptor.type = "custom.googleapis.com/orders/created"
descriptor.metric_kind = monitoring_v3.MetricKind.GAUGE
descriptor.value_type = monitoring_v3.MetricDescriptor.ValueType.INT64
client.create_metric_descriptor(name="projects/my-project", metric_descriptor=descriptor)

# Write time series
series = monitoring_v3.TimeSeries()
series.metric.type = "custom.googleapis.com/orders/created"
series.metric.labels["environment"] = "production"
point = series.points.add()
point.value.int64_value = 150
point.interval.end_time.seconds = int(time.time())
client.create_time_series(name="projects/my-project", time_series=[series])
```

### Managed Prometheus (GKE)
```bash
gcloud container clusters update my-cluster \
  --enable-managed-prometheus

# Google-managed Prometheus:
# - Collects metrics from GKE workloads
# - Stores in Monarch (Google's time-series DB)
# - Queryable via PromQL in Cloud Monitoring
# - No Prometheus server to manage
```

## Azure Examples

### Azure Monitor Metrics
```python
from azure.mgmt.monitor import MonitorManagementClient
from azure.identity import DefaultAzureCredential

client = MonitorManagementClient(DefaultAzureCredential(), subscription_id)

# List metric definitions for a resource
metrics = client.metrics.list(
    resource_uri="/subscriptions/.../virtualMachines/myVM",
    timespan="PT1H",
    interval="PT5M",
    metricnames="Percentage CPU",
    aggregation="Average"
)

for item in metrics.value:
    for timeserie in item.timeseries:
        for data in timeserie.data:
            print(f"Time: {data.time_stamp}, CPU: {data.average}%")
```

### Azure Monitor Alerts
```bash
az monitor metrics alert create \
  --name "high-cpu-alert" \
  --resource-group myResourceGroup \
  --scopes /subscriptions/.../virtualMachines/myVM \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action /subscriptions/.../actionGroups/ops-team
```

## Summary

| Metrics Tool | Best For | Model | Storage |
|-------------|----------|-------|---------|
| Prometheus | Kubernetes, microservices | Pull | Local TSDB + Thanos for long-term |
| Grafana | Visualization, dashboards | N/A | Connects to Prometheus, CloudWatch, etc. |
| CloudWatch | AWS-native monitoring | Push + built-in | Managed, 15-month retention |
| Cloud Monitoring | GCP-native monitoring | Push | Managed, 6-week retention |
| Azure Monitor | Azure-native monitoring | Push | Managed, 93-day retention |
| Datadog | Multi-cloud, SaaS | Agent push | Managed, configurable |

Metrics provide the eyes and ears of a distributed system. Without them, operations is blind. The RED method (application) and USE method (infrastructure) provide comprehensive coverage. Prometheus has become the de facto standard for cloud-native metrics due to its dimensional data model, powerful query language (PromQL), and Kubernetes-native integration. The architect's job is to ensure every component emits metrics, every metric has a purpose, and dashboards provide actionable insight rather than vanity graphs.
