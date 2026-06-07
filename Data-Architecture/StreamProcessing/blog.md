# Stream Processing

## Introduction
Stream Processing is the paradigm of continuously processing data in motion—as it arrives—rather than collecting it first and processing it in batches. The demands of real-time business drove this shift: fraud detection must block a transaction in milliseconds, not hours. Ride-sharing must match drivers to riders in real-time. Stock trading algorithms must react to price changes instantly. Batch processing, which collects data for hours then processes it, cannot meet these latency requirements.

The stream processing ecosystem has matured from Apache Storm (2011) through Apache Flink (2014) and Kafka Streams (2016) to cloud-managed services (Kinesis Data Analytics, Dataflow, Azure Stream Analytics). The fundamental challenge—the one every stream processor must solve—is handling out-of-order events, exactly-once processing, and state management in the face of failures.

## Definition

**Stream Processing** is a data processing paradigm that continuously ingests, processes, and analyzes data streams in near real-time, producing results with latency measured in milliseconds to seconds rather than hours to days.

**Key characteristics**:
- **Continuous**: Processing never stops; data flows continuously through operators
- **Event-at-a-time or Micro-batch**: Process individual events or small windows of events
- **Stateful**: Maintain state across events (counts, aggregations, joins, session windows)
- **Low latency**: Milliseconds to seconds from event ingestion to result

## Concept Explanation

### Event Time vs Processing Time vs Ingestion Time

```
Event Time:      When the event actually happened (user clicked at 10:00:00)
Processing Time: When the system processes the event (Flink processes at 10:02:30)
Ingestion Time:  When the event entered the system (Kafka received at 10:02:15)

The difference matters because:
- Events can arrive late (mobile device offline → buffered events → sent when reconnected)
- Events can arrive out of order (network delays cause different paths)
- Processing can be delayed (backpressure, failures)
```

```python
# Event time semantics: window based on when the event happened, not when processed
# 10:00:00 click processed at 10:15 is still in the 10:00-10:05 window

class WindowingStrategy:
    """Handle late-arriving data gracefully"""
    
    def assign_windows(self, event):
        # Window by event time, not processing time
        return event.event_time.timestamp() // (5 * 60)  # 5-minute windows
    
    def handle_late_event(self, event, watermark):
        if event.event_time < watermark:
            # Event is late (older than current watermark)
            # Option 1: Drop it (if late tolerance exceeded)
            # Option 2: Update previously emitted results
            # Option 3: Emit to side output for special handling
            pass
```

### Watermarks

```
Watermarks track event time progress. They say:
"All events with timestamp < T have (probably) arrived."

[Events flowing in with event times]
... 10:00:03, 10:00:15, 10:00:07, 10:00:12, 10:00:20, 10:00:11, 10:00:01(late!)

Watermark at 10:00:15 → "All events before 10:00:15 have been seen"
But 10:00:01 arrives AFTER watermark → it's late!

Watermarks are heuristics:
- Bounded out-of-orderness: watermark = max_event_time - allowed_lateness
- Periodic: update watermark every N seconds based on max observed event time
```

### Processing Guarantees

```
At-Most-Once: Events may be lost, never duplicated
  → Fastest, acceptable for metrics where occasional loss is OK

At-Least-Once: Events never lost, but may be duplicated
  → Most common; requires idempotent downstream operations
  → Kafka default, Kinesis default

Exactly-Once: Events processed exactly once, no loss, no duplication
  → Hardest to implement; requires distributed checkpointing
  → Flink checkpoint + Kafka transactions, Kinesis Data Analytics
  → Comes with latency and throughput cost
```

### Stream Processing Patterns

