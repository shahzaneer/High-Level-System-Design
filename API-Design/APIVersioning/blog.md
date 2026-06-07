# API Versioning

## Introduction
API Versioning is the practice of managing changes to an API over time without breaking existing consumers. Every API that has consumers will need to evolve—new fields, changed behavior, deprecated endpoints. Without a versioning strategy, every change risks breaking mobile apps (users on old versions), partner integrations (unchanged for months), and internal services (different deployment schedules).

Versioning is fundamentally a compatibility problem: how do you change a contract while honoring the old contract for consumers who haven't upgraded? The answer is not "just don't break things"—eventually, breaking changes are necessary. The strategy determines whether those breaking changes cause controlled migration or uncontrolled outages.

## Definition

**API Versioning** is the set of strategies and practices for evolving an API's interface while maintaining backward compatibility for existing consumers during a migration window. It answers: how are versions identified, how long are old versions supported, and how are consumers migrated.

**Key terms**:
- **Backward compatible**: Old clients work with new server (additive changes)
- **Forward compatible**: New clients work with old server (rarely achievable)
- **Deprecation**: Announcing that a feature will be removed in a future version
- **Sunset**: The date after which an old version stops working
- **Migration window**: Time between deprecation announcement and sunset

## Concept Explanation

### Versioning Strategies

#### 1. URL Versioning (Most Common)

```python
# Explicit version in URL path
@app.route('/v1/orders')
@app.route('/v2/orders')

# Pros: Simple, visible, easy to route, easy to deprecate whole version
# Cons: URL changes on upgrade, "v1/v2" in URLs forever
```

#### 2. Header Versioning (Content Negotiation)

```python
# Version in Accept header
@app.route('/orders')
def get_orders():
    accept_header = request.headers.get('Accept', '')
    
    if 'application/vnd.company.orders.v2+json' in accept_header:
        return jsonify(orders_v2_response())
    else:
        return jsonify(orders_v1_response())

# Pros: Same URL, clean, uses HTTP content negotiation
# Cons: Harder to test (curl needs custom headers), caching at CDN is harder
```

#### 3. Query Parameter Versioning

```python
@app.route('/orders')
def get_orders():
    version = request.args.get('version', '1')
    if version == '2':
        return jsonify(orders_v2_response())
    return jsonify(orders_v1_response())

# Pros: Easy to test in browser
# Cons: Pollutes query parameters, breaks caching
```

### Backward-Compatible Changes (Safe)

```python
# These changes DON'T require a new version:

# ✓ Add new fields to response
# v1: {"id": 1, "name": "Alice"}
# v2: {"id": 1, "name": "Alice", "email": "alice@example.com"}

# ✓ Add new endpoints
# v1: GET /orders
# v2: GET /orders + GET /orders/search

# ✓ Add optional request parameters
# v1: GET /orders
# v2: GET /orders?status=pending (old clients ignore this, works as before)

# ✓ Relax validation (accept more values)
# v1: status must be "PENDING" or "COMPLETE"
# v2: status can be "PENDING", "CONFIRMED", "SHIPPED", "COMPLETE"

# ✓ Add new HTTP methods to existing resources
# v1: GET /orders/123
# v2: GET /orders/123 + PATCH /orders/123
```

### Breaking Changes (Require New Version)

```python
# These changes REQUIRE a new version:

# ✗ Remove or rename fields
# v1: {"id": 1, "name": "Alice"}
# v2: {"customer_id": 1, "full_name": "Alice"}  -- BREAKS v1 clients

# ✗ Change field types
# v1: {"total": 99.99}       (float)
# v2: {"total": "99.99 USD"} (string)  -- BREAKS v1 parsing

# ✗ Change field meaning
# v1: {"price": 99.99}     (includes tax)
# v2: {"price": 89.99}     (excludes tax)  -- BREAKS v1 logic

# ✗ Remove endpoints
# v1: GET /orders/legacy-report  -- removed in v2

# ✗ Add required request parameters
# v1: POST /orders {"items": [...]}
# v2: POST /orders {"items": [...], "currency": "USD"}  -- BREAKS v1 clients

# ✗ Tighten validation (reject previously valid values)
# v1: name can be any string
# v2: name must match regex ^[a-zA-Z ]+$  -- BREAKS v1 clients with special chars
```

### Deprecation Lifecycle

```python
class DeprecationManager:
    """
    Formal deprecation process prevents breaking consumers without warning.
    """
    
    def deprecate_endpoint(self, version, endpoint, sunset_date, replacement=None):
        # Phase 1: Announce deprecation
        # - Add Deprecation header to responses
        # - Add Sunset header with date
        # - Log warnings for consumers
        # - Email registered API consumers
        
        self._add_response_headers(version, endpoint, {
            'Deprecation': 'true',
            'Sunset': sunset_date.isoformat(),
            'Link': f'<{replacement}>; rel="successor-version"' if replacement else None
        })
        
        # Phase 2: Monitor usage
        usage = self._track_usage(version, endpoint)
        self._notify_heavy_users(usage)
        
        # Phase 3: Enforce (after sunset_date)
        if datetime.now() > sunset_date:
            self._disable_endpoint(version, endpoint, return_410=True)
    
    def _add_response_headers(self, version, endpoint, headers):
        @app.after_request
        def add_deprecation_headers(response):
            if request.path.startswith(f'/{version}{endpoint}'):
                for key, value in headers.items():
                    if value:
                        response.headers[key] = value
            return response
```

### HTTP Headers for Versioning and Deprecation

```http
HTTP/1.1 200 OK
Content-Type: application/json
API-Version: 2024-06-15
Deprecation: true
Sunset: Sat, 31 Dec 2024 23:59:59 GMT
Link: <https://api.example.com/v2/orders>; rel="successor-version"
```

### Stripe-Style Date Versioning

```python
# Stripe uses dates instead of numbers: API-Version: 2024-06-15
# This enables very granular versioning without v1/v2/v3 proliferation

@app.route('/orders')
def get_orders():
    api_version = request.headers.get('API-Version', '2024-01-01')
    
    if api_version >= '2024-06-15':
        return orders_response_after_june_2024()
    elif api_version >= '2024-03-01':
        return orders_response_after_march_2024()
    else:
        return orders_response_original()
```

## Summary

| Strategy | Best For | Caching | Complexity |
|----------|----------|---------|------------|
| URL (/v1/) | Public APIs, simple routing | Easy (different URL) | Low |
| Header (Accept) | REST purists, same URL | Hard (Vary: Accept) | Medium |
| Query param (?v=2) | Internal, quick testing | Easy | Low |
| Date-based (Stripe) | Granular evolution | Medium | Medium |

API versioning is not a one-time decision—it's a lifecycle. Announce deprecation, monitor usage, support old versions during migration, and sunset old versions on schedule. The architect's responsibility is to minimize breaking changes through careful API design (additive-only by default) and to manage the inevitable breaking changes through clear versioning, communication, and migration support.
