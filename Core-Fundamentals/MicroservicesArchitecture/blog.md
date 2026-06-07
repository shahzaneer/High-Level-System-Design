# Microservices Architecture

## Introduction
Microservices architecture emerged as a response to the limitations of monolithic applications at scale. Pioneered by companies like Amazon (early 2000s), Netflix (2009), and Uber, the pattern decomposes applications into independently deployable, loosely coupled services. Each service owns its own business capability, data store, and development lifecycle. While the term gained mainstream traction around 2014 with Martin Fowler's and James Lewis's seminal article, the principles—modularity, loose coupling, and independent deployment—trace back to distributed computing and service-oriented architecture (SOA).

Today, microservices are not a universal solution but a widely adopted architectural style for organizations needing independent team autonomy, rapid iteration, and horizontal scalability. The critical skill for solution architects is knowing when to apply microservices and when a monolithic or modular monolith approach is more appropriate.

## Definition
**Microservices Architecture** is an architectural style that structures an application as a collection of small, autonomous services modeled around business domains. Each service:
- Is independently deployable and scalable
- Owns its own data store (database per service)
- Communicates over well-defined APIs (REST, gRPC, message queues)
- Is developed and operated by a small, cross-functional team
- Can be built using different technology stacks (polyglot persistence and programming)

## Concept Explanation

### Core Principles

#### 1. Single Responsibility / Business Capability
Each microservice owns one business capability from end to end. Not "one service per technical layer" (a services that does all user CRUD), but "one service per business function" (billing service, shipping service, recommendation service).

#### 2. Decentralized Data Management
Each service has its own database. No service directly accesses another service's database. This prevents the distributed monolith anti-pattern where services share databases and become tightly coupled at the data layer.

```
[Order Service]           [Inventory Service]        [Payment Service]
     │                          │                          │
[Order DB]                [Inventory DB]             [Payment DB]
```

#### 3. Communication Patterns

**Synchronous (Request-Response)**:
```python
# Service A calls Service B directly
response = requests.get("http://payment-service/charge", json={"amount": 99.99})
```

**Asynchronous (Event-Driven)**:
```python
# Order service publishes event after order is placed
kafka_producer.send("orders", {"event": "OrderPlaced", "order_id": 123})

# Payment service subscribes and processes
for message in kafka_consumer:
    if message["event"] == "OrderPlaced":
        process_payment(message["order_id"])
```

**Choreography vs Orchestration**:
- **Choreography**: Services react to events independently. No central coordinator. Like dancers who know their steps without a conductor.
- **Orchestration**: A central service (orchestrator/saga orchestrator) coordinates the workflow. Like a conductor directing an orchestra.

#### 4. API Gateway
A single entry point for clients that routes requests to appropriate microservices. Handles cross-cutting concerns: authentication, rate limiting, request transformation, and protocol translation.

#### 5. Service Discovery
Services need to find each other's network locations dynamically (since instances come and go with auto-scaling):

```python
# Consul-based service discovery
import consul
c = consul.Consul()
_, services = c.catalog.service("payment-service")
# Returns list of healthy payment service instances
```

### Design Patterns for Microservices

#### Saga Pattern (Distributed Transactions)
Since each service has its own database, you cannot use ACID transactions across services. Sagas implement distributed transactions via a sequence of local transactions with compensating actions for rollback:

```
Create Order → Reserve Inventory → Charge Payment → Ship
                   ↓ (failure)
          Compensate: Cancel Order + Refund Payment
```

```python
# Orchestrator-based saga
def create_order_saga(order):
    try:
        order_id = order_service.create(order)           # Local transaction
        inventory_service.reserve(order.items)            # Local transaction
        payment_service.charge(order.total)               # Local transaction
        
    except InventoryUnavailableError:
        # Compensating transaction
        order_service.cancel(order_id)
        raise
        
    except PaymentFailedError:
        # Compensating transactions in reverse order
        inventory_service.release(order.items)
        order_service.cancel(order_id)
        raise
```

#### Circuit Breaker
Prevents cascading failures. If a downstream service fails, the circuit breaker "opens" and fast-fails subsequent requests, giving the downstream service time to recover:

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
def call_payment_service(amount):
    response = requests.post("http://payment-service/charge", json={"amount": amount})
    if response.status_code >= 500:
        raise Exception("Payment service error")
    return response.json()
```

#### CQRS (Command Query Responsibility Segregation)
Separate read and write models. Commands (writes) use a normalized model optimized for transactional integrity. Queries (reads) use a denormalized model optimized for the specific read patterns:

```
[Command Service] ──write──→ [Write DB] ──events──→ [Read DB] ←──read── [Query Service]
```

Useful when read and write patterns are radically different (e.g., e-commerce where you rarely write a product but display it in dozens of different views).

#### Event Sourcing
Instead of storing the current state, store the sequence of events that led to the current state. The current state is a projection of the event log:

```python
# Event store
events = event_store.get_events("order-123")
# [
#   {"type": "OrderPlaced", "items": [...]},
#   {"type": "PaymentCharged", "amount": 99.99},
#   {"type": "OrderShipped", "tracking": "1Z999AA..."}
# ]

