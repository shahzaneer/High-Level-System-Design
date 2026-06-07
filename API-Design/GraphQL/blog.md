# GraphQL

## Introduction
GraphQL is a query language and runtime for APIs developed by Facebook in 2012 and open-sourced in 2015. It addresses two fundamental REST limitations: over-fetching (receiving more data than needed) and under-fetching (needing multiple round-trips to gather related data). With GraphQL, clients specify exactly what data they need, and the server returns exactly that—nothing more, nothing less—in a single request.

GraphQL shifts power from the server to the client. The server defines a strongly-typed schema describing available data and operations. Clients query against that schema, composing flexible queries that traverse relationships between types. This is transformative for mobile applications and complex UIs where network bandwidth and round-trips are expensive. However, GraphQL introduces new challenges: query complexity management, N+1 resolution, caching, and security.

## Definition

**GraphQL** is a query language for APIs that allows clients to request exactly the data they need and nothing more, using a strongly-typed schema to define available types, queries, mutations, and subscriptions. Key concepts:

- **Schema**: Defines the types and operations available (the contract)
- **Query**: Read operation—client specifies fields to return
- **Mutation**: Write operation—client specifies fields to modify and return
- **Subscription**: Real-time operation—server pushes updates to subscribed clients
- **Resolver**: Function that fetches data for a specific field
- **Type System**: Scalar (String, Int, Float, Boolean, ID), Object, Enum, Interface, Union

## Concept Explanation

### Schema Definition

```graphql
type Query {
  orders(status: OrderStatus, first: Int, after: String): OrderConnection!
  order(id: ID!): Order
}

type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  cancelOrder(id: ID!): CancelOrderPayload!
}

type Subscription {
  orderUpdated(customerId: ID!): Order
}

type Order {
  id: ID!
  customer: Customer!
  items: [OrderItem!]!
  total: Float!
  status: OrderStatus!
  createdAt: String!
}

type Customer {
  id: ID!
  name: String!
  email: String!
  orders(first: Int): OrderConnection!
}

type OrderItem {
  sku: String!
  quantity: Int!
  unitPrice: Float!
  product: Product!
}

type Product {
  sku: String!
  name: String!
  description: String
  price: Float!
}

enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}

input CreateOrderInput {
  customerId: ID!
  items: [OrderItemInput!]!
}

type CreateOrderPayload {
  order: Order
  errors: [UserError!]
}

type UserError {
  message: String!
  field: String
}
```

### Query vs REST

```graphql
# GraphQL: one request gets exactly what's needed
query {
  orders(first: 10, status: PENDING) {
    edges {
      node {
        id
        total
        customer {
          name
          email
        }
        items {
          quantity
          product {
            name
            price
          }
        }
      }
    }
  }
}
```

```
REST equivalent: 3+ round trips
  GET /orders?status=pending&limit=10           → order IDs + customer IDs + product SKUs
  GET /customers?ids=1,2,3,4,5                  → customer names + emails
  GET /products?skus=SKU-1,SKU-2,SKU-3          → product names + prices

GraphQL: 1 round trip, exactly the data needed
  → 10 orders, their customers, their items, each product's name/price
```

### Resolvers (Data Fetching)

```python
import graphene
from graphene import relay

class OrderType(graphene.ObjectType):
    id = graphene.ID()
    customer = graphene.Field(CustomerType)
    items = graphene.List(OrderItemType)
    total = graphene.Float()
    status = graphene.String()

class Query(graphene.ObjectType):
    orders = graphene.List(OrderType, status=graphene.String())
    
    def resolve_orders(self, info, status=None):
        # Resolver: fetches orders from database
        query = Order.objects.all()
        if status:
            query = query.filter(status=status)
        return query[:10]
    
    def resolve_customer(self, info):
        # N+1 danger: this resolver called ONCE PER ORDER
        # Solution: DataLoader batches requests
        return dataloader.load_customer(self.customer_id)

# DataLoader: batches and caches resolver calls to prevent N+1
from promise import Promise
from promise.dataloader import DataLoader

class CustomerLoader(DataLoader):
    def batch_load_fn(self, customer_ids):
        customers = Customer.objects.filter(id__in=customer_ids)
        customer_map = {c.id: c for c in customers}
        return Promise.resolve([customer_map.get(cid) for cid in customer_ids])

customer_loader = CustomerLoader()

class OrderType(graphene.ObjectType):
    def resolve_customer(self, info):
        # DataLoader batches: 1 DB query for ALL customers, not 1 per order
        return customer_loader.load(self.customer_id)
```

