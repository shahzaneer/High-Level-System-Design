# Modernization Patterns

## Introduction
Application Modernization is the process of transforming legacy applications to leverage cloud-native capabilities—elasticity, resilience, managed services, and DevOps agility. Unlike "lift and shift" (rehost), which moves applications without changing them, modernization modifies the application to extract greater value from the cloud. The Strangler Fig pattern (Martin Fowler, 2004) is the canonical approach, but a family of patterns has emerged to handle different modernization scenarios.

Modernization is the most value-creating but riskiest cloud migration strategy. Done well, it transforms a monthly $50K monolithic application into a $15K serverless system that deploys 10x faster and handles 10x the load. Done poorly, it becomes a multi-year rewrite with no intermediate value delivery—the "big bang" rewrite anti-pattern that has killed countless modernization projects.

## Definition

**Application Modernization** is the process of updating legacy software to newer computing approaches, including cloud-native architectures, microservices, serverless computing, DevOps practices, and managed services.

**Key Modernization Patterns**:

1. **Strangler Fig**: Incrementally replace legacy system functions with new implementations
2. **Anti-Corruption Layer**: Protect new systems from legacy system design flaws
3. **Event Interception**: Capture events from legacy system to drive new functionality
4. **Parallel Run**: Run old and new systems simultaneously, comparing outputs
5. **Sidecar / Ambassador**: Attach modernization capabilities to legacy containers
6. **Database Decomposition**: Break monolithic databases into per-service databases

## Concept Explanation

### Pattern 1: Strangler Fig

```
                    ┌─────────────────────┐
                    │    API Gateway       │
                    │  (Routes requests)   │
                    └──────┬──────────┬────┘
                           │          │
              ┌────────────▼──┐   ┌──▼──────────────┐
              │  NEW Services  │   │  LEGACY Monolith │
              │  (microservices)│   │  (shrinking)     │
              │                │   │                  │
              │  Order Service │   │  Everything else │
              │  Payment Svc   │   │  (being strangled)│
              │  Catalog Svc   │   │                  │
              └────────────────┘   └──────────────────┘
```

```python
class StranglerRouter:
    """
    Routes requests to either new microservice or legacy monolith
    based on which functionality has been extracted.
    """
    
    def __init__(self):
        self.migration_map = {
            'POST /orders': 'order-service',      # Extracted
            'GET /orders': 'order-service',        # Extracted
            'POST /orders/:id/cancel': 'order-service',  # Extracted
            'POST /payments': 'payment-service',   # Extracted
            'GET /catalog': 'catalog-service',     # Extracted
            # Everything else → legacy monolith
        }
    
    def route(self, request):
        key = f"{request.method} {self._normalize_path(request.path)}"
        target = self.migration_map.get(key, 'legacy-monolith')
        
        if target == 'legacy-monolith':
            return proxy_to_legacy(request)
        else:
            return proxy_to_service(target, request)
    
    def extract_functionality(self, new_service, paths):
        """After new service is deployed and tested, flip the route"""
        for path in paths:
            self.migration_map[path] = new_service
        # Deploy the routing change (e.g., update API Gateway config)
```

### Pattern 2: Anti-Corruption Layer (ACL)

```python
class LegacyOrderAdapter:
    """
    New systems should NOT inherit legacy design flaws.
    The ACL translates between clean new domain model and messy legacy model.
    """
    
    def create_order(self, new_order_dto: NewOrderDTO) -> str:
        # New system: clean, validated DTO
        # Legacy system: nested SOAP XML with inconsistent field names
        
        legacy_payload = {
            'ORD_HDR': {
                'CUST_NUM': new_order_dto.customer_id,
                'ORD_TOTAL': str(new_order_dto.total_cents / 100),
                'ORD_DATE': new_order_dto.created_at.strftime('%m/%d/%Y'),
                'STATUS_CD': self._map_status(new_order_dto.status)
            },
            'ORD_LINES': [
                {
                    'SKU': item.sku,
                    'QTY': str(item.quantity),
                    'UNIT_PRC': str(item.unit_price_cents / 100)
                }
                for item in new_order_dto.items
            ]
        }
        
        # Call legacy SOAP endpoint
        response = self._legacy_soap_call('CreateOrder', legacy_payload)
        
        # Translate legacy response back to new domain
        return self._parse_legacy_response(response)
    
    def _map_status(self, new_status: str) -> str:
        """Legacy uses numeric codes, new system uses descriptive strings"""
        STATUS_MAP = {
            'PENDING': '01',
            'CONFIRMED': '02',
            'SHIPPED': '03',
            'CANCELLED': '99'
        }
        return STATUS_MAP.get(new_status, '01')
```

### Pattern 3: Event Interception

