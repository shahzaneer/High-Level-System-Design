# Distributed Tracing

## Introduction
Distributed Tracing is the observability practice of tracking a single request as it propagates through dozens of microservices, databases, caches, and message queues. In a monolith, debugging a slow request was simple: check the application log. In a microservices architecture, a single user request might touch 15 services, each with its own logs, and identifying which service caused a 2-second delay requires stitching together the journey across all of them.

Google's Dapper paper (2010) introduced the concept to the industry, spawning open-source implementations like Zipkin (Twitter) and Jaeger (Uber). The OpenTelemetry project has since standardized distributed tracing as part of a unified observability framework, making it a first-class architectural concern rather than an optional debugging tool.

## Definition

**Distributed Tracing** is the practice of recording and correlating the path of a request as it flows through a distributed system, creating a visual representation (trace) of the entire journey and the time spent at each step (span).

**Key concepts**:
- **Trace**: The complete journey of a single request through all services, represented as a directed acyclic graph of spans
- **Span**: A single unit of work within a trace (e.g., an HTTP call, a database query, a cache lookup)
- **Span Context**: Metadata (trace ID, span ID, parent span ID, baggage) passed between services to correlate spans into a trace
- **Context Propagation**: The mechanism by which trace context is transmitted between services (HTTP headers, gRPC metadata, message queue headers)

## Concept Explanation

### Anatomy of a Trace

```
TRACE: "Create Order" (total: 450ms)
│
├── SPAN: API Gateway (10ms)
│   │   span_id: abc123, parent: none (root)
│   │
│   └── SPAN: Order Service (420ms)
│       │   span_id: def456, parent: abc123
│       │
│       ├── SPAN: Validate Order (5ms)
│       │   span_id: ghi789, parent: def456
│       │
│       ├── SPAN: Check Inventory (150ms)
│       │   │   span_id: jkl012, parent: def456
│       │   │
│       │   ├── SPAN: Redis Cache Lookup (5ms)  ← cache hit
│       │   │   span_id: mno345, parent: jkl012
│       │   │
│       │   └── SPAN: PostgreSQL Query (140ms)   ← the bottleneck!
│       │       span_id: pqr678, parent: jkl012
│       │
│       └── SPAN: Charge Payment (250ms)
│           │   span_id: stu901, parent: def456
│           │
│           └── SPAN: Stripe API Call (248ms)
│               span_id: vwx234, parent: stu901
│
└── Attribute: user_id=42, order_total=99.99, items_count=3
```

### Context Propagation

```python
import uuid
from opentelemetry import trace
from opentelemetry.trace import SpanKind, Status, StatusCode
from opentelemetry.propagate import inject, extract

tracer = trace.get_tracer(__name__)

# Service A: creates root span, propagates context via HTTP headers
@app.route('/api/orders', methods=['POST'])
def create_order():
    with tracer.start_as_current_span(
        "create_order",
        kind=SpanKind.SERVER
    ) as span:
        span.set_attribute("user.id", g.current_user['sub'])
        
        # Call Inventory Service with context propagation
        headers = {}
        inject(headers)  # Injects traceparent header into headers
        
        # Headers now contain:
        # traceparent: 00-{trace_id}-{span_id}-01
        # tracestate: vendor-specific state
        
        response = requests.post(
            'http://inventory-service/check',
            json={'items': request.json['items']},
            headers=headers  # Propagates trace context
        )
        
        span.set_attribute("http.status_code", response.status_code)
        return jsonify(response.json())

# Service B: extracts propagated context
@app.route('/check', methods=['POST'])
def check_inventory():
    # Extract trace context from incoming headers
    ctx = extract(request.headers)
    
    with tracer.start_as_current_span(
        "check_inventory",
        context=ctx,
        kind=SpanKind.SERVER
    ) as span:
        # This span is now a child of the span from Service A
        # The trace connects across service boundaries
        
        # Database query is automatically a child span
        inventory = db.execute("SELECT stock FROM inventory WHERE sku IN (?)", 
                               [item['sku'] for item in request.json['items']])
        
        return jsonify(inventory)
```

### W3C Trace Context Standard

```
HTTP Headers:
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
             │  │                                      │              │
             │  │                                      │              └── trace-flags (01 = sampled)
             │  │                                      └── parent-id (parent span)
             │  └── trace-id (16 bytes hex = 32 chars)
             └── version (00)

tracestate: vendor1=opaqueValue1,vendor2=opaqueValue2
```

### Sampling Strategies

