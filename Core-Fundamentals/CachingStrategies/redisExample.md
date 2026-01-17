# Distributed Redis Cache - Real-Life Example

---

## **Scenario: E-commerce with 3 Redis Clusters**

**Setup:**
- Cluster 1: User sessions (us-east-1)
- Cluster 2: Product catalog (us-west-2)
- Cluster 3: Shopping carts (eu-west-1)

---

## **Configuration (Python)**

```python
import redis

# Connect to 3 different Redis clusters
redis_sessions = redis.Redis(
    host='sessions.us-east-1.cache.amazonaws.com',
    port=6379,
    decode_responses=True
)

redis_products = redis.Redis(
    host='products.us-west-2.cache.amazonaws.com',
    port=6379,
    decode_responses=True
)

redis_carts = redis.Redis(
    host='carts.eu-west-1.cache.amazonaws.com',
    port=6379,
    decode_responses=True
)
```

---

## **1. User Session (Cluster 1 - Sessions)**

```python
import json

def create_session(user_id, user_data):
    session_id = generate_uuid()
    
    # Store in Sessions cluster
    redis_sessions.setex(
        f"session:{session_id}",
        1800,  # 30 minutes TTL
        json.dumps({'user_id': user_id, 'data': user_data})
    )
    return session_id

def get_session(session_id):
    data = redis_sessions.get(f"session:{session_id}")
    return json.loads(data) if data else None

# Usage
session_id = create_session(1000, {'name': 'John', 'role': 'customer'})
session = get_session(session_id)
```

---

## **2. Product Catalog (Cluster 2 - Products)**

```python
def cache_product(product_id, product_data):
    # Store in Products cluster
    redis_products.setex(
        f"product:{product_id}",
        3600,  # 1 hour TTL
        json.dumps(product_data)
    )

def get_product(product_id):
    # Try cache first
    cached = redis_products.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)
    
    # Cache miss - fetch from DB
    product = db.query("SELECT * FROM products WHERE id = ?", product_id)
    cache_product(product_id, product)
    return product

# Usage
product = get_product(500)  # First call: DB query
product = get_product(500)  # Second call: Redis cache (fast)
```

---

## **3. Shopping Cart (Cluster 3 - Carts)**

```python
def add_to_cart(user_id, product_id, quantity):
    cart_key = f"cart:{user_id}"
    
    # Get current cart from Carts cluster
    cart = redis_carts.hgetall(cart_key)
    
    # Add/update item (Redis Hash)
    redis_carts.hset(cart_key, product_id, quantity)
    
    # Extend TTL (24 hours)
    redis_carts.expire(cart_key, 86400)

def get_cart(user_id):
    cart_key = f"cart:{user_id}"
    return redis_carts.hgetall(cart_key)  # Returns dict

def remove_from_cart(user_id, product_id):
    redis_carts.hdel(f"cart:{user_id}", product_id)

# Usage
add_to_cart(user_id=1000, product_id=500, quantity=2)
add_to_cart(user_id=1000, product_id=501, quantity=1)
cart = get_cart(1000)  # {500: 2, 501: 1}
```

---

## **Complete Checkout Flow (Using All 3 Clusters)**

```python
def checkout(session_id):
    # 1. Get user from Sessions cluster
    session = redis_sessions.get(f"session:{session_id}")
    if not session:
        raise Exception("Invalid session")
    
    user_id = json.loads(session)['user_id']
    
    # 2. Get cart from Carts cluster
    cart = redis_carts.hgetall(f"cart:{user_id}")
    if not cart:
        raise Exception("Empty cart")
    
    # 3. Get product details from Products cluster (batch)
    product_ids = cart.keys()
    products = []
    
    for product_id in product_ids:
        product = redis_products.get(f"product:{product_id}")
        if product:
            products.append(json.loads(product))
        else:
            # Cache miss - fetch from DB
            product = db.query("SELECT * FROM products WHERE id = ?", product_id)
            redis_products.setex(f"product:{product_id}", 3600, json.dumps(product))
            products.append(product)
    
    # 4. Process order
    order = {
        'user_id': user_id,
        'items': [
            {'product': p, 'quantity': cart[str(p['id'])]} 
            for p in products
        ],
        'total': sum(p['price'] * int(cart[str(p['id'])]) for p in products)
    }
    
    # 5. Clear cart after checkout
    redis_carts.delete(f"cart:{user_id}")
    
    return order

# Usage
order = checkout(session_id)
```

