# Event-Driven Architecture

## Introduction
Event-Driven Architecture (EDA) is an architectural pattern where systems communicate by producing and consuming events—immutable facts about things that happened. Unlike request-driven architectures where services call each other synchronously, event-driven services react to events asynchronously. This decoupling is transformative: the order service doesn't call the notification service; it simply emits an "OrderPlaced" event, and whatever services care about orders (notifications, inventory, analytics, fraud detection) react independently.

The paradigm shift from "tell" to "inform" is subtle but profound. The order service no longer needs to know which downstream services exist, what APIs they expose, or whether they're available. This enables independent evolution, resilience (downstream failures don't cascade), and extensibility (new services can subscribe to existing events without touching producers). Combined with Event Sourcing and CQRS, EDA provides the foundation for systems that are both scalable and auditable.

## Definition

**Event-Driven Architecture** is a software architecture pattern where system components communicate through the production, detection, and consumption of events. An **event** is an immutable record of something that happened (past tense): `OrderPlaced`, `PaymentCharged`, `ShipmentDelivered`.

**Key concepts**:
- **Event Producer**: The component that detects a state change and publishes an event
- **Event Consumer**: The component that subscribes to events and takes action
- **Event Broker**: The infrastructure that routes events from producers to consumers (Kafka, EventBridge, Pub/Sub)
- **Event Stream**: An ordered, immutable, append-only log of events

## Concept Explanation

### Events vs Commands vs Messages

```
COMMAND: "Do something" (imperative, directed)
  { "type": "ChargePayment", "order_id": "123", "amount": 99.99 }
  → Sent TO a specific service
  → Can be rejected ("insufficient funds")
  → Service coupling: sender must know receiver

EVENT: "Something happened" (declarative, broadcast)
  { "type": "PaymentCharged", "order_id": "123", "amount": 99.99, "timestamp": "..." }
  → Published TO a topic/channel
  → Cannot be rejected (it already happened)
  → Service decoupling: sender doesn't know receivers

MESSAGE: Generic term; both commands and events are messages
```

### Choreography vs Orchestration

```
CHOREOGRAPHY (Event-Driven):
  Each service reacts to events independently.
  No central coordinator.
  
  Order Service:   emits OrderPlaced
  Inventory:       reacts to OrderPlaced → reserves stock → emits InventoryReserved
  Payment:         reacts to InventoryReserved → charges → emits PaymentCharged
  Shipping:        reacts to PaymentCharged → creates label → emits ShipmentCreated
  Notification:    reacts to ShipmentCreated → emails customer

  Like dancers who know their steps without a conductor.


ORCHESTRATION (Central Coordinator):
  A central Saga orchestrator directs each step.
  
  SagaOrchestrator:
    order_id = OrderService.create_order()
    inventory_result = InventoryService.reserve(order_id)
    payment_result = PaymentService.charge(order_id)
    shipment_id = ShippingService.create(order_id)
  
  Like a conductor directing an orchestra.
```

### Event Sourcing

```python
class EventSourcedOrder:
    """
    Instead of storing current state, store the sequence of events.
    Current state is derived by replaying events.
    """
    
    def __init__(self, order_id):
        self.order_id = order_id
        self.events = []
        self.state = self._initial_state()
    
    def apply_event(self, event):
        """Append event and update state"""
        self.events.append(event)
        
        if event['type'] == 'OrderPlaced':
            self.state['status'] = 'PLACED'
            self.state['items'] = event['items']
            self.state['customer_id'] = event['customer_id']
        
        elif event['type'] == 'PaymentCharged':
            self.state['status'] = 'PAID'
            self.state['payment_amount'] = event['amount']
            self.state['payment_method'] = event['method']
        
        elif event['type'] == 'OrderShipped':
            self.state['status'] = 'SHIPPED'
            self.state['tracking_number'] = event['tracking']
        
        elif event['type'] == 'OrderDelivered':
            self.state['status'] = 'DELIVERED'
            self.state['delivered_at'] = event['timestamp']
    
    def replay_events(self, events):
        """Rebuild state from event history"""
        self.state = self._initial_state()
        for event in events:
            self.apply_event(event)
        return self.state
    
    def get_current_state(self):
        return self.state

# Event store:
# order-123:
#   [OrderPlaced, PaymentCharged, OrderShipped, OrderDelivered]
# 
# Current state derived: { status: 'DELIVERED', items: [...], ... }
# 
# Benefits:
# - Complete audit trail (every state change is recorded)
# - Time travel (query state at any point in time)
# - Debugging (replay events to reproduce bugs)
# - No ORM mismatch (events ARE the persistence model)
```