#### Windowed Aggregation
```python
# Tumbling window: fixed-size, non-overlapping windows
# Events from [10:00-10:05] in one window, [10:05-10:10] in next

# Flink SQL equivalent
"""
SELECT 
    TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_start,
    product_id,
    COUNT(*) as order_count,
    SUM(total) as revenue
FROM orders
GROUP BY TUMBLE(event_time, INTERVAL '5' MINUTE), product_id;
"""

# Sliding window: overlapping windows
# Window [10:00-10:05], [10:01-10:06], [10:02-10:07] ...
"""
SELECT 
    HOP_START(event_time, INTERVAL '1' MINUTE, INTERVAL '5' MINUTE) as window_start,
    COUNT(*) as order_count
FROM orders
GROUP BY HOP(event_time, INTERVAL '1' MINUTE, INTERVAL '5' MINUTE);
"""

# Session window: dynamic windows based on activity gaps
# User session = events until 30 minutes of inactivity
"""
SELECT 
    SESSION_START(event_time, INTERVAL '30' MINUTE) as session_start,
    user_id,
    COUNT(*) as events_in_session
FROM clicks
GROUP BY SESSION(event_time, INTERVAL '30' MINUTE), user_id;
"""
```

#### Stream-Table Join (Enrichment)
```python
# Join a stream of orders with a slowly-changing customer table
# Enrich each order with customer segment at time of order

# Kafka Streams
order_stream = builder.stream("orders")
customer_table = builder.table("customers")

enriched_orders = order_stream \
    .join(customer_table,
          lambda order, customer: EnrichedOrder(order, customer.segment),
          JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofSeconds(0)))
    .to("enriched-orders")
```

#### CEP (Complex Event Processing)
```python
# Pattern: "Order placed → Payment NOT charged within 5 minutes → Alert"

# Flink CEP
Pattern<OrderEvent, ?> pattern = Pattern.<OrderEvent>begin("order")
    .where(event -> event.type == "ORDER_PLACED")
    .next("payment")
    .where(event -> event.type == "PAYMENT_CHARGED")
    .within(Time.minutes(5));

// If pattern NOT matched (payment not received in 5 min):
PatternStream<OrderEvent> patternStream = CEP.pattern(stream, pattern);
DataStream<Alert> alerts = patternStream.select(
    pattern -> new Alert("Payment missing for order " + pattern.get("order").orderId)
);
```

### Exactly-Once with Checkpointing

```python
# Apache Flink: distributed snapshots for exactly-once
# Flink periodically checkpoints:
# 1. Source offsets (Kafka partition offsets)
# 2. Operator state (window contents, aggregation values)
# 3. Sink state (transaction IDs for idempotent writes)
# All part of a consistent, distributed snapshot

# On failure:
# 1. Rewind all sources to last checkpoint offset
# 2. Restore all operator state from checkpoint
# 3. Replay events from checkpoint → recover exact state
# 4. Idempotent sink writes prevent duplicates
```

## Layman's Explanation

### The Sushi Conveyor Belt
Stream processing is like a sushi conveyor belt restaurant:

- **Event Stream**: Plates of sushi (events) continuously moving on the belt
- **Stream Processor**: The chef preparing sushi and placing it on the belt (Kafka producer) and customers picking up plates (Kafka consumer/processor)
- **Windowing**: The manager counts how many salmon rolls were ordered every 5 minutes (tumbling window)
- **Watermarks**: The clock on the wall. If a chef places a plate with a "10:00" timestamp on the belt at 10:15, the system says "this event is late—it was supposedly made 15 minutes ago"
- **State**: The manager's notepad tracking "how many of each type ordered today" (counter)
- **Exactly-Once**: Even if the belt stops and restarts (failure), every plate is counted exactly once, no double-counting, no missed plates

