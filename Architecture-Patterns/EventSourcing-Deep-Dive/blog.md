# Event Sourcing Deep Dive

Event Sourcing stores state as a sequence of immutable events rather than as current state. Instead of `UPDATE orders SET status='SHIPPED'`, you APPEND `OrderShipped { tracking: '1Z999AA' }`. Current state is derived by replaying all events from the beginning. The event store is the source of truth—the database of record, the audit log, and the temporal query engine all in one.

Event Sourcing is NOT CQRS (they're orthogonal but complementary). It IS NOT the Outbox pattern (outbox publishes events from a state-based store; event sourcing the events ARE the store).

## The Event Store

```python
class EventStore:
    def __init__(self, db):
        self.db = db
    
    def save_events(self, aggregate_id, events, expected_version):
        """Append events atomically with optimistic concurrency control"""
        with self.db.transaction():
            current_version = self.db.fetch_one(
                "SELECT MAX(version) FROM events WHERE aggregate_id = ?",
                [aggregate_id]
            ) or 0
            
            if expected_version != current_version:
                raise ConcurrencyConflict(
                    f"Expected version {expected_version}, got {current_version}"
                )
            
            for i, event in enumerate(events):
                self.db.execute("""
                    INSERT INTO events (aggregate_id, version, event_type, payload, timestamp)
                    VALUES (?, ?, ?, ?, ?)
                """, [
                    aggregate_id,
                    expected_version + i + 1,
                    event['type'],
                    json.dumps(event['payload']),
                    datetime.utcnow()
                ])

# Events table:
# aggregate_id | version | event_type      | payload                    | timestamp
# order-123    | 1       | OrderPlaced      | {"items":[...],"total":99} | 10:00
# order-123    | 2       | PaymentCharged   | {"amount":99.99}           | 10:01
# order-123    | 3       | OrderShipped     | {"tracking":"1Z999"}       | 10:30
```

## Aggregate Root

```python
class Order:
    def __init__(self, order_id):
        self.order_id = order_id
        self.version = 0
        self.pending_events = []
        self.state = self._initial_state()
    
    @classmethod
    def create(cls, customer_id, items):
        order = cls(str(uuid.uuid4()))
        order._apply(OrderPlaced(customer_id=customer_id, items=items, total=calculate_total(items)))
        return order
    
    @classmethod
    def from_events(cls, order_id, events):
        order = cls(order_id)
        for event in events:
            order._apply(event)
            order.version += 1
        return order
    
    def charge_payment(self, amount, method):
        if self.state['status'] != 'PLACED':
            raise DomainException("Can only charge payment for PLACED orders")
        self._apply(PaymentCharged(amount=amount, method=method))
    
    def ship(self, tracking_number):
        if self.state['status'] != 'PAID':
            raise DomainException("Can only ship PAID orders")
        self._apply(OrderShipped(tracking=tracking_number))
    
    def _apply(self, event):
        self.pending_events.append(event)
        event.apply(self.state)  # Mutate state
```

## Snapshots (Performance)

Replaying ALL events from the beginning is slow for long-lived aggregates. Snapshots capture state at a point in time.

```python
class SnapshotStore:
    def get_snapshot(self, aggregate_id):
        return self.db.fetch_one(
            "SELECT state, version FROM snapshots WHERE aggregate_id = ? ORDER BY version DESC LIMIT 1",
            [aggregate_id]
        )
    
    def save_snapshot(self, aggregate_id, state, version):
        self.db.execute(
            "INSERT INTO snapshots (aggregate_id, state, version, created_at) VALUES (?, ?, ?, ?)",
            [aggregate_id, json.dumps(state), version, datetime.utcnow()]
        )

def load_order(order_id):
    # Load most recent snapshot
    snapshot = snapshot_store.get_snapshot(order_id)
    
    if snapshot:
        order = Order(order_id)
        order.state = snapshot['state']
        order.version = snapshot['version']
        # Replay only events AFTER the snapshot
        events = event_store.get_events(order_id, after_version=snapshot['version'])
    else:
        order = Order(order_id)
        events = event_store.get_events(order_id)
    
    for event in events:
        order._apply(event)
        order.version += 1
    
    return order
```

## Time Travel

```python
# Query order state at any point in time
def get_order_at(order_id, timestamp):
    events = event_store.get_events_up_to(order_id, timestamp)
    return Order.from_events(order_id, events)

# What did this order look like yesterday at 3 PM?
order_yesterday = get_order_at('order-123', datetime(2024, 6, 15, 15, 0, 0))
```

## When to Use Event Sourcing

```
USE when:
  ✓ Complete audit trail required (finance, healthcare, compliance)
  ✓ Temporal queries needed ("what was the state on June 1st?")
  ✓ Debugging needs full history ("how did we get to this state?")
  ✓ Multiple read models from same events (CQRS + ES)

DON'T USE when:
  ✗ Simple CRUD with no audit requirements
  ✗ Eventual consistency between event store and projections is unacceptable
  ✗ Team unfamiliar with event-driven patterns
  ✗ Performance-critical writes (event store is append-only, but projections add latency)
```

Event Sourcing provides the ultimate audit trail and the ability to derive any state from the event history. The price is complexity: event schema evolution, snapshots, eventual consistency, and a fundamentally different mental model from CRUD.
