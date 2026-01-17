# Cache Eviction Strategies

## **Cache is limited. Something must go.**

---

## **1. LRU (Least Recently Used)**

### What It Does

Removes data **not accessed recently**

### Example (Python)

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity=3):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key):
        if key in self.cache:
            self.cache.move_to_end(key)  # Mark as recently used
            return self.cache[key]
        return None

    def set(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value

        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # Remove least recently used

# Usage
cache = LRUCache(capacity=3)
cache.set('A', 1)  # Cache: [A]
cache.set('B', 2)  # Cache: [A, B]
cache.set('C', 3)  # Cache: [A, B, C]
cache.get('A')     # Cache: [B, C, A] (A moved to end)
cache.set('D', 4)  # Cache: [C, A, D] (B evicted - least recently used)
```

**Use Case**: General-purpose caching, browser cache, OS page cache

---

## **2. LFU (Least Frequently Used)**

### What It Does

Removes data **used least often**

### Example (Python)

```python
class LFUCache:
    def __init__(self, capacity=3):
        self.cache = {}
        self.freq = {}  # Track access frequency
        self.capacity = capacity

    def get(self, key):
        if key in self.cache:
            self.freq[key] += 1  # Increment frequency
            return self.cache[key]
        return None

    def set(self, key, value):
        if key in self.cache:
            self.freq[key] += 1
        else:
            if len(self.cache) >= self.capacity:
                # Remove least frequently used
                lfu_key = min(self.freq, key=self.freq.get)
                del self.cache[lfu_key]
                del self.freq[lfu_key]

            self.cache[key] = value
            self.freq[key] = 1

# Usage
cache = LFUCache(capacity=3)
cache.set('A', 1)  # freq: {A: 1}
cache.set('B', 2)  # freq: {A: 1, B: 1}
cache.set('C', 3)  # freq: {A: 1, B: 1, C: 1}
cache.get('A')     # freq: {A: 2, B: 1, C: 1}
cache.get('A')     # freq: {A: 3, B: 1, C: 1}
cache.set('D', 4)  # B or C evicted (both have freq=1)
```

**Use Case**: CDN caching popular content, video streaming

---

## **3. FIFO (First In, First Out)**

### What It Does

Removes **oldest entry** (like a queue)

### Example (Python)

```python
from collections import deque

class FIFOCache:
    def __init__(self, capacity=3):
        self.cache = {}
        self.order = deque()
        self.capacity = capacity

    def get(self, key):
        return self.cache.get(key)

    def set(self, key, value):
        if key not in self.cache:
            if len(self.cache) >= self.capacity:
                # Remove first item added
                oldest = self.order.popleft()
                del self.cache[oldest]

            self.order.append(key)

        self.cache[key] = value

# Usage
cache = FIFOCache(capacity=3)
cache.set('A', 1)  # Queue: [A]
cache.set('B', 2)  # Queue: [A, B]
cache.set('C', 3)  # Queue: [A, B, C]
cache.set('D', 4)  # Queue: [B, C, D] (A evicted - first in)
```

**Use Case**: Simple message queues, basic logging buffers

---

## **4. TTL (Time To Live)**

### What It Does

Data **expires after time**

### Example (Python)

```python
import time

class TTLCache:
    def __init__(self):
        self.cache = {}
        self.expiry = {}

    def get(self, key):
        if key in self.cache:
            # Check if expired
            if time.time() < self.expiry[key]:
                return self.cache[key]
            else:
                # Expired - remove
                del self.cache[key]
                del self.expiry[key]
        return None

    def set(self, key, value, ttl=60):
        self.cache[key] = value
        self.expiry[key] = time.time() + ttl

# Usage
cache = TTLCache()
cache.set('session', 'abc123', ttl=5)  # Expires in 5 seconds

print(cache.get('session'))  # 'abc123'
time.sleep(6)
print(cache.get('session'))  # None (expired)
```

**Use Case**: Sessions, API tokens, temporary data

---

## **5. TTL + LRU (Best Combo)**

### What It Does

**Expires old data** + **Evicts least recently used**

### Example (Redis - Real World)

```python
import redis

cache = redis.Redis(host='localhost', port=6379)

# Set with TTL
cache.setex('user:1000', 3600, 'John')  # Expires in 1 hour

# Redis uses LRU when memory is full
# Configure in redis.conf:
# maxmemory 2gb
# maxmemory-policy allkeys-lru
```

### Example (Python Custom)

```python
from collections import OrderedDict
import time

class TTLLRUCache:
    def __init__(self, capacity=100):
        self.cache = OrderedDict()
        self.expiry = {}
        self.capacity = capacity

    def get(self, key):
        if key in self.cache:
            # Check TTL
            if time.time() < self.expiry[key]:
                self.cache.move_to_end(key)  # LRU: mark as recent
                return self.cache[key]
            else:
                # Expired
                del self.cache[key]
                del self.expiry[key]
        return None

    def set(self, key, value, ttl=3600):
        if key in self.cache:
            self.cache.move_to_end(key)
        else:
            if len(self.cache) >= self.capacity:
                # Evict LRU
                oldest = next(iter(self.cache))
                del self.cache[oldest]
                del self.expiry[oldest]

        self.cache[key] = value
        self.expiry[key] = time.time() + ttl

# Usage
cache = TTLLRUCache(capacity=3)
cache.set('A', 1, ttl=10)
cache.set('B', 2, ttl=10)
cache.set('C', 3, ttl=10)
cache.get('A')     # A marked as recently used
cache.set('D', 4)  # B evicted (least recently used)
```

**Use Case**: Web applications, API caching, most production systems

---

## **Comparison Table**

| Policy      | Evicts            | Good For            | Bad For         |
| ----------- | ----------------- | ------------------- | --------------- |
| **LRU**     | Not recently used | General use         | Scan patterns   |
| **LFU**     | Rarely used       | Popular content     | New hot items   |
| **FIFO**    | Oldest            | Simple queues       | Everything else |
| **TTL**     | Expired           | Time-sensitive data | Long-lived data |
| **TTL+LRU** | Expired + LRU     | Production systems  | ✅ Best combo   |

---

## **Real-World Configurations**

### Redis

```bash
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru  # Options: lru, lfu, random, ttl
```

### Memcached

```bash
# Memcached always uses LRU
memcached -m 1024  # 1GB memory, auto-evicts with LRU
```

### Browser

```javascript
// TTL via Cache-Control header
res.setHeader("Cache-Control", "max-age=3600"); // TTL: 1 hour
```

---

## **Architect Tip**

**TTL + LRU is the most practical combo:**

- TTL removes stale data automatically
- LRU removes unpopular data when space is needed
- Used by Redis, Memcached, CDNs, browsers

**Default choice**: Always start with TTL + LRU unless you have a specific reason not to.