### Why Not Just Databases?
You could write every click to a database and query it. But:
- 1M clicks/second × writing to PostgreSQL = database melts at 5K writes/second
- Stream processing handles 1M events/second by distributing across 100 workers
- Results appear in seconds, not "wait for the batch job that runs at midnight"

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Stream vs Batch**: If results are needed within seconds-minutes, stream processing is required. If results are acceptable within hours, batch is simpler and cheaper. Many architectures do both: streaming for real-time dashboards and alerts; batch for end-of-day reconciliation and model training.
- **At-Least-Once vs Exactly-Once**: Exactly-once processing is 2-3x more expensive in terms of throughput and adds operational complexity. Most applications are fine with at-least-once if: (a) downstream systems are idempotent, (b) occasional duplicates are acceptable (analytics dashboards with 0.01% double-counting). Financial reconciliation requires exactly-once.
- **State Management**: Stateful stream processing requires checkpointing and state backend storage (RocksDB, remote storage). State grows over time—a join with customer data covering 5 years of history needs efficient state management (TTL, compaction).
- **Late Data Handling**: Every stream processing system must define a policy for late data. Options: drop (simple), emit updates (complex, requires updatable sinks), or route to dead letter stream (operationally visible).

### Business Impact
- **Real-Time Fraud Detection**: Batch fraud detection catches fraud tomorrow. Streaming catches it during the transaction, saving millions. Credit card companies process thousands of events/second with sub-100ms latency for fraud scoring.
- **Operational Visibility**: Streaming dashboards show exactly what's happening NOW—not what happened in yesterday's batch. This enables immediate response to incidents, marketing campaign performance, and user behavior changes.
- **Personalization**: Real-time recommendations (you viewed X → immediately show related Y) depend on stream processing of clickstream data with sub-second latency.

## On-Premises Examples

### Kafka Streams
```java
StreamsBuilder builder = new StreamsBuilder();
KStream<String, Order> orders = builder.stream("orders");

// Filter high-value orders
KStream<String, Order> highValueOrders = orders
    .filter((key, order) -> order.getTotal() > 1000);

// Count orders per product (stateful aggregation)
KTable<String, Long> ordersPerProduct = orders
    .groupBy((key, order) -> order.getProductId())
    .count();

// Join stream with table
KStream<String, EnrichedOrder> enriched = orders
    .join(ordersPerProduct,
        (order, count) -> new EnrichedOrder(order, count));

enriched.to("enriched-orders");
```

### Apache Flink (Advanced Stream Processing)
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(60000);  // Exactly-once every 60s

DataStream<Order> orders = env
    .addSource(new FlinkKafkaConsumer<>("orders", new OrderSchema(), properties))
    .assignTimestampsAndWatermarks(
        WatermarkStrategy.<Order>forBoundedOutOfOrderness(Duration.ofSeconds(30))
            .withTimestampAssigner((order, timestamp) -> order.getEventTime())
    );

// Windowed aggregation
DataStream<DailyRevenue> revenue = orders
    .keyBy(Order::getProductCategory)
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .aggregate(new RevenueAggregator());

revenue.addSink(new FlinkKafkaProducer<>("daily-revenue", new RevenueSchema(), properties));
```

## AWS Examples

### Kinesis Data Analytics (Flink Managed)
```python
# AWS Managed Flink application
# Kinesis Data Stream → Flink → Kinesis Data Stream / S3

# No cluster management; AWS handles Flink cluster, checkpointing, scaling
```

```hcl
resource "aws_kinesisanalyticsv2_application" "stream_processor" {
  name                   = "order-stream-processor"
  runtime_environment    = "FLINK-1_18"
  service_execution_role = aws_iam_role.flink.arn

  application_configuration {
    application_code_configuration {
      code_content {
        s3_content_location {
          bucket_arn = aws_s3_bucket.code.arn
          file_key   = "flink-app.jar"
        }
      }
      code_content_type = "ZIPFILE"
    }

    environment_properties {
      property_group {
        property_group_id = "kinesis"
        property_map = {
          "input_stream"  = aws_kinesis_stream.orders.name
          "output_stream" = aws_kinesis_stream.enriched.name
        }
      }
    }
  }
}
```

### Kinesis Data Streams + Lambda (Serverless Stream)
```python
# Lambda processes each batch of records
def lambda_handler(event, context):
    for record in event['Records']:
        # Kinesis record is base64 encoded
        payload = base64.b64decode(record['kinesis']['data'])
        order = json.loads(payload)
        
        # Process: enrich, filter, transform
        enriched = enrich_order(order)
        
        # Write to DynamoDB, S3, or another Kinesis stream
        dynamodb.put_item(TableName='EnrichedOrders', Item=enriched)
