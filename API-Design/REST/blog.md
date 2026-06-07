# REST API Design

## Introduction
REST (Representational State Transfer) is the dominant architectural style for web APIs, introduced by Roy Fielding in his 2000 doctoral dissertation. REST is not a protocol—it's a set of architectural constraints that, when applied, produce APIs that are scalable, stateless, cacheable, and have a uniform interface. Today, the vast majority of public APIs are RESTful, making it the most important API design skill for solution architects.

Despite its ubiquity, REST is widely misunderstood. "RESTful" is often used to describe any HTTP API that returns JSON, but true REST involves HATEOAS, content negotiation, and resource-oriented design. The pragmatic middle ground—"REST-like" or "HTTP API"—follows REST principles without striving for academic purity. This is what most production APIs implement.

## Definition

**REST** defines six architectural constraints:
1. **Client-Server**: Separation of UI concerns from data storage concerns
2. **Stateless**: Each request contains all information needed; no server-side session state
3. **Cacheable**: Responses must explicitly define themselves as cacheable or not
4. **Uniform Interface**: Resources identified in requests, manipulated through representations
5. **Layered System**: Client cannot tell if it's connected directly to the end server
6. **Code on Demand** (optional): Server can extend client functionality (JavaScript)

## Concept Explanation

### Resource-Oriented Design

```
Resources, NOT actions:

BAD:  POST /createOrder
      GET /getOrder?id=123
      POST /updateOrderStatus

GOOD: POST   /orders          (Create order)
      GET    /orders/123      (Retrieve order)
      PATCH  /orders/123      (Update order)
      DELETE /orders/123      (Cancel order)

Key principle: URLs identify nouns (resources), HTTP methods identify verbs (actions)
```

### HTTP Methods and Semantics

| Method | Semantics | Idempotent | Safe | Example |
|--------|-----------|------------|------|---------|
| GET | Retrieve | Yes | Yes | `GET /orders/123` |
| POST | Create | No | No | `POST /orders` |
| PUT | Replace (full update) | Yes | No | `PUT /orders/123` |
| PATCH | Partial update | No | No | `PATCH /orders/123` |
| DELETE | Remove | Yes | No | `DELETE /orders/123` |
| HEAD | Like GET, no body | Yes | Yes | `HEAD /orders/123` |
| OPTIONS | Discover methods | Yes | Yes | `OPTIONS /orders` |

**Idempotent**: Calling it multiple times has the same effect as calling once.
**Safe**: Doesn't modify the resource.

### Resource Naming Conventions

```python
# REST resource design patterns
class OrderAPI:
    # Collection resource
    GET    /orders                  # List orders (with pagination, filtering)
    POST   /orders                  # Create order
    
    # Singleton resource
    GET    /orders/123              # Get specific order
    PUT    /orders/123              # Replace order (full update)
    PATCH  /orders/123              # Update order (partial)
    DELETE /orders/123              # Cancel order
    
    # Sub-collection
    GET    /orders/123/items        # List items for order 123
    POST   /orders/123/items        # Add item to order 123
    GET    /orders/123/items/5      # Get specific item
    
    # Actions that don't fit CRUD (use verb as sub-resource)
    POST   /orders/123/cancel       # Cancel order
    POST   /orders/123/fulfill      # Fulfill order
    POST   /orders/123/refund       # Refund order
    
    # Compound resources
    GET    /orders/123/status       # Current status
    
    # Search
    GET    /orders?status=pending&customer_id=42&sort=created_at:desc&page=2&limit=20
```

### Status Codes Used Correctly