```python
class SamplingStrategy:
    """
    Not every request can be traced (cost, performance).
    Sampling determines which requests are traced.
    """
    
    @staticmethod
    def head_sampling(percentage=10):
        """Decide at trace start: sample X% of requests"""
        return random.random() < (percentage / 100)
    
    @staticmethod
    def tail_sampling(percentage=100):
        """
        Decide after trace completes: keep all errors + slow ones.
        Discard fast, successful traces.
        """
        def should_keep(trace):
            if trace.has_error:
                return True  # Keep ALL errors
            if trace.duration_ms > 1000:  # > 1 second
                return True  # Keep ALL slow traces
            if random.random() < 0.01:  # 1% of fast successes
                return True
            return False
        return should_keep
    
    @staticmethod
    def adaptive_sampling():
        """
        Sample more when traffic is low, less when traffic is high.
        Maintains a target traces-per-second rate.
        """
        target_traces_per_second = 100
        current_tps = get_current_traffic_rate()
        
        if current_tps <= target_traces_per_second:
            return 1.0  # Sample everything when traffic is low
        else:
            return target_traces_per_second / current_tps
```

### OpenTelemetry Architecture

```yaml
# OpenTelemetry Collector configuration
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1000
  
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
  
  tail_sampling:
    decision_wait: 30s
    policies:
      - name: errors-policy
        type: status_code
        status_code: ERROR
      - name: latency-policy
        type: latency
        latency: { threshold_ms: 1000 }
      - name: probabilistic
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  
  otlp:
    endpoint: observability-platform:4317

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, tail_sampling]
      exporters: [jaeger, otlp]
```

### Code Instrumentation

```python
# Manual instrumentation
from opentelemetry import trace
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor

# Automatic: instrument common libraries
RequestsInstrumentor().instrument()
FlaskInstrumentor().instrument_app(app)
Psycopg2Instrumentor().instrument()

# Custom spans for business logic
@app.route('/api/orders', methods=['POST'])
def create_order():
    with tracer.start_as_current_span("create_order") as span:
        # Business logic spans
        with tracer.start_as_current_span("validate_order_items") as validate_span:
            validate_span.set_attribute("items.count", len(request.json['items']))
            validate_span.set_attribute("order.total", request.json['total'])
            validation_result = validate_order(request.json)
            validate_span.set_attribute("validation.passed", validation_result.is_valid)
        
        if not validation_result.is_valid:
            span.set_status(Status(StatusCode.ERROR, "Validation failed"))
            span.record_exception(ValidationError("Invalid order"))
            return {"error": "Validation failed"}, 400
        
        # The trace now shows exactly where time is spent
        with tracer.start_as_current_span("reserve_inventory") as inv_span:
            inventory = inventory_service.reserve(request.json['items'])
            inv_span.set_attribute("inventory.reserved", inventory.success)
        
        with tracer.start_as_current_span("process_payment") as pay_span:
            payment = payment_service.charge(request.json['total'])
            pay_span.set_attribute("payment.processor", "stripe")
            pay_span.set_attribute("payment.amount", str(request.json['total']))
```

## Layman's Explanation

### The Package Tracking Number
Distributed tracing is like package tracking for software requests:

- **Tracking Number (Trace ID)**: When you ship a package, FedEx gives you a tracking number. Every scan along the journey—pickup, sorting facility, out for delivery, delivered—is recorded against that tracking number.

- **Scan Events (Spans)**: Each scan is a span: "Arrived at Memphis sorting facility at 2:15 AM" (40ms). "Departed Memphis at 3:30 AM." "Arrived at local facility at 6:00 AM." "On delivery truck at 7:00 AM."

- **Root Span**: The initial pickup scan is the root span—it starts the trace.

- **Child Spans**: Each subsequent scan is a child of the previous scan. They form a chain (or tree if the package splits across multiple routes).

- **Troubleshooting**: If your package is 2 days late, you look at the tracking history and see it sat at the Memphis facility for 48 hours. That's your bottleneck. Without tracking, you'd just know "it took 5 days instead of 3" with no idea why.