# Current state is derived from events
current_state = reduce(apply_event, events, initial_state={})
```

### When NOT to Use Microservices

Microservices introduce significant operational complexity. The rule of thumb:

- **Start with a monolith** (or modular monolith) for new products until you have proven product-market fit
- **Extract microservices** when you have clear bounded contexts, independent scaling needs, and the organizational structure to support them (Conway's Law)
- **Don't do microservices** if you have fewer than 3 development teams, no need for independent scaling of components, or lack DevOps maturity for container orchestration and observability

## Layman's Explanation

### Microservices: The Food Court
A traditional restaurant (monolith) has one kitchen that does everything—burgers, sushi, pizza, desserts. When it's busy, the entire kitchen backs up. If the pizza oven breaks, the whole restaurant closes. The kitchen staff must all coordinate on the same schedule, use the same tools, and any change requires everyone to adapt.

A food court (microservices) has independent stalls: the burger stall, sushi counter, pizza station, and dessert kiosk. Each stall:
- Has its own kitchen, equipment, and cash register (database per service)
- Can stay open or close independently
- Can scale up (add more cooks) without affecting other stalls
- The burger stall can be replaced without affecting the sushi counter
- Customers talk to each stall in their own "language" (API)

The trade-off: The food court needs a central directory (service discovery), a way to handle payments across stalls (API gateway), and customers sometimes need to visit multiple stalls (orchestration). The restaurant is simpler to manage, but the food court scales better.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Boundary Identification**: The hardest part of microservices is getting the boundaries right. Wrong boundaries create a distributed monolith—all the complexity of microservices with all the coupling of a monolith. Domain-Driven Design (DDD) bounded contexts guide this decision.
- **Data Decomposition**: Breaking a monolithic database into per-service databases requires understanding data ownership, foreign key boundaries, and how data that was previously joined via SQL will be joined at the application layer (or denormalized into read models).
- **Communication Protocol**: Synchronous REST/gRPC for real-time needs vs. asynchronous messaging for resilience and decoupling. Wrong choice leads to tight coupling and cascading failures.
- **Observability Strategy**: Monolith debugging is "check the logs." Microservice debugging requires distributed tracing (Jaeger, Zipkin), centralized logging (ELK, Loki), and metrics aggregation (Prometheus, Datadog).

### Business Impact
- **Team Autonomy**: Microservices enable small teams to own services independently—different release cadences, different tech stacks, different deployment schedules. This maps to Amazon's "two-pizza team" philosophy.
- **Time to Market**: Independent deployment means the payment team can ship a new feature without coordinating with the notification team. No release train. No merge conflicts.
- **Scalability Independence**: Scale only the services that need it. During Black Friday, scale up the product catalog and checkout services 10x without touching the reporting service.
- **Technology Evolution**: A Python monolith that needs a machine learning component can add a Python microservice without rewriting 500,000 lines of existing Java code.

### Organizational Alignment (Conway's Law)
"Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations." If your team structure doesn't match your microservice boundaries, you've created organizational chaos alongside technical chaos. The "Inverse Conway Maneuver": design your team structure first, and the architecture will follow.

## On-Premises Examples

### Docker Compose for Development
```yaml
# docker-compose.yml
services:
  api-gateway:
    build: ./api-gateway
    ports: ["8080:8080"]
    depends_on: [order-service, payment-service]

  order-service:
    build: ./order-service
    environment:
      DATABASE_URL: postgresql://order-db/orders
    depends_on: [order-db, kafka]

  payment-service:
    build: ./payment-service
    environment:
      DATABASE_URL: postgresql://payment-db/payments
    depends_on: [payment-db, kafka]

  order-db:
    image: postgres:16
    volumes: [order_data:/var/lib/postgresql/data]

  payment-db:
    image: postgres:16
    volumes: [payment_data:/var/lib/postgresql/data]

  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
```

### Kubernetes (On-Prem with kubeadm)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: registry.internal/order-service:1.2.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: order-db-credentials
              key: host
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 8080
    targetPort: 8080
```

### Service Mesh (Istio + Envoy)
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        version:
          exact: v2
    route:
    - destination:
        host: order-service
        subset: v2
      weight: 10  # 10% canary for v2
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 90
```

## AWS Examples

### Amazon ECS + Fargate (Serverless Containers)
```hcl
resource "aws_ecs_service" "order_service" {
  name            = "order-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.order.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.order_service.id]
  }

  service_registries {
    registry_arn = aws_service_discovery_service.order.arn
  }
}

