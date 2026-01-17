# Database Sharding and Partitioning
---

## **Introduction**

Database sharding and partitioning are critical strategies for scaling databases beyond the limits of a single server. As applications grow from thousands to millions of users, a single database becomes a bottleneck—it can't handle the volume of reads, writes, or storage. 

- The fundamental principle is **divide and conquer**: split your data across multiple databases or tables so that no single instance is overwhelmed.

The concept emerged in the early 2000s when companies like Google, Amazon, and eBay hit the limits of vertical scaling (bigger servers). They pioneered horizontal scaling—splitting data across many commodity servers instead of buying increasingly expensive mainframes. Today, sharding powers the world's largest applications: Facebook (billions of users), Amazon (millions of products), Uber (millions of rides per day).

**The Core Problem**: A single PostgreSQL or MySQL server can handle ~10,000-50,000 queries per second. Beyond that, you need to distribute the load. Sharding and partitioning are the solutions, but they introduce complexity: distributed transactions, cross-shard queries, rebalancing, and consistency challenges.

Understanding these concepts is essential for Solution Architects because choosing the wrong strategy can lead to:
- **Hotspots**: Some shards overloaded while others sit idle
- **Cross-shard queries**: Slow, complex queries across multiple databases
- **Data migration nightmares**: Moving data between shards as the system grows
- **Operational complexity**: Managing hundreds of database instances

---

## **Definition**

### **Partitioning**
**Partitioning** is splitting a large table into smaller, more manageable pieces called **partitions**, all within a **single database instance**. Each partition holds a subset of the data based on a partition key (e.g., date ranges, geographic regions, ID ranges). Queries can target specific partitions instead of scanning the entire table.

### **Sharding**
**Sharding** (also called **horizontal partitioning**) is splitting data across **multiple database instances** (servers). Each shard is an independent database holding a subset of the data. The application determines which shard to query based on a **shard key**.

**Key Difference**:
- **Partitioning**: Multiple pieces, one database instance → Improves query performance
- **Sharding**: Multiple pieces, multiple database instances → Improves scalability and throughput

---

## **Concept Explanation**

### **Partitioning Deep Dive**

Imagine a table with 1 billion rows. Scanning this table takes minutes. Partitioning splits it into smaller chunks:

**Example**: Orders table with 1 billion orders
```
orders_2020 (100M rows)
orders_2021 (250M rows)
orders_2022 (300M rows)
orders_2023 (250M rows)
orders_2024 (100M rows)
```

Query: `SELECT * FROM orders WHERE order_date = '2024-01-15'`
- **Without partitioning**: Scans 1 billion rows
- **With partitioning**: Scans only `orders_2024` (100M rows) → 10x faster

#### **Types of Partitioning**

**1. Range Partitioning**
Data divided by ranges of values (dates, IDs, prices).

```sql
-- PostgreSQL example
CREATE TABLE orders (
    order_id BIGINT,
    order_date DATE,
    amount DECIMAL
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**Use Case**: Time-series data (logs, orders, events), historical data archival

**2. List Partitioning**
Data divided by specific lists of values (countries, categories, statuses).

```sql
CREATE TABLE users (
    user_id BIGINT,
    country VARCHAR(2),
    name VARCHAR(100)
) PARTITION BY LIST (country);

CREATE TABLE users_us PARTITION OF users
    FOR VALUES IN ('US');

CREATE TABLE users_eu PARTITION OF users
    FOR VALUES IN ('UK', 'DE', 'FR', 'ES');

CREATE TABLE users_asia PARTITION OF users
    FOR VALUES IN ('CN', 'JP', 'IN');
```

**Use Case**: Geographic segmentation, multi-tenant applications, category-based data

**3. Hash Partitioning**
Data divided by hash function on a key. Ensures even distribution.

```sql
CREATE TABLE events (
    event_id BIGINT,
    user_id BIGINT,
    event_type VARCHAR(50)
) PARTITION BY HASH (user_id);

CREATE TABLE events_p0 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE events_p1 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE events_p2 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE events_p3 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

**How it works**: `hash(user_id) % 4` determines partition
- user_id=1000 → hash=5432 → 5432%4=0 → events_p0
- user_id=1001 → hash=7891 → 7891%4=3 → events_p3

**Use Case**: Even data distribution, no natural partitioning key

**4. Composite Partitioning**
Combination of partitioning strategies.

```sql
-- Range-Hash: Partition by date, then hash within each date partition
CREATE TABLE metrics (
    metric_id BIGINT,
    timestamp TIMESTAMP,
    user_id BIGINT,
    value FLOAT
) PARTITION BY RANGE (timestamp);

CREATE TABLE metrics_2024_01 PARTITION OF metrics
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')
    PARTITION BY HASH (user_id);

CREATE TABLE metrics_2024_01_p0 PARTITION OF metrics_2024_01
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- ... more sub-partitions
```