### N+1 Problem and DataLoader

```
WITHOUT DataLoader:
  Query resolves 10 orders → 1 DB query
  For EACH order, resolve customer → 10 DB queries
  For EACH order, resolve items → 10 DB queries
  For EACH item, resolve product → 20 DB queries
  TOTAL: 41 DB queries for one GraphQL request

WITH DataLoader:
  Query resolves 10 orders → 1 DB query
  DataLoader batches all 10 customer IDs → 1 DB query
  DataLoader batches all order IDs → 1 DB query for items
  DataLoader batches all 20 product SKUs → 1 DB query
  TOTAL: 4 DB queries for one GraphQL request
```

### Query Complexity and Depth Limiting

```python
from graphql import validate, parse
from graphql.validation.rules import depth_limit_validator

# Prevent deeply nested queries (attack vector)
MAX_QUERY_DEPTH = 5

def validate_query(query_string, schema):
    document = parse(query_string)
    errors = validate(schema, document, [depth_limit_validator(MAX_QUERY_DEPTH)])
    
    if errors:
        return False, errors
    
    # Complexity analysis: each field costs 1, connections cost 10 × first
    complexity = calculate_complexity(document)
    if complexity > 1000:
        return False, [GraphQLError(f"Query too complex: {complexity}")]
    
    return True, None
```

### Persisted Queries (Production Security)

```python
# Store query on server; client sends hash instead of full query
# Benefits: reduced bandwidth, prevents arbitrary query execution

PERSISTED_QUERIES = {
    "a1b2c3": "query GetOrders($status: OrderStatus) { orders(status: $status) { id total } }"
}

@app.route('/graphql', methods=['POST'])
def graphql_handler():
    body = request.json
    
    if 'extensions' in body and 'persistedQuery' in body['extensions']:
        query_hash = body['extensions']['persistedQuery']['sha256Hash']
        query = PERSISTED_QUERIES.get(query_hash)
        if not query:
            return jsonify({'errors': [{'message': 'PersistedQueryNotFound'}]}), 400
        body['query'] = query
    
    result = schema.execute(body['query'], variables=body.get('variables'))
    return jsonify(result)
```

## Layman's Explanation

**REST is a fixed-menu restaurant**: The chef decides exactly what's on each plate. You order the "Chicken Dinner" and get chicken, mashed potatoes, green beans, and gravy—whether you like green beans or not (over-fetching). If you also want a side salad, you place a second order (under-fetching).

**GraphQL is a build-your-own plate**: The menu lists every ingredient available. You specify: "I want chicken, extra mashed potatoes, no green beans, and a side salad." One order (one request), exactly what you want (no over-fetching), no second trip to the counter (no under-fetching).

The price: The kitchen (server) now has to handle custom orders (more complexity), and greedy customers could ask for "one of everything, twice" (query complexity attacks).

## Why Solution Architects Must Acquire This

- **Mobile-First Architecture**: Mobile apps with limited bandwidth and high latency benefit enormously from GraphQL's ability to fetch all needed data in one round-trip
- **API Gateway Consolidation**: GraphQL can serve as a unified API layer aggregating multiple REST services, gRPC endpoints, and databases behind a single schema
- **Developer Experience**: Strongly typed schema serves as self-documenting contract between frontend and backend teams. GraphQL introspection enables auto-complete, documentation generation, and type-safe code generation.
- **When NOT to use GraphQL**: Simple CRUD APIs (REST is simpler), file upload/download (REST multipart is better), server-to-server communication (gRPC is faster), caching-heavy workloads (GraphQL caching is complex)

## Summary

| Consideration | GraphQL Design Choice |
|--------------|----------------------|
| N+1 queries | DataLoader for batching |
| Query complexity | Depth limiting + complexity scoring |
| Security | Persisted queries in production |
| Caching | Client-side normalized cache (Apollo); CDN caching is hard |
| File upload | GraphQL multipart request spec |
| Real-time | Subscriptions over WebSocket |
| Schema evolution | Additive changes only; deprecate with @deprecated directive |

GraphQL is the right choice when clients have diverse data needs, bandwidth is constrained (mobile), and multiple backend services need to be unified behind a single API. REST remains better for simple CRUD, caching at CDN, and server-to-server communication. The architect must choose based on the client's data requirements, not technology trends.