# Cloud Map for service discovery
resource "aws_service_discovery_service" "order" {
  name = "order-service"
  
  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id
    dns_records {
      ttl  = 10
      type = "A"
    }
  }
}
```

### AWS Lambda (Serverless Microservices)
```python
# Order service as Lambda function
import json
import boto3

dynamodb = boto3.resource('dynamodb')
eventbridge = boto3.client('events')

def lambda_handler(event, context):
    order = json.loads(event['body'])
    
    table = dynamodb.Table('Orders')
    table.put_item(Item={
        'OrderID': order['id'],
        'Status': 'PLACED',
        'Items': order['items']
    })
    
    # Emit event to EventBridge (pub-sub for other services)
    eventbridge.put_events(Entries=[{
        'Source': 'com.myapp.orders',
        'DetailType': 'OrderPlaced',
        'Detail': json.dumps(order)
    }])
    
    return {'statusCode': 201, 'body': json.dumps({'order_id': order['id']})}
```

### AWS Step Functions (Saga Orchestration)
```json
{
  "Comment": "Order processing saga",
  "StartAt": "CreateOrder",
  "States": {
    "CreateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:create-order",
      "Next": "ReserveInventory"
    },
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:reserve-inventory",
      "Catch": [{"ErrorEquals": ["InsufficientInventory"], "Next": "CancelOrder"}],
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "Catch": [{"ErrorEquals": ["PaymentFailed"], "Next": "ReleaseInventory"}],
      "Next": "OrderComplete"
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "Next": "CancelOrder"
    },
    "CancelOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "End": true
    },
    "OrderComplete": {
      "Type": "Succeed"
    }
  }
}
```

## GCP Examples

### Cloud Run (Serverless Containers)
```bash
# Deploy order service
gcloud run deploy order-service \
  --image=gcr.io/my-project/order-service:v1.2.0 \
  --region=us-central1 \
  --platform=managed \
  --allow-unauthenticated \
  --set-env-vars "DB_HOST=10.0.0.5"

# Cloud Run auto-scales from 0 to N instances
# Each instance gets its own URL, but Cloud Run load balances
```

### GKE (Managed Kubernetes)
```bash
gcloud container clusters create microservices-cluster \
  --region=us-central1 \
  --num-nodes=3 \
  --enable-autoscaling \
  --min-nodes=1 --max-nodes=10 \
  --workload-pool=my-project.svc.id.goog

# Deploy with Workload Identity for secure service-to-service auth
kubectl apply -f order-service.yaml
```

### Eventarc (Event-Driven)
```bash
# Route Cloud Storage events to Cloud Run
gcloud eventarc triggers create order-image-trigger \
  --location=us-central1 \
  --destination-run-service=image-processor \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=order-images"
```

## Azure Examples

### Azure Kubernetes Service (AKS)
```bash
az aks create \
  --resource-group myResourceGroup \
  --name microservices-cluster \
  --node-count 3 \
  --enable-cluster-autoscaler \
  --min-count 1 --max-count 10 \
  --network-plugin azure \
  --network-policy calico
```

### Azure Container Apps (Serverless Microservices)
```bash
az containerapp create \
  --name order-service \
  --resource-group myResourceGroup \
  --environment managedEnvironment \
  --image registry.azurecr.io/order-service:v1.2.0 \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 --max-replicas 5 \
  --scale-rule-name http-requests \
  --scale-rule-type http \
  --scale-rule-http-concurrency 50
```

### Azure Durable Functions (Saga/Orchestration)
```csharp
[FunctionName("OrderOrchestrator")]
public static async Task RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var order = context.GetInput<Order>();

    await context.CallActivityAsync("CreateOrder", order);
    
    try
    {
        await context.CallActivityAsync("ReserveInventory", order);
    }
    catch
    {
        await context.CallActivityAsync("CancelOrder", order);
        throw;
    }

    await context.CallActivityAsync("ChargePayment", order);
    await context.CallActivityAsync("ShipOrder", order);
}
```

## Summary Decision Matrix

| Factor | Monolith | Microservices |
|--------|----------|---------------|
| Team size | 1-2 teams (3-15 devs) | 3+ teams, each owning services |
| Deployment complexity | Simple (one artifact) | Complex (CI/CD per service, orchestration) |
| Debugging | Simple (single process) | Complex (distributed tracing required) |
| Scalability | Whole application scales | Independent per-service scaling |
| Technology flexibility | One stack (mostly) | Polyglot (per service) |
| Data consistency | ACID transactions | Eventual consistency, sagas |
| Release velocity | Slower (coordinated releases) | Faster (independent releases) |
| Operational overhead | Low | High (requires DevOps/Platform Engineering) |

Microservices are an organizational pattern as much as a technical one. The decision to adopt them should be driven by team scaling needs, not just technical scaling needs. A well-designed modular monolith serves many organizations better than poorly bounded microservices.
