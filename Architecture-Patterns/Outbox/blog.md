# Outbox Pattern

The Outbox Pattern solves the dual-write problem: when a service must atomically update its database AND publish an event, but the database and message broker are separate systems with no shared transaction. Without the Outbox, you risk: database committed but event not published (lost event), or event published but database rolled back (phantom event).

The solution: write the event to an OUTBOX table within the same database transaction as the business data. A separate process (CDC or polling) reads the outbox table and publishes events to the message broker reliably.

## The Problem

```python
# BROKEN: No atomicity between DB and message broker
def create_order(order_data):
    db.execute("INSERT INTO orders ...")  # COMMIT
    kafka.send("OrderPlaced", event)      # This can FAIL → lost event
    # OR
    kafka.send("OrderPlaced", event)      # This succeeds
    db.execute("INSERT INTO orders ...")  # This ROLLS BACK → phantom event
```

## The Solution

```sql
-- Outbox table in the same database
CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(200) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    published_at TIMESTAMP NULL
);
```

```python
def create_order(order_data):
    # SINGLE database transaction
    with db.transaction():
        order_id = db.execute("INSERT INTO orders (customer_id, total) VALUES (?, ?)",
                              [order_data['customer_id'], order_data['total']])
        
        # Event in same transaction
        db.execute("""
            INSERT INTO outbox (id, aggregate_type, aggregate_id, event_type, payload)
            VALUES (?, 'Order', ?, 'OrderPlaced', ?)
        """, [str(uuid.uuid4()), order_id, json.dumps({
            'order_id': order_id,
            'customer_id': order_data['customer_id'],
            'total': order_data['total']
        })])
    
    # Transaction committed → both order AND event are persisted atomically
```

### Outbox Publisher (CDC Approach)

```python
# Debezium CDC: Capture outbox table changes and publish to Kafka
# Debezium connector monitors PostgreSQL WAL → emits to Kafka topic

# outbox table INSERT → Debezium detects change → publishes to Kafka
# Kafka topic: outbox.event.OrderPlaced

# Consumer receives from Kafka with exactly-once semantics
```

### Outbox Publisher (Polling Approach)

```python
import threading
import json

class OutboxPublisher:
    def __init__(self, db, kafka_producer, poll_interval=0.1):
        self.db = db
        self.kafka = kafka_producer
        self.poll_interval = poll_interval
    
    def start(self):
        thread = threading.Thread(target=self._poll, daemon=True)
        thread.start()
    
    def _poll(self):
        while True:
            # Read unpublished events (with row lock to prevent duplicate publishing)
            events = self.db.execute("""
                SELECT id, aggregate_type, event_type, payload
                FROM outbox
                WHERE published_at IS NULL
                ORDER BY created_at
                LIMIT 100
                FOR UPDATE SKIP LOCKED
            """).fetchall()
            
            for event in events:
                try:
                    # Publish to message broker
                    topic = f"{event['aggregate_type']}.{event['event_type']}"
                    self.kafka.send(topic, key=event['aggregate_id'], value=event['payload'])
                    
                    # Mark as published
                    self.db.execute(
                        "UPDATE outbox SET published_at = NOW() WHERE id = ?",
                        [event['id']]
                    )
                except Exception as e:
                    # Will retry on next poll
                    pass
            
            time.sleep(self.poll_interval)
```

### Cleanup

```python
# Periodically delete old published events
db.execute("""
    DELETE FROM outbox
    WHERE published_at IS NOT NULL
    AND published_at < NOW() - INTERVAL '7 days'
""")
```

## Debezium Outbox Event Router

```json
{
  "transforms": "outbox",
  "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
  "transforms.outbox.route.topic.replacement": "${routedByValue}",
  "transforms.outbox.table.field.event.id": "id",
  "transforms.outbox.table.field.event.key": "aggregate_id",
  "transforms.outbox.table.field.event.payload": "payload",
  "transforms.outbox.route.by.field": "event_type"
}
```

## When to Use

- Any service that must atomically persist data AND publish events
- Microservices with their own database needing reliable event publishing
- Event sourcing implementations where event publication must be guaranteed

The Outbox Pattern provides the atomicity guarantee that distributed transactions cannot. It's the standard pattern for reliable event publishing in microservices.