---

## **Redis Sentinel (High Availability)**

### **Setup with Automatic Failover**

```python
from redis.sentinel import Sentinel

# Configure Sentinel for automatic failover
sentinel = Sentinel([
    ('sentinel1.example.com', 26379),
    ('sentinel2.example.com', 26379),
    ('sentinel3.example.com', 26379)
], socket_timeout=0.1)

# Get master connection (auto-failover if master dies)
redis_sessions = sentinel.master_for(
    'sessions-master',
    socket_timeout=0.1,
    decode_responses=True
)

# Get slave connection (for reads)
redis_sessions_read = sentinel.slave_for(
    'sessions-master',
    socket_timeout=0.1,
    decode_responses=True
)

# Write to master
redis_sessions.set('key', 'value')

# Read from slave (better performance)
value = redis_sessions_read.get('key')
```

---

## **Redis Cluster (Sharding)**

### **Automatic Data Distribution Across Nodes**

```python
from rediscluster import RedisCluster

# Connect to Redis Cluster (6 nodes: 3 masters + 3 replicas)
startup_nodes = [
    {"host": "node1.example.com", "port": 7000},
    {"host": "node2.example.com", "port": 7001},
    {"host": "node3.example.com", "port": 7002}
]

redis_cluster = RedisCluster(
    startup_nodes=startup_nodes,
    decode_responses=True,
    skip_full_coverage_check=True
)

# Data automatically sharded across nodes
redis_cluster.set('user:1000', 'data1')  # Goes to node based on hash slot
redis_cluster.set('user:1001', 'data2')  # Might go to different node
redis_cluster.set('user:1002', 'data3')  # Automatic distribution

# Reads automatically routed to correct node
user = redis_cluster.get('user:1000')
```

---

## **Visualization**

```
┌─────────────────────────────────────────────────────┐
│                   Load Balancer                      │
└─────────────────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌─────────┐     ┌─────────┐     ┌─────────┐
   │ App     │     │ App     │     │ App     │
   │ Server  │     │ Server  │     │ Server  │
   │ (US)    │     │ (US)    │     │ (EU)    │
   └─────────┘     └─────────┘     └─────────┘
         │               │               │
   ┌─────┼───────────────┼───────────────┼─────┐
   │     │               │               │     │
   ▼     ▼               ▼               ▼     ▼
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Redis    │      │ Redis    │      │ Redis    │
│ Sessions │      │ Products │      │ Carts    │
│ (us-e-1) │      │ (us-w-2) │      │ (eu-w-1) │
└──────────┘      └──────────┘      └──────────┘
```

---

## **Key Concepts**

**1. Geographic Distribution:**
- Sessions in US East (close to auth server)
- Products in US West (close to warehouse)
- Carts in EU (GDPR compliance)

**2. Data Separation:**
- Each cluster optimized for specific data type
- Independent scaling
- Failure isolation

**3. High Availability:**
- Sentinel monitors masters
- Auto-failover if master dies
- Slaves serve read traffic

**4. Sharding:**
- Redis Cluster distributes data automatically
- 16,384 hash slots
- Keys hashed to slots, slots assigned to nodes

---

## **Production Best Practices**

```python
# Connection pooling
redis_sessions = redis.Redis(
    connection_pool=redis.ConnectionPool(
        host='sessions.cache.amazonaws.com',
        port=6379,
        max_connections=50,  # Reuse connections
        decode_responses=True
    )
)

# Retry logic
from redis.retry import Retry
from redis.backoff import ExponentialBackoff

retry = Retry(ExponentialBackoff(), 3)
redis_sessions = redis.Redis(retry=retry, retry_on_timeout=True)

# Error handling
def safe_cache_get(key):
    try:
        return redis_sessions.get(key)
    except redis.ConnectionError:
        # Cache down - fallback to DB
        return db.query(key)
```

---

**This setup handles millions of requests with sub-millisecond latency across multiple regions!**