```python
class LegacyEventInterceptor:
    """
    Legacy system emits data changes via database triggers or CDC.
    New system subscribes to these events for additional processing.
    Legacy system is UNCHANGED—it doesn't know the new system exists.
    """
    
    def __init__(self):
        # Debezium connector captures MySQL binlog changes
        # Publishes to Kafka: {table}.{database}.{event_type}
        self.topics = {
            'legacy_db.orders.INSERT': self.on_order_created,
            'legacy_db.orders.UPDATE': self.on_order_updated,
            'legacy_db.customers.INSERT': self.on_customer_created
        }
    
    def on_order_created(self, event):
        # Legacy order created → create event in new event-driven system
        order_data = event['after']  # Debezium provides before/after state
        
        # New system reactions (legacy doesn't know about these):
        new_event_bus.publish('OrderPlaced', {
            'order_id': order_data['id'],
            'customer_id': order_data['customer_id'],
            'total': order_data['total'],
            'source': 'LEGACY_INTERCEPTION'
        })
        
        # New recommendation engine
        recommendation_engine.on_purchase(order_data['customer_id'], order_data['items'])
        
        # New analytics pipeline
        analytics.track_order(order_data)
    
    def on_order_updated(self, event):
        if event['after']['status'] == 'SHIPPED' and event['before']['status'] != 'SHIPPED':
            # Legacy order shipped → trigger new notification service
            new_event_bus.publish('OrderShipped', {...})
```

### Pattern 4: Parallel Run

```python
class ParallelRunValidator:
    """
    Run old and new systems in parallel. Compare results.
    Old system handles production traffic. 
    New system runs in shadow mode with mirrored requests.
    """
    
    def process_request(self, request):
        # 1. Route to old system (production)
        old_result = self.old_system.process(request)
        
        # 2. Mirror request to new system (shadow)
        try:
            new_result = self.new_system.process(request)
            
            # 3. Compare results
            differences = self._compare(old_result, new_result)
            if differences:
                # Log differences but DON'T surface to user
                self._log_discrepancy(request, old_result, new_result, differences)
                
                # Track divergence rate
                self.metrics.increment('parallel_run.divergence')
        
        except Exception as e:
            # New system error → log but don't surface
            self.metrics.increment('parallel_run.new_system_error')
            self._log_error(request, e)
        
        # Always return old system result
        return old_result
    
    def promote_to_production(self):
        """After parallel run shows acceptable divergence rate (<0.1%)"""
        divergence_rate = self.metrics.divergence_rate_last_30_days()
        if divergence_rate < 0.001:  # <0.1% divergence
            logger.info("Promoting new system to production")
            # Cutover: swap routing to new system
            # Keep old system in shadow mode for 1 week as rollback insurance
```

### Pattern 5: Database Decomposition

```
MONOLITHIC DATABASE:
┌──────────────────────────────┐
│  orders │ customers │ payments │ shipping │ inventory │
├──────────────────────────────┤
│  Foreign keys across tables  │
│  Single schema dependencies  │
└──────────────────────────────┘

DECOMPOSED DATABASES:
┌──────────┐  ┌───────────┐  ┌──────────┐  ┌──────────┐
│ Orders DB│  │Customer DB│  │Payment DB│  │Shipping DB│
└──────────┘  └───────────┘  └──────────┘  └──────────┘
```

```python
class DatabaseDecomposer:
    """
    Break monolithic database into per-service databases.
    This is the hardest part of modernization.
    """
    
    def decompose_orders(self):
        # Phase 1: Dual writes
        # Write to BOTH monolithic DB and new Orders DB
        # Read from monolithic DB only (new DB isn't trusted yet)
        
        # Phase 2: Backfill
        # Copy all historical orders from monolithic to new Orders DB
        # Verify consistency (row count, checksum per customer)
        
        # Phase 3: Read verification
        # Read from new Orders DB in shadow mode
        # Compare with monolithic reads
        # Fix discrepancies
        
        # Phase 4: Cutover reads
        # orders-service reads from new Orders DB
        # Monolithic still writes to both
        
        # Phase 5: Cutoff writes
        # Monolithic stops writing to old DB for orders
        # Old orders table frozen (kept for rollback)
        
        # Phase 6: Cleanup
        # After 30 days of no rollback: drop old orders table
```

## Layman's Explanation

### Strangler Fig: Renovating While Living in the House
You're renovating a house (legacy application) while still living in it:

1. The kitchen is old and inefficient. You build a new kitchen in the garage (extract a microservice).
2. You redirect meal preparation to the new kitchen (route /orders to new service).
3. The family keeps using the old kitchen's dining area (remaining functionality stays in monolith).
4. Next, you renovate the bathroom (extract another service). The old bathroom is demolished.
5. Eventually, the entire house is renovated—no original room remains. The "house" is now entirely new, but you lived in it the whole time.

You never had to move out (big bang rewrite). At no point was the kitchen or bathroom unusable (zero downtime). Each renovation delivered immediate value.