### CQRS (Command Query Responsibility Segregation)

```
CQRS separates reads and writes into different models:

COMMAND SIDE (Writes):
  POST /api/orders { items, customer_id }
  → Validate command
  → Generate event: OrderPlaced
  → Append to event store
  → Return 202 Accepted (not 200—processing is async)

QUERY SIDE (Reads):
  GET /api/orders/123
  → Read from query-optimized database (denormalized)
  → Fast, simple queries
  → Eventually consistent with command side

Architecture:
  [Command API] → [Event Store] → [Projections] → [Query API]
       │               │               │              │
       │          order-123:      Build read     GET /orders/123
       │          [OrderPlaced]   model from     → {"status":"PLACED",...}
       │          [PaymentCharged] events
  POST /orders
  → append event
```

```python
class OrderProjection:
    """
    Build a read-optimized view from events.
    Each projection serves a specific use case.
    """
    
    def __init__(self, db):
        self.db = db  # Could be PostgreSQL, DynamoDB, Elasticsearch
    
    def on_order_placed(self, event):
        self.db.execute("""
            INSERT INTO orders_view (order_id, customer_id, items, status, created_at)
            VALUES (?, ?, ?, 'PLACED', ?)
        """, (event.order_id, event.customer_id, json.dumps(event.items), event.timestamp))
    
    def on_payment_charged(self, event):
        self.db.execute("""
            UPDATE orders_view 
            SET status = 'PAID', payment_amount = ?, updated_at = ?
            WHERE order_id = ?
        """, (event.amount, event.timestamp, event.order_id))
    
    def on_order_shipped(self, event):
        self.db.execute("""
            UPDATE orders_view 
            SET status = 'SHIPPED', tracking = ?, updated_at = ?
            WHERE order_id = ?
        """, (event.tracking, event.timestamp, event.order_id))
```

### Event Bus Architecture

```yaml
# EventBridge pattern: central event bus with routing rules
EventBus:
  Name: "order-events"
  Rules:
    - name: "high-value-orders"
      pattern:
        detail-type: ["OrderPlaced"]
        detail:
          total: [{numeric: [">", 1000]}]
      targets:
        - fraud-detection-service
        - premium-support-queue
    
    - name: "all-orders-analytics"
      pattern:
        detail-type: ["OrderPlaced", "OrderShipped", "OrderCancelled"]
      targets:
        - analytics-stream (Kinesis)
        - data-lake (Firehose → S3)
    
    - name: "customer-notifications"
      pattern:
        detail-type: ["OrderShipped", "OrderDelivered"]
      targets:
        - notification-service
```

## Layman's Explanation

### The Newspaper Analogy
**Request-Driven (Synchronous)**: You need to tell your neighbors about a block party. You walk to each neighbor's house, knock on the door, and tell them individually. If one neighbor isn't home, you must either wait or come back later. You must know every neighbor's address. This is microservices calling each other via REST APIs.

**Event-Driven (Asynchronous)**: You publish a notice in the community newspaper (event broker). Every neighbor who subscribes to the newspaper (event consumer) sees the announcement. You don't need to know who's reading. New neighbors who just moved in also see it. If one neighbor's paper is delayed, they still get it eventually. You published once and moved on.