```python
from flask import Flask, request, jsonify, make_response

@app.route('/orders', methods=['POST'])
def create_order():
    try:
        order_request = CreateOrderRequest(**request.json)
        order = order_service.create(order_request)
        return jsonify(order.to_dict()), 201  # Created
    except ValidationError:
        return jsonify({'error': 'validation_failed'}), 400  # Bad Request

@app.route('/orders/<order_id>', methods=['GET'])
def get_order(order_id):
    order = order_service.find(order_id)
    if not order:
        return jsonify({'error': 'not_found'}), 404
    return jsonify(order.to_dict()), 200  # OK

@app.route('/orders/<order_id>', methods=['PATCH'])
def update_order(order_id):
    order = order_service.find(order_id)
    if not order:
        return jsonify({'error': 'not_found'}), 404
    
    # Check if resource changed since client last fetched it (optimistic locking)
    if request.headers.get('If-Match') != order.version:
        return jsonify({'error': 'conflict'}), 409  # Conflict
    
    order_service.update(order_id, request.json)
    return jsonify(order.to_dict()), 200

@app.route('/orders/<order_id>/cancel', methods=['POST'])
def cancel_order(order_id):
    order = order_service.find(order_id)
    if order.status == 'delivered':
        return jsonify({'error': 'cannot_cancel_delivered'}), 422  # Unprocessable
    order_service.cancel(order_id)
    return jsonify(order.to_dict()), 200
```

### Pagination

```python
# Cursor-based (recommended for large datasets)
@app.route('/orders')
def list_orders():
    cursor = request.args.get('cursor')  # Opaque token
    limit = min(request.args.get('limit', 20, type=int), 100)
    
    orders, next_cursor = order_service.list(cursor=cursor, limit=limit)
    
    response = {
        'data': [o.to_dict() for o in orders],
        'pagination': {
            'cursor': next_cursor,
            'has_more': next_cursor is not None
        }
    }
    return jsonify(response)

# Offset-based (simpler, OK for small datasets)
@app.route('/orders')
def list_orders_offset():
    page = request.args.get('page', 1, type=int)
    per_page = min(request.args.get('per_page', 20, type=int), 100)
    
    orders, total = order_service.paginate(page=page, per_page=per_page)
    
    response = {
        'data': [o.to_dict() for o in orders],
        'meta': {
            'page': page,
            'per_page': per_page,
            'total': total,
            'pages': (total + per_page - 1) // per_page
        }
    }
    return jsonify(response)
```

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request parameters failed validation",
    "details": [
      {
        "field": "customer_id",
        "message": "Must be at least 1 character",
        "code": "min_length"
      },
      {
        "field": "total",
        "message": "Must be greater than 0",
        "code": "min_value"
      }
    ],
    "request_id": "req_abc123"
  }
}
```

### Versioning Strategies

```python
# URL versioning (most common)
@app.route('/v1/orders')
@app.route('/v2/orders')

# Header versioning
# Accept: application/vnd.company.orders.v2+json

# Query parameter versioning
# GET /orders?version=2
```

## Why Solution Architects Must Acquire This

- **API is the Contract**: REST APIs are the public interface of your system. Changing them breaks consumers. Design them carefully—they outlive individual implementations.
- **Backward Compatibility**: API changes must be additive (new fields, new endpoints). Never remove fields or change semantics without a new version. This discipline prevents breaking mobile apps, partner integrations, and internal services.
- **Performance Implications**: REST encourages statelessness (enables horizontal scaling) and caching (reduces load). The architectural style directly impacts system scalability.

## Summary

| Principle | Practice |
|-----------|----------|
| Resources over actions | Design URLs around nouns |
| HTTP methods as verbs | GET=read, POST=create, PUT=replace, PATCH=update, DELETE=remove |
| Stateless | No server-side sessions; auth via tokens |
| Hypermedia (HATEOAS) | Include links to related resources in responses |
| Content negotiation | Support JSON; consider gRPC/protobuf for internal services |
| Pagination | Cursor-based for scale; offset-based for simplicity |
| Idempotency | PUT, DELETE, GET are idempotent; POST is not |

REST remains the universal language of APIs. Even as GraphQL and gRPC grow, REST's simplicity, tooling ecosystem, and universal HTTP compatibility make it the default choice for public APIs. A well-designed REST API is intuitive, predictable, and durable—it survives framework changes, team changes, and decades of evolution.