### The Vine Analogy (Why It's Called Strangler Fig)
In nature, a strangler fig seed germinates in the canopy of a host tree. It grows downward, eventually reaching the ground and developing its own root system. The fig gradually envelops the host tree, which eventually dies and rots away, leaving the fig standing independently. The legacy system is "strangled" over time as new services replace its functions.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Incremental Value Delivery**: The biggest modernization risk is the multi-year rewrite that delivers nothing until the final cutover. Strangler Fig delivers value incrementally—each extracted service is production-ready and provides immediate benefit. Stakeholder patience lasts months, not years.
- **Data Decomposition is the Hardest Part**: Breaking a monolithic database into per-service databases while maintaining referential integrity (what was a SQL JOIN is now an API call + eventual consistency) is the hardest technical challenge. Plan for this explicitly—it takes 60-70% of modernization effort.
- **Rollback Strategy per Extraction**: Each strangled service must be independently rollbackable. If the new order service fails, traffic routes back to the monolith's order functionality. This requires the monolith to remain functional until ALL services are extracted and stable.
- **Accepting the "Remaining Monolith"**: Not everything needs extraction. The monolith may shrink to 30% of its original size and remain there. Accept this as a successful outcome. The goal is not zero monolith; it's the right-size monolith with critical functions extracted.

### Business Impact
- **Risk Reduction**: Strangler Fig limits blast radius. A failed extraction affects one business function, not all of them. The business can tolerate one service being degraded while keeping revenue-generating functions operational.
- **Faster Time-to-Market**: Each extracted service can be enhanced independently. A team can ship new payment features in the payment service without coordinating with the monolith's release schedule. This compounds over time.
- **ROI Justification**: Each extraction delivers measurable value (e.g., "catalog page load time reduced by 60%"). This creates a positive feedback loop—each success justifies the next extraction, rather than waiting years for one big payoff.

## On-Premises Examples

### Feature Flag-Driven Strangler
```python
from flask import Flask, request, jsonify
import ldclient

app = Flask(__name__)
ld_client = ldclient.LDClient("sdk-key")

@app.route('/api/orders', methods=['POST'])
def create_order():
    user = get_current_user()
    
    # Feature flag: which users go to new order service?
    if ld_client.variation('new-order-service-enabled', user, False):
        return proxy_to_new_order_service(request)
    else:
        return legacy_create_order(request)

# Gradually increase flag:
# Day 1: 1% of users → new service
# Day 2: 5% → if error rate OK
# Day 7: 25% → if latency improved
# Day 14: 100% → fully migrated
```

### Change Data Capture (CDC) with Debezium
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: legacy-db-connector
spec:
  class: io.debezium.connector.mysql.MySqlConnector
  config:
    database.hostname: legacy-db.internal
    database.port: 3306
    database.user: debezium
    database.server.id: 1
    database.include.list: legacy_db
    table.include.list: legacy_db.orders,legacy_db.customers
    transforms: unwrap
    transforms.unwrap.type: io.debezium.transforms.ExtractNewRecordState
```

## AWS Examples

### AWS DMS (Database Migration + CDC)
```hcl
resource "aws_dms_replication_task" "migrate_orders" {
  migration_type           = "full-load-and-cdc"  # Full + ongoing changes
  replication_task_id      = "orders-migration"
  source_endpoint_arn      = aws_dms_endpoint.legacy.arn
  target_endpoint_arn      = aws_dms_endpoint.aurora.arn
  replication_instance_arn = aws_dms_replication_instance.main.arn

  table_mappings = jsonencode({
    rules = [{
      "rule-type" = "selection"
      "rule-id"   = "1"
      "rule-name" = "orders-only"
      "object-locator" = {
        "schema-name" = "legacy_db"
        "table-name"  = "orders"
      }
      "rule-action" = "include"
    }]
  })
}
```

### API Gateway Canary Deployments
```yaml
# Strangler via canary: gradually route to new service
DeploymentCanarySettings:
  PercentTraffic: 10.0
  StageVariableOverrides:
    lambdaAlias: canary
  UseStageCache: false
```

## GCP Examples

### Cloud Data Fusion (Legacy to Cloud ETL)
```bash
# Visual pipeline to extract from legacy DB
# Transform: modernize schema
# Load into BigQuery
gcloud data-fusion instances create modernization-pipeline \
  --location=us-central1
```

## Azure Examples

### Azure App Service Migration Assistant
```bash
# Assess and migrate .NET apps to Azure
dotnet tool install -g dotnet-app-migration
app-migration assess --project-path ./LegacyApp

# Generates modernization roadmap:
# - Compatibility issues
# - Required code changes
# - Containerization recommendations
```

## Summary

| Modernization Pattern | When to Use | Effort | Risk |
|----------------------|-------------|--------|------|
| Strangler Fig | Most common; incremental extraction | Medium-High (per service) | Low (incremental) |
| Anti-Corruption Layer | Legacy has poor domain model | Medium | Low |
| Event Interception | Legacy cannot be modified | Low | Low |
| Parallel Run | High accuracy requirement | Medium | Low |
| Big Bang Rewrite | Rarely justified | Very High | Very High |

Modernization is a marathon, not a sprint. The Strangler Fig pattern enables incremental progress with continuous value delivery. Start with the most valuable and least risky extraction (often read-heavy functionality like catalog/search), build confidence, then progress to more complex extractions. Never attempt a "big bang" rewrite. The architects who succeed at modernization are those who deliver working software every few weeks, not those who promise perfection in two years.