### The Receipt Analogy
When you buy something at a store, you get a receipt (event). The receipt is a FACT—it happened. You can't "reject" a receipt. The receipt serves many purposes: you use it for returns (consumer 1), your accountant uses it for taxes (consumer 2), the store uses a copy for inventory tracking (consumer 3). The cashier didn't need to know about any of these downstream uses—they just generated the receipt.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Event Schema Evolution**: Events are immutable—once published, they exist forever in the event stream. Schema evolution must be backward-compatible (new fields are optional) and forward-compatible (old consumers ignore new fields). Avro with Schema Registry (Confluent), Protobuf with Buf, or JSON with strict versioning are standard approaches.
- **Ordering Guarantees**: Global ordering is the enemy of scalability. Partition ordering (events for the same entity go to the same partition and are ordered within that partition) is the practical sweet spot. Kafka: partition by `order_id`. SQS FIFO: message group ID. Design consumers to tolerate out-of-order events across partitions.
- **Idempotency**: Event consumers MUST be idempotent because at-least-once delivery means events may be processed multiple times. Store `event_id` in a processed-events table; check before processing. Never assume exactly-once delivery without explicit deduplication.
- **Eventual Consistency**: Event-driven systems are eventually consistent. After OrderPlaced is published, the query-side projection may not be updated for milliseconds to seconds. Compensate with read-your-writes patterns (route reads to the command side for recently modified entities).

### Business Impact
- **Independent Team Velocity**: The notification team can add SMS notifications to the OrderShipped event without coordinating with the order team. They just subscribe to an existing event. This is the organizational superpower of event-driven architecture.
- **Complete Audit Trail**: Event sourcing provides a complete, immutable history of every state change. This satisfies auditors directly—"show me every change to order #123" is answered by replaying its event stream.
- **New Capabilities from Old Data**: An event stream from last year can be replayed through new analytics models to derive insights that weren't envisioned when the data was collected. This is impossible with state-based persistence where old state is overwritten.

## On-Premises Examples

### Kafka + Avro + Schema Registry
```bash
# Confluent Schema Registry
schema-registry-start /etc/schema-registry/schema-registry.properties

# Register Avro schema
curl -X POST http://schema-registry:8081/subjects/orders-value/versions \
  -H "Content-Type: application/json" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"OrderPlaced\",\"fields\":[...]}"
  }'

# Producer with schema validation
# Consumer with schema compatibility checking
```

```java
// Kafka producer with Avro serialization
ProducerRecord<String, OrderPlaced> record = new ProducerRecord<>(
    "orders",
    order.getOrderId(),
    OrderPlaced.newBuilder()
        .setOrderId(order.getOrderId())
        .setCustomerId(order.getCustomerId())
        .setTotal(order.getTotal())
        .setTimestamp(Instant.now())
        .build()
);
producer.send(record);
```

### Debezium CDC (Database → Event Stream)
```yaml
# Debezium captures database changes as events
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
spec:
  class: io.debezium.connector.postgresql.PostgresConnector
  config:
    database.hostname: orders-db
    database.port: 5432
    plugin.name: pgoutput
    table.include.list: public.orders
    transforms: outbox
    # Changes to orders table → Kafka topic "orders-db.public.orders"
```

## AWS Examples

### Amazon EventBridge
```python
import boto3

eventbridge = boto3.client('events')

# Publish event
eventbridge.put_events(Entries=[{
    'Source': 'com.myapp.orders',
    'DetailType': 'OrderPlaced',
    'Detail': json.dumps({
        'order_id': 'ORD-123',
        'customer_id': 'CUST-789',
        'total': 99.99,
        'items_count': 3
    }),
    'EventBusName': 'default'
}])

# Rule: route to Lambda, SQS, SNS, Step Functions, etc.
# Schema registry: discover and validate event schemas
```