**Use Case**: Very large datasets requiring multiple levels of organization

---

### **Sharding Deep Dive**

Sharding splits data across multiple **independent database instances**. Each shard is a full database with its own CPU, memory, disk, and network.

**Example**: Users table with 100 million users
```
Shard 1 (DB Server 1): users with ID 1-25M
Shard 2 (DB Server 2): users with ID 25M-50M
Shard 3 (DB Server 3): users with ID 50M-75M
Shard 4 (DB Server 4): users with ID 75M-100M
```

Query: `SELECT * FROM users WHERE user_id = 30000000`
- Application calculates: 30M is in Shard 2
- Query sent only to DB Server 2

#### **Sharding Strategies**

**1. Range-Based Sharding**
Data divided by ranges of shard key values.

```python
def get_shard(user_id):
    if user_id < 25_000_000:
        return db_shard1
    elif user_id < 50_000_000:
        return db_shard2
    elif user_id < 75_000_000:
        return db_shard3
    else:
        return db_shard4

# Usage
user_id = 30_000_000
shard = get_shard(user_id)
user = shard.query("SELECT * FROM users WHERE user_id = ?", user_id)
```

**Pros**:
- Simple to understand and implement
- Range queries easy (get all users 40M-45M)
- Adding new ranges straightforward

**Cons**:
- **Uneven distribution**: Early users might be more active (shard 1 overloaded)
- **Hotspots**: New users all go to last shard
- **Rebalancing complex**: Moving ranges between shards is difficult

**Use Case**: Time-series data where old data is cold (e.g., orders by date)

---

**2. Hash-Based Sharding**
Data divided by hash function on shard key.

```python
def get_shard(user_id):
    shard_count = 4
    shard_id = hash(user_id) % shard_count
    
    shard_map = {
        0: db_shard1,
        1: db_shard2,
        2: db_shard3,
        3: db_shard4
    }
    return shard_map[shard_id]

# Usage
user_id = 30_000_000
shard = get_shard(user_id)  # hash(30M) % 4 = shard_id
user = shard.query("SELECT * FROM users WHERE user_id = ?", user_id)
```

**Pros**:
- **Uniform distribution**: Data evenly spread across shards
- **No hotspots**: New data distributed evenly
- **Simple logic**: Hash function is deterministic

**Cons**:
- **Range queries impossible**: Can't get "users 40M-45M" without querying all shards
- **Rebalancing nightmare**: Adding/removing shards changes all hash mappings
- **Cross-shard queries complex**: Related data scattered across shards

**Use Case**: Write-heavy workloads, even distribution critical, no range queries

---

**3. Geographic/Location-Based Sharding**
Data divided by geographic location.

```python
shard_map = {
    'US': db_us_shard,
    'EU': db_eu_shard,
    'ASIA': db_asia_shard
}

def get_shard(country_code):
    if country_code in ['US', 'CA', 'MX']:
        return shard_map['US']
    elif country_code in ['UK', 'DE', 'FR', 'ES', 'IT']:
        return shard_map['EU']
    else:
        return shard_map['ASIA']

# Usage
user_country = 'UK'
shard = get_shard(user_country)
users = shard.query("SELECT * FROM users WHERE country = ?", user_country)
```

**Pros**:
- **Low latency**: Data close to users
- **Regulatory compliance**: EU data stays in EU (GDPR)
- **Natural boundaries**: Geographic regions are stable
- **Disaster recovery**: Region failure doesn't affect other regions

**Cons**:
- **Uneven distribution**: US might have 10x more users than other regions
- **Cross-region queries**: Global analytics require querying all shards
- **Migration complexity**: Users moving regions

**Use Case**: Global applications, regulatory requirements, latency-sensitive apps

---

**4. Consistent Hashing** (Advanced)
Minimizes data movement when adding/removing shards.

