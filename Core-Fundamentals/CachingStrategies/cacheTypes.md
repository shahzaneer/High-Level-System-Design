# Where Caching Lives (Cache Layers)

---

## **1. Client-Side Cache**

### **Browser Cache**

**What It Does**: Browser stores files locally to avoid re-downloading.

**Example (Python Flask)**:
```python
@app.route('/logo.png')
def logo():
    response = send_file('logo.png')
    response.headers['Cache-Control'] = 'public, max-age=31536000'  # 1 year
    return response

@app.route('/api/user')
def user_api():
    response = jsonify(user_data)
    response.headers['Cache-Control'] = 'no-store'  # Don't cache
    return response
```

**Use Case**: Static files (images, CSS, JS) cached for 1 year. Dynamic data not cached.

---

### **Mobile App Cache**

**What It Does**: Apps store data locally to work offline.

**Example (JavaScript - React Native)**:
```javascript
async function getUser(userId) {
  const cacheKey = `user_${userId}`;
  
  // Try cache first
  const cached = await AsyncStorage.getItem(cacheKey);
  if (cached) {
    const item = JSON.parse(cached);
    if (Date.now() - item.timestamp < 3600000) { // 1 hour
      return item.data;
    }
  }
  
  // Fetch from API
  const response = await fetch(`/api/users/${userId}`);
  const user = await response.json();
  
  // Cache it
  await AsyncStorage.setItem(cacheKey, JSON.stringify({
    data: user,
    timestamp: Date.now()
  }));
  
  return user;
}
```

**Use Case**: Profile data cached locally, works offline.

---

### **CDN Edge Cache**

**What It Does**: Content cached worldwide, close to users.

**Example (Node.js)**:
```javascript
app.get('/product/:id', (req, res) => {
  const product = getProduct(req.params.id);
  
  // Tell CDN to cache for 1 hour
  res.set('Cache-Control', 'public, max-age=3600');
  res.json(product);
});

app.get('/api/cart', (req, res) => {
  // User-specific, don't cache at CDN
  res.set('Cache-Control', 'private, no-cache');
  res.json(cart);
});
```

**Use Case**: User in Tokyo gets product from Tokyo edge (10ms) instead of US origin (200ms).

---

## **2. Application Cache**

### **In-Memory Cache (Local)**

**What It Does**: Cache inside application server's memory.

**Example (Python)**:
```python
cache = {}

def get_user(user_id):
    # Check cache
    if user_id in cache:
        return cache[user_id]  # Instant
    
    # Query database
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # Cache it
    cache[user_id] = user
    return user
```

**Use Case**: Configuration, reference data. Fast (microseconds) but not shared across servers.

---

### **Redis (Distributed Cache)**

**What It Does**: Shared cache across all application servers.

**Example (Python)**:
```python
import redis
import json

redis_client = redis.Redis(host='localhost', port=6379)

def get_product(product_id):
    # Try cache
    cached = redis_client.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)
    
    # Query database
    product = db.query("SELECT * FROM products WHERE id = ?", product_id)
    
    # Cache for 1 hour
    redis_client.setex(f"product:{product_id}", 3600, json.dumps(product))
    return product
```

**Use Case**: Session data, shopping carts. All servers see same cache.

---

### **Memcached (Distributed Cache)**

**What It Does**: Simple, fast distributed cache.

**Example (Python)**:
```python
import memcache

mc = memcache.Client(['127.0.0.1:11211'])

def get_user(user_id):
    user = mc.get(f"user_{user_id}")
    if user:
        return user
    
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    mc.set(f"user_{user_id}", user, time=3600)  # 1 hour
    return user
```

**Use Case**: Similar to Redis but simpler. Good for basic key-value caching.

---

## **3. Database Cache**

### **Query Cache**

**What It Does**: Database caches query results.

**Example (MySQL)**:
```sql
-- First execution: 500ms (disk read)
SELECT * FROM products WHERE category = 'electronics';

-- Second execution: 5ms (cached)
SELECT * FROM products WHERE category = 'electronics';
```

**Use Case**: Frequently run queries cached automatically.

---

### **Buffer Pool**

**What It Does**: Database caches table/index data in RAM.

**Configuration**:
```ini
# MySQL - Allocate 70% of RAM
innodb_buffer_pool_size = 8G
```

**Use Case**: Hot data stays in RAM. 99% of reads from memory instead of disk.

---

## **4. CDN Cache**

### **CloudFront/Cloudflare**

**What It Does**: Global content delivery network.

**Example (Node.js)**:
```javascript
app.get('/images/:name', (req, res) => {
  // CDN caches images for 1 year
  res.set('Cache-Control', 'public, max-age=31536000, immutable');
  res.sendFile(req.params.name);
});

app.get('/products/:id', (req, res) => {
  // CDN caches for 5 minutes
  res.set('Cache-Control', 'public, max-age=300');
  res.json(product);
});
```

**Use Case**: Static assets, product pages. Served from 100+ edge locations worldwide.

---

## **Multi-Layer Example (E-commerce)**

**Real-World Flow**:

```javascript
app.get('/product/:id', async (req, res) => {
  const id = req.params.id;
  
  // Layer 1: Local memory (0.1ms)
  let product = localCache.get(id);
  if (product) return res.json(product);
  
  // Layer 2: Redis (2ms)
  product = await redis.get(`product:${id}`);
  if (product) {
    localCache.set(id, JSON.parse(product));
    return res.json(JSON.parse(product));
  }
  
  // Layer 3: Database (100ms)
  product = await db.query('SELECT * FROM products WHERE id = ?', [id]);
  
  // Populate caches
  redis.setex(`product:${id}`, 3600, JSON.stringify(product));
  localCache.set(id, product);
  
  res.json(product);
});
```

**Request Timeline**:
```
Request 1: Database → 100ms
Request 2-1000: Local cache → 0.1ms
After 60s: Redis → 2ms
After 1 hour: Database → 100ms
```

**With CDN**:
```
User 1 (NYC): 100ms (database)
User 2-1000 (NYC): 10ms (CDN edge in NYC)
User in London: 15ms (CDN edge in London)
```

---

## **Summary Table**

| Layer | Speed | Shared? | Use Case | TTL |
|-------|-------|---------|----------|-----|
| **Browser** | Instant | No | Static files | 1 year |
| **CDN** | 10-50ms | Yes (globally) | Images, CSS, JS | Hours-Years |
| **Local Memory** | 0.1ms | No | Config, hot data | Minutes |
| **Redis** | 1-5ms | Yes (cluster) | Sessions, carts | Hours |
| **Database** | 50-200ms | Yes | Query cache | Varies |

**Key Principle**: Cache closest to user for best performance. Use multiple layers for optimal results.