### EventBridge Pipes (Point-to-Point Event Flow)
```hcl
resource "aws_pipes_pipe" "order_enrichment" {
  name     = "order-enrichment"
  role_arn = aws_iam_role.pipe.arn

  source = aws_sqs_queue.orders.arn
  target = aws_sqs_queue.enriched_orders.arn

  enrichment = aws_lambda_function.enrich_order.arn

  source_parameters {
    sqs_queue_parameters {
      batch_size = 10
    }
  }
}
```

### Step Functions (Saga Orchestrator)
```json
{
  "Comment": "Order processing saga",
  "StartAt": "CreateOrder",
  "States": {
    "CreateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "Next": "ReserveInventory"
    },
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "Catch": [{"ErrorEquals": ["InsufficientInventory"], "Next": "CancelOrder"}],
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "Catch": [{"ErrorEquals": ["PaymentFailed"], "Next": "ReleaseInventory"}],
      "End": true
    },
    "ReleaseInventory": {
      "Type": "Task", "Resource": "...", "Next": "CancelOrder"
    },
    "CancelOrder": {
      "Type": "Task", "Resource": "...", "End": true
    }
  }
}
```

## GCP Examples

### Eventarc
```bash
# Trigger Cloud Run from events
gcloud eventarc triggers create order-placed-trigger \
  --location=us-central1 \
  --destination-run-service=order-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.audit.log.v1.written" \
  --event-filters="serviceName=order-service" \
  --event-filters="methodName=google.cloud.audit.log.v1.CreateOrder"

# 90+ event sources: Cloud Storage, BigQuery, Firestore, Pub/Sub, 3rd party
```

### Pub/Sub (Event Messaging)
```python
from google.cloud import pubsub_v1

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path('my-project', 'order-events')

# Publish with ordering key (same key → same partition, ordered)
publisher.publish(
    topic_path,
    data=json.dumps(event).encode(),
    ordering_key=event['order_id']  # All events for same order are ordered
)
```

## Azure Examples

### Event Grid
```json
{
  "topic": "/subscriptions/.../resourceGroups/myRG/providers/Microsoft.EventGrid/topics/order-events",
  "subject": "orders/ORD-123",
  "eventType": "OrderPlaced",
  "data": {
    "orderId": "ORD-123",
    "customerId": "CUST-789",
    "total": 99.99
  },
  "dataVersion": "1.0",
  "eventTime": "2024-06-15T10:30:00Z"
}
```

### Durable Functions (Stateful Orchestration)
```csharp
[FunctionName("OrderSaga")]
public static async Task RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var order = context.GetInput<Order>();
    
    await context.CallActivityAsync("CreateOrder", order);
    
    try {
        await context.CallActivityAsync("ReserveInventory", order);
    } catch {
        await context.CallActivityAsync("CancelOrder", order);
        throw;
    }
    
    await context.CallActivityAsync("ChargePayment", order);
    await context.CallActivityAsync("ShipOrder", order);
}
```

## Summary

| EDA Pattern | When to Use | Complexity | Consistency |
|------------|-------------|-----------|-------------|
| Simple Pub/Sub | Decoupled notifications, analytics | Low | Eventual |
| Event Sourcing | Audit trail, time travel, full history | High | Strong (event log) |
| CQRS | Different read/write patterns, scaling reads | High | Eventual (between sides) |
| Choreography | Independent team ownership | Medium | Eventual |
| Orchestration (Saga) | Complex multi-step transactions | Medium | Eventual (with compensating) |

Event-driven architecture is not all-or-nothing. Start with simple pub/sub for cross-service notifications. Progress to CQRS for services with radically different read/write patterns. Adopt event sourcing where audit trail and temporal queries are required. The key insight is that events decouple services in time (producers and consumers don't need to be available simultaneously), space (they don't need to know each other's addresses), and synchronization (they don't block waiting for each other). This decoupling is the architectural foundation for scalable, resilient, and independently deployable services.