For software: a slow API request. Jaeger traces show it spent 140ms in PostgreSQL (bottleneck) and 248ms in Stripe (external, can't control). Without tracing, you'd just know "the API takes 450ms" with no breakdown.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Sampling Rate**: Trace everything and your observability infrastructure costs more than your production infrastructure. 100% sampling of 10,000 requests/second at 100 spans/request = 1M spans/second = 86.4B spans/day = petabytes/year. Sampling is not optional—design the sampling strategy before deployment.
- **Context Propagation Mechanism**: Every communication channel must propagate trace context. HTTP: W3C traceparent headers. gRPC: metadata. Message queues: message headers (Kafka headers, SQS message attributes). Async workers: context carried in the task payload. If any link in the chain drops context, the trace breaks.
- **Span Naming Convention**: Consistent naming enables trace search and aggregation. Pattern: `{service}.{operation}` or `{HTTP_METHOD} {path}`. Inconsistent naming (e.g., "create_order" in one service, "CreateOrder" in another) prevents cross-service analysis.
- **Tail-Based Sampling**: Modern tracing platforms (Jaeger, Tempo, Honeycomb) support tail sampling—deciding to keep or discard a trace after it completes. This enables "keep all errors + all slow traces + 1% of fast successes"—the optimal balance of coverage and cost.

### Business Impact
- **MTTR Reduction**: At Uber, Jaeger reduced root cause identification time from hours to minutes. When the checkout API slows down, a Jaeger trace immediately shows whether it's the payment processor, inventory service, or database. No log spelunking needed.
- **Performance Optimization**: Traces identify the slowest spans. Optimizing a span that consumes 80% of trace duration (the Stripe API call) delivers more value than optimizing 20 spans that each consume 1%.
- **Dependency Discovery**: Traces reveal hidden dependencies. "Why is the recommendation service calling the billing service?" A trace shows the unexpected connection, enabling architectural cleanup and security review.

## On-Premises Examples

### Jaeger (Uber)
```bash
# Jaeger all-in-one for development
docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest

# Production deployment with Elasticsearch/Cassandra storage
# jaeger-collector + jaeger-query + jaeger-agent sidecars
```

```python
from jaeger_client import Config

config = Config(
    config={
        'sampler': {'type': 'const', 'param': 1},
        'logging': True,
    },
    service_name='order-service',
)
tracer = config.initialize_tracer()
```

### Zipkin (Twitter)
```java
// Spring Boot auto-instrumentation
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();
}
// Zipkin automatically traces all REST calls, database queries, and message handlers
// with spring-cloud-starter-sleuth + spring-cloud-sleuth-zipkin
```

## AWS Examples

### AWS X-Ray
```python
from aws_xray_sdk.core import xray_recorder, patch_all

# Auto-instrument supported libraries
patch_all()

@app.route('/api/orders', methods=['POST'])
def create_order():
    # Subsegment: custom business logic
    with xray_recorder.in_subsegment('validate_order') as subsegment:
        subsegment.put_annotation('order_total', request.json['total'])
        subsegment.put_metadata('items', request.json['items'])
        validate_order(request.json)
    
    # X-Ray automatically captures:
    # - HTTP requests (Flask/boto3 requests)
    # - AWS SDK calls (DynamoDB, SQS, S3, Lambda)
    # - SQL queries (via SDK interception)
    
    return jsonify(order)
```

```hcl
# X-Ray sampling rule
resource "aws_xray_sampling_rule" "orders" {
  rule_name      = "orders-api"
  priority       = 1
  reservoir_size = 10
  fixed_rate     = 0.05      # 5% of requests beyond reservoir
  host           = "*"
  http_method    = "*"
  service_name   = "order-service"
  url_path       = "/api/orders*"
  resource_arn   = "*"
}
```

### AWS Distro for OpenTelemetry
```bash
# AWS-managed OpenTelemetry Collector for EKS
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name adot \
  --addon-version v0.90.0

# Collects traces and exports to X-Ray, metrics to CloudWatch
```

## GCP Examples

### Cloud Trace
```python
from opentelemetry import trace
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

exporter = CloudTraceSpanExporter()
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(exporter)
)

# Traces appear in Cloud Console → Trace Explorer
# Automatic integration with Cloud Run, GKE, App Engine
```

## Azure Examples

### Application Insights
```python
from azure.monitor.opentelemetry import configure_azure_monitor

configure_azure_monitor(
    connection_string="InstrumentationKey=..."
)

# All Flask requests, HTTP calls, and DB queries are automatically traced

@app.route('/api/orders', methods=['POST'])
def create_order():
    # Application Insights automatically creates:
    # - Request telemetry
    # - Dependency telemetry (calls to other services)
    # - Exception telemetry if errors occur
    # - Trace telemetry from log messages
    return order
```

### Application Map
```bash
# Azure automatically builds an application map from traces
# Visualizes: service dependencies, call rates, failure rates
# Identifies bottlenecks and unexpected dependencies
```

## Summary

| Tracing Platform | Best For | Storage | Sampling |
|-----------------|----------|---------|----------|
| Jaeger | Kubernetes, OpenTelemetry native | Elasticsearch/Cassandra | Head + Tail |
| Zipkin | Spring Boot/Java ecosystem | MySQL/Elasticsearch | Head |
| AWS X-Ray | AWS-native applications | Managed (30 days) | Fixed rate + reservoir |
| Cloud Trace | GCP-native applications | Managed (30 days) | Configurable |
| Application Insights | Azure + .NET ecosystem | Managed (90 days) | Adaptive |
| Grafana Tempo | OpenTelemetry native, S3-backed | S3/GCS/Minio | Tail (traces are cheap) |

Distributed tracing transforms debugging from "which of our 47 microservices is slow?" to "the /api/orders endpoint spends 250ms in Stripe and 140ms in PostgreSQL—here's the exact trace ID to investigate." In a microservices architecture, distributed tracing is as essential as logging. The architect must ensure every service propagates trace context, every communication channel supports W3C traceparent headers, and the sampling strategy balances coverage against infrastructure cost.
