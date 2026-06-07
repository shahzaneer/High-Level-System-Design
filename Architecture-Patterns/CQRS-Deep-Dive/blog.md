# CQRS Deep Dive

CQRS separates read and write operations into different models, optimized for their respective workloads. Writes use a normalized, transactional model. Reads use denormalized, query-optimized models. The write model publishes events after state changes; read models subscribe to events and update their projections.

CQRS is not event sourcing (though they're often used together). Event sourcing means storing state as a sequence of events. CQRS means separating read and write models. You can do CQRS without event sourcing (just maintain separate read tables updated synchronously or asynchronously).

## The Architecture

```
                  WRITE SIDE                          READ SIDE
              ┌─────────────────┐              ┌─────────────────┐
  Command →   │ Write API        │              │ Query API       │  ← Query
              │  ┌─────────────┐ │              │  ┌────────────┐ │
              │  │ Domain Model│ │   Events     │  │Read Models │ │
              │  │ (Normalized)│ │──────────────→│  │(Denorm'd)  │ │
              │  └─────────────┘ │              │  └────────────┘ │
              │  ┌─────────────┐ │              │                 │
              │  │ Event Store │ │              │ PostgreSQL      │
              │  │ (Write DB)  │ │              │ Elasticsearch   │
              │  └─────────────┘ │              │ DynamoDB        │
              └─────────────────┘              └─────────────────┘
```

## Write Side

```python
class OrderCommandHandler:
    def handle_create_order(self, command: CreateOrderCommand):
        # Validate business rules
        customer = customer_repo.get(command.customer_id)
        if not customer.is_active:
            raise DomainException("Inactive customer cannot place orders")
        
        # Create domain object
        order = Order.create(
            customer_id=command.customer_id,
            items=command.items
        )
        
        # Save to event store (or ORM with normalized tables)
        event_store.save(order.pending_events)
        
        # Publish events for read side
        for event in order.pending_events:
            event_bus.publish(event)
        
        return order.id
```

## Read Side (Projections)

```python
class OrderProjection:
    """Builds and maintains read-optimized views"""
    
    def __init__(self, read_db):
        self.db = read_db  # Denormalized PostgreSQL, or Elasticsearch
    
    def on_order_placed(self, event: OrderPlaced):
        self.db.execute("""
            INSERT INTO orders_view (order_id, customer_id, customer_name,
                                      items, total, status, created_at)
            VALUES (?, ?, ?, ?, ?, 'PLACED', ?)
        """, [
            event.order_id, event.customer_id, event.customer_name,
            json.dumps(event.items), event.total, event.timestamp
        ])
    
    def on_payment_charged(self, event: PaymentCharged):
        self.db.execute("""
            UPDATE orders_view
            SET status = 'PAID', payment_amount = ?, payment_method = ?
            WHERE order_id = ?
        """, [event.amount, event.method, event.order_id])
    
    def on_order_shipped(self, event: OrderShipped):
        self.db.execute("""
            UPDATE orders_view
            SET status = 'SHIPPED', tracking_number = ?
            WHERE order_id = ?
        """, [event.tracking, event.order_id])
```

## Query Side

```python
@app.route('/api/orders/<order_id>')
def get_order(order_id):
    # Simple, fast query against denormalized read model
    order = read_db.fetch_one(
        "SELECT * FROM orders_view WHERE order_id = ?", [order_id]
    )
    return jsonify(order)

@app.route('/api/orders/search')
def search_orders():
    # Rich query against read-optimized store (e.g., Elasticsearch)
    results = elasticsearch.search(
        index='orders',
        body={
            'query': {
                'bool': {
                    'must': [
                        {'match': {'customer_name': request.args['q']}},
                        {'term': {'status': request.args.get('status', 'PAID')}}
                    ]
                }
            }
        }
    )
    return jsonify(results)

@app.route('/api/dashboard/revenue')
def revenue_dashboard():
    # Aggregation query against read model
    return read_db.fetch_all("""
        SELECT DATE(created_at) as date,
               SUM(total) as revenue,
               COUNT(*) as order_count
        FROM orders_view
        WHERE created_at >= DATE('now', '-30 days')
        GROUP BY DATE(created_at)
        ORDER BY date
    """)
```

## When to Use CQRS

```
USE CQRS when:
  ✓ Read and write patterns are radically different
    - Simple writes, complex queries with joins/aggregations
    - Different scaling requirements (1000 writes/sec, 10000 reads/sec)
  
  ✓ Multiple read models for different consumers
    - Mobile app needs lightweight response
    - Admin dashboard needs full details + aggregations
    - Search needs full-text indexed model

DON'T USE CQRS when:
  ✗ Simple CRUD with similar read/write patterns
  ✗ Strong consistency between write and read is required
    (CQRS is eventually consistent between write and read sides)
  ✗ Small system with low complexity
    (CQRS adds significant infrastructure complexity)
```

## Eventual Consistency

The read model is UPDATED ASYNCHRONOUSLY from the write model. After a write, the read model may be stale for milliseconds to seconds. Compensate with:

```python
# Read-your-writes: route reads for recently modified data to write side
def get_order(order_id, user_id):
    # If user recently modified this, read from write side
    if was_recently_written_by(user_id, order_id, seconds=5):
        return write_db.fetch_one(...)  # Strongly consistent
    return read_db.fetch_one(...)       # Eventually consistent
```

CQRS adds complexity but unlocks independent scaling of reads and writes, query-optimized read models, and the ability to build new read models from existing events without touching write-side code.
