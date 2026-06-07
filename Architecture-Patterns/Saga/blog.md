# Saga Pattern

A Saga is a distributed transaction pattern that maintains data consistency across multiple services, each with its own database. Since ACID transactions don't span service boundaries in microservices, Sagas implement a sequence of local transactions, each followed by an event or command that triggers the next step. If any step fails, compensating transactions undo the preceding steps.

There are two implementation styles: Choreography (services react to events independently, like dancers who know their steps) and Orchestration (a central coordinator directs each step, like a conductor). Both achieve eventual consistency across services.

## Choreography (Event-Driven)

Each service listens for events and performs its local transaction, then publishes its own event.

```
Order Service:  CREATE order → emit OrderPlaced
Inventory:      ON OrderPlaced → RESERVE stock → emit InventoryReserved
Payment:        ON InventoryReserved → CHARGE payment → emit PaymentCharged
Shipping:       ON PaymentCharged → CREATE shipment → emit OrderShipped
```

**Failure handling**:
```
Payment: ON InventoryReserved → CHARGE FAILS → emit PaymentFailed
Order:   ON PaymentFailed → COMPENSATE (cancel order) → emit OrderCancelled
Inventory: ON OrderCancelled → COMPENSATE (release stock) → emit InventoryReleased
```

Implementation:
```python
# Order Service
def create_order(order_data):
    order = db.insert('orders', order_data)
    event_bus.publish('OrderPlaced', {'order_id': order.id, 'items': order_data['items']})
    return order

def handle_order_cancelled(event):
    db.update('orders', event['order_id'], {'status': 'CANCELLED'})

# Inventory Service
def handle_order_placed(event):
    try:
        for item in event['items']:
            db.decrement('inventory', item['sku'], item['quantity'])
        event_bus.publish('InventoryReserved', {'order_id': event['order_id']})
    except InsufficientStock:
        event_bus.publish('InventoryFailed', {'order_id': event['order_id'], 'reason': 'out_of_stock'})
```

## Orchestration (Coordinator)

A central Saga orchestrator manages the workflow explicitly.

```python
class OrderSagaOrchestrator:
    def execute(self, order_data):
        saga_id = str(uuid.uuid4())
        self.state = {'saga_id': saga_id, 'status': 'STARTED', 'steps_completed': []}
        
        steps = [
            ('CreateOrder', self._create_order, self._cancel_order),
            ('ReserveInventory', self._reserve_inventory, self._release_inventory),
            ('ChargePayment', self._charge_payment, self._refund_payment),
            ('CreateShipment', self._create_shipment, None),  # No compensation needed
        ]
        
        for step_name, action, compensation in steps:
            try:
                result = action(order_data, self.state)
                self.state['steps_completed'].append(step_name)
                self.state[step_name] = result
            except Exception as e:
                # Compensate: undo completed steps in reverse order
                self._compensate(compensation)
                raise SagaFailedError(f"Failed at {step_name}: {e}")
        
        self.state['status'] = 'COMPLETED'
        return self.state
    
    def _compensate(self, failed_step_name):
        completed = self.state['steps_completed']
        compensation_map = {
            'CreateOrder': self._cancel_order,
            'ReserveInventory': self._release_inventory,
            'ChargePayment': self._refund_payment,
        }
        for step_name in reversed(completed):
            if step_name in compensation_map:
                compensation_map[step_name](self.state)
```

### Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| Coupling | Loose (events only) | Tighter (orchestrator knows all steps) |
| Visibility | Hard to see overall state | Centralized state |
| Debugging | Trace events across services | Inspect orchestrator state |
| Complexity | Spread across services | Centralized in orchestrator |
| When to use | Simple flows, independent teams | Complex flows, visibility needed |

### Key Concerns
- **Idempotency**: Every step handler must be idempotent—events may be delivered multiple times
- **Exactly-once semantics**: Use idempotency keys or deduplication tables
- **Timeout**: Sagas can take time. Track timeout; trigger compensation if stuck
- **Isolation**: Between steps, other transactions may see intermediate state. Design for it.

Sagas trade ACID for BASE. They provide eventual consistency across service boundaries—the fundamental transaction pattern for microservices.