```python
import hashlib

class ConsistentHash:
    def __init__(self, nodes, replicas=150):
        self.replicas = replicas
        self.ring = {}
        self.sorted_keys = []
        
        for node in nodes:
            self.add_node(node)
    
    def add_node(self, node):
        # Create virtual nodes (replicas) for even distribution
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            self.ring[key] = node
            self.sorted_keys.append(key)
        self.sorted_keys.sort()
    
    def remove_node(self, node):
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            del self.ring[key]
            self.sorted_keys.remove(key)
    
    def get_node(self, key):
        if not self.ring:
            return None
        
        hash_key = self._hash(key)
        
        # Find first node clockwise on ring
        for ring_key in self.sorted_keys:
            if hash_key <= ring_key:
                return self.ring[ring_key]
        
        # Wrap around to first node
        return self.ring[self.sorted_keys[0]]
    
    def _hash(self, key):
        return int(hashlib.md5(str(key).encode()).hexdigest(), 16)

# Usage
shards = ['shard1', 'shard2', 'shard3', 'shard4']
ch = ConsistentHash(shards)

user_id = 30_000_000
shard = ch.get_node(user_id)
print(f"User {user_id} -> {shard}")

# Add new shard (only ~25% of keys remapped instead of 100%)
ch.add_node('shard5')
```

**Pros**:
- **Minimal rebalancing**: Adding shard only moves K/N keys (K=total keys, N=shards)
- **Elastic scaling**: Easy to add/remove shards
- **Even distribution**: Virtual nodes prevent hotspots

**Cons**:
- **Complex implementation**: Harder to understand and debug
- **Still no range queries**: Hash-based limitation remains

**Use Case**: Dynamic scaling environments, cloud-native applications, microservices

---

**5. Directory-Based Sharding** (Lookup Table)
Maintains a lookup table mapping keys to shards.

```python
# Lookup table (could be in Redis, database, or service)
shard_directory = {
    'user_1': 'shard1',
    'user_2': 'shard1',
    'user_3': 'shard2',
    'user_1000000': 'shard3',
    # ... millions of entries
}

def get_shard(user_id):
    shard_name = shard_directory.get(f'user_{user_id}')
    return db_connections[shard_name]

# Usage
user_id = 1000000
shard = get_shard(user_id)
user = shard.query("SELECT * FROM users WHERE user_id = ?", user_id)
```

**Pros**:
- **Flexible**: Can rebalance by updating lookup table
- **No algorithm needed**: Simple lookup
- **Custom logic**: VIP users on premium shards, etc.

**Cons**:
- **Lookup table is bottleneck**: Extra query for every request
- **Storage overhead**: Millions of entries in lookup table
- **Single point of failure**: If lookup service down, everything breaks

**Use Case**: Small to medium datasets, need flexibility in shard assignment

---

## **Layman's Explanation**

### **Partitioning**
Imagine a massive library with 10 million books, all on one endless shelf. Finding a book takes forever. So you reorganize:
- **Floor 1**: Books published 2000-2005
- **Floor 2**: Books published 2006-2010
- **Floor 3**: Books published 2011-2015
- **Floor 4**: Books published 2016-2024

Now when someone asks for a 2020 book, you go straight to Floor 4. Much faster! But it's still ONE library (one database).

### **Sharding**
Now imagine the library gets so big that even with floors, it can't handle all the visitors. So you build **4 separate library buildings**:
- **Library Building 1**: Books starting with A-F
- **Library Building 2**: Books starting with G-L
- **Library Building 3**: Books starting with M-R
- **Library Building 4**: Books starting with S-Z

Each building has its own staff, shelves, and systems. Looking for "Harry Potter"? Go to Building 2 (H). This is sharding—multiple independent databases.

---

## **Why Solution Architects Must Acquire This**

### **Business Impact**

**1. Scalability Requirements**
- **Scenario**: E-commerce startup grows from 10K to 10M users
- **Without sharding**: Database crashes at 1M users, business halts
- **With sharding**: Seamlessly scale to billions by adding shards
- **Cost**: 1 massive server ($50K/month) vs 10 commodity servers ($5K/month each)

**2. Performance SLAs**
- **Scenario**: Finance app requires <100ms response time for 99.9% of requests
- **Single database**: 500ms average at peak load → SLA violated
- **Sharded database**: 50ms average (each shard handles 1/10th the load)

**3. Regulatory Compliance**
- **Scenario**: Healthcare app must keep EU patient data in EU
- **Geographic sharding**: EU data in Frankfurt shard, US data in Virginia shard
- **Compliance**: GDPR, HIPAA requirements automatically met

**4. Disaster Recovery**
- **Scenario**: One datacenter catches fire
- **Single database**: Total outage, hours of downtime
- **Geo-sharded**: 3 other regions continue operating, 75% capacity maintained


**Common Mistakes to Avoid**:
1. **Premature sharding**: Over-engineering for scale you don't have yet
2. **Wrong shard key**: Choosing key that creates hotspots (e.g., timestamp for writes)
3. **Cross-shard JOINs**: Designing schema requiring joins across shards
4. **No rebalancing plan**: Starting with 4 shards, no plan for 40
5. **Ignoring monitoring**: Not tracking per-shard metrics