```

## GCP Examples

### Dataflow (Apache Beam Managed)
```python
import apache_beam as beam
from apache_beam.transforms.window import FixedWindows

with beam.Pipeline() as pipeline:
    orders = (
        pipeline
        | 'Read from PubSub' >> beam.io.ReadFromPubSub(
            subscription='projects/my-project/subscriptions/orders-sub'
        )
        | 'Parse JSON' >> beam.Map(json.loads)
        | 'Window' >> beam.WindowInto(FixedWindows(300))  # 5-minute windows
    )
    
    # Revenue by product
    revenue = (
        orders
        | 'Key by product' >> beam.Map(lambda o: (o['product_category'], o['total']))
        | 'Sum revenue' >> beam.CombinePerKey(sum)
        | 'Format' >> beam.Map(lambda r: f"{r[0]},{r[1]}")
    )
    
    # Write to BigQuery in streaming mode
    orders | 'Write to BQ' >> beam.io.WriteToBigQuery(
        'analytics.orders',
        write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND
    )
```

## Azure Examples

### Azure Stream Analytics
```sql
-- SQL-like language for stream processing
-- Input: Event Hub / IoT Hub
-- Output: Power BI, SQL Database, Blob Storage

SELECT 
    System.Timestamp AS window_end,
    product_category,
    COUNT(*) AS order_count,
    SUM(total) AS revenue
INTO powerbi_output
FROM orders_input TIMESTAMP BY event_time
GROUP BY 
    TumblingWindow(minute, 5),
    product_category;
```

### Azure Databricks (Spark Structured Streaming)
```python
# Structured Streaming: treat stream as unbounded table
orders_stream = (
    spark.readStream
        .format("delta")
        .load("/data-lake/bronze/orders")
)

revenue = (
    orders_stream
        .withWatermark("event_time", "30 seconds")
        .groupBy(
            window("event_time", "5 minutes"),
            "product_category"
        )
        .agg(
            count("*").alias("order_count"),
            sum("total").alias("revenue")
        )
)

revenue.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "/checkpoints/revenue") \
    .start("/data-lake/gold/revenue")
```

## Summary Decision Matrix

| Stream Processor | Best For | State | Exactly-Once | Complexity |
|-----------------|----------|-------|-------------|------------|
| Kafka Streams | Simple aggregations, joins within Kafka | Local RocksDB | Yes | Low-Medium |
| Apache Flink | Complex event processing, CEP | RocksDB/Remote | Yes | High |
| Spark Streaming | Unified batch+stream, Delta Lake integration | In-memory/Checkpoint | Yes (with Delta) | Medium |
| Kinesis Analytics | AWS-managed Flink, serverless | Managed | Yes | Low (managed) |
| Dataflow | GCP-managed, Apache Beam | Managed | Yes | Medium |
| Stream Analytics | Azure-managed, SQL-based | Managed | At-least-once | Low (managed) |
| Lambda + Kinesis | Simple, serverless, stateless | External (DynamoDB) | No (at-least-once) | Low |

Stream processing is no longer a niche technology—it's a mainstream architectural requirement for any application needing real-time insights or sub-second reactions. The architect must decide: what results need sub-second latency (stream processing), what can wait minutes to hours (batch), and how to build a unified pipeline that handles both without duplicating logic. The Lambda architecture (separate stream + batch layers) is giving way to the Kappa architecture (stream-only with replay capability), but both remain valid patterns depending on the specific latency and correctness requirements.
