# Cache Strategies
---

## **1. Cache-Aside (Lazy Loading)**

### How It Works

1. App checks cache
2. If cache miss → read from DB
3. Store result in cache
4. Return response

### Example (Python)

```python
def get_user(user_id):
    # 1. Check cache
    user = cache.get(f"user:{user_id}")
    if user:
        return user  # Cache hit
    
    # 2. Cache miss - read from DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. Store in cache
    cache.set(f"user:{user_id}", user, ttl=3600)
    
    # 4. Return
    return user
```

### Pros

* Simple
* Cache failures don't break system
* Most common strategy

### Cons

* Cache miss penalty
* Stale data risk

### Use When

* Read-heavy systems
* Data changes infrequently

**This is the default choice for most systems.**

---

## **2. Write-Through Cache**

### How It Works

1. App writes data to cache
2. Cache writes data to DB
3. Reads always hit cache

### Example (Python)

```python
def update_user(user_id, data):
    # 1. Write to cache
    cache.set(f"user:{user_id}", data, ttl=3600)
    
    # 2. Write to DB (synchronous)
    db.update("UPDATE users SET ... WHERE id = ?", user_id, data)
    
    return data

def get_user(user_id):
    # 3. Always read from cache (pre-populated)
    return cache.get(f"user:{user_id}")
```

### Pros

* Strong consistency
* No stale reads

### Cons

* Slower writes
* Cache is on write path

### Use When

* Consistency is critical
* Read-after-write must be guaranteed

---

## **3. Write-Behind (Write-Back) Cache**

### How It Works

1. App writes data to cache
2. Cache asynchronously writes to DB later

### Example (Python)

```python
import queue
import threading

write_queue = queue.Queue()

def update_user(user_id, data):
    # 1. Write to cache (instant)
    cache.set(f"user:{user_id}", data, ttl=3600)
    
    # 2. Queue DB write (async)
    write_queue.put(('users', user_id, data))
    
    return data  # Return immediately

# Background worker
def db_writer():
    while True:
        table, id, data = write_queue.get()
        db.update(f"UPDATE {table} SET ... WHERE id = ?", id, data)
        write_queue.task_done()

# Start background thread
threading.Thread(target=db_writer, daemon=True).start()
```

### Pros

* Very fast writes
* Reduced DB load

### Cons

* Data loss risk on cache failure
* Eventual consistency

### Use When

* High write throughput
* Some data loss is acceptable

---

## **4. Read-Through Cache**

### How It Works

* Cache itself fetches data from DB on miss
* App never talks to DB directly

### Example (Python)

```python
class CacheLayer:
    def get(self, key):
        # Check cache
        data = self.cache.get(key)
        if data:
            return data
        
        # Cache auto-fetches from DB
        data = self.fetch_from_db(key)
        self.cache.set(key, data, ttl=3600)
        return data
    
    def fetch_from_db(self, key):
        entity_type, entity_id = key.split(':')
        return db.query(f"SELECT * FROM {entity_type}s WHERE id = ?", entity_id)

# App code (simple)
cache_layer = CacheLayer()

def get_user(user_id):
    # App doesn't touch DB directly
    return cache_layer.get(f"user:{user_id}")
```

### Pros

* Cleaner app logic
* Centralized caching logic

### Cons

* Cache becomes critical dependency

### Use When

* Managed caching layers
* Large distributed systems

---

## **5. Write-Around Cache**

### How It Works

* Writes go directly to DB (bypass cache)
* Cache populated only on reads

### Example (Python)

```python
def create_log(log_data):
    # Write only to DB (skip cache)
    db.insert("INSERT INTO logs ...", log_data)
    # No cache write

def get_logs(filter_params):
    cache_key = f"logs:{hash(filter_params)}"
    
    # Check cache
    logs = cache.get(cache_key)
    if logs:
        return logs
    
    # Query DB
    logs = db.query("SELECT * FROM logs WHERE ...", filter_params)
    
    # Cache query results
    cache.set(cache_key, logs, ttl=300)
    return logs
```

### Pros

* Prevents cache pollution
* Fast writes

### Cons

* First read is slow

### Use When

* Write-once, read-rarely data
* Logs, analytics

---

## **6. Refresh-Ahead (Predictive Refresh)**

### How It Works

* Cache proactively refreshes data before expiration
* Popular data never expires

### Example (Python)

```python
def get_product(product_id):
    cache_key = f"product:{product_id}"
    data, ttl_remaining = cache.get_with_ttl(cache_key)
    
    if data:
        # If TTL < 5 min and popular, refresh in background
        if ttl_remaining < 300 and is_popular(cache_key):
            threading.Thread(target=refresh_product, args=(product_id,)).start()
        return data
    
    # Cache miss
    product = db.query("SELECT * FROM products WHERE id = ?", product_id)
    cache.set(cache_key, product, ttl=3600)
    return product

def refresh_product(product_id):
    product = db.query("SELECT * FROM products WHERE id = ?", product_id)
    cache.set(f"product:{product_id}", product, ttl=3600)
```

### Pros

* No cache miss penalty for hot data
* Predictable performance

### Cons

* Complex
* May refresh unused data

### Use When

* Highly trafficked data
* Can't tolerate slow requests

---

## **Comparison Table**

| Strategy | Write Speed | Read Speed | Consistency | Complexity |
|----------|-------------|------------|-------------|------------|
| **Cache-Aside** | Fast | Slow on miss | Eventual | Low |
| **Write-Through** | Slow | Fast | Strong | Medium |
| **Write-Behind** | Very Fast | Fast | Eventual | High |
| **Read-Through** | Fast | Slow on miss | Eventual | Medium |
| **Write-Around** | Fast | Slow on miss | Eventual | Low |
| **Refresh-Ahead** | Fast | Very Fast | Eventual | High |

---

## **Quick Decision Guide**

**Choose Cache-Aside when**: Default choice, general-purpose

**Choose Write-Through when**: Banking, financial systems (consistency critical)

**Choose Write-Behind when**: Social media likes, view counts (high writes, loss acceptable)

**Choose Read-Through when**: Using ORM/framework with caching

**Choose Write-Around when**: Log systems, write-once data

**Choose Refresh-Ahead when**: Homepage, trending content (can't tolerate misses)