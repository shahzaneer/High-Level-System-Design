## **Partitioning Examples**

### **MySQL Partitioning**

```sql
-- Range Partitioning (Orders by year)
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT,
    user_id BIGINT,
    order_date DATE,
    amount DECIMAL(10,2),
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Query only scans relevant partition
SELECT * FROM orders
WHERE order_date BETWEEN '2023-01-01' AND '2023-12-31';
-- Only scans p2023 partition

-- Add new partition for 2025
ALTER TABLE orders ADD PARTITION (
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- Drop old partition (archive/delete old data)
ALTER TABLE orders DROP PARTITION p2020;
```

**Benefits**:

- Queries 10-100x faster (partition pruning)
- Easy to archive old data (drop partition)
- Parallel query execution per partition

---

### **PostgreSQL Partitioning**

```sql
-- List Partitioning (Users by region)
CREATE TABLE users (
    user_id BIGSERIAL,
    username VARCHAR(50),
    country VARCHAR(2),
    created_at TIMESTAMP
) PARTITION BY LIST (country);

CREATE TABLE users_north_america PARTITION OF users
    FOR VALUES IN ('US', 'CA', 'MX');

CREATE TABLE users_europe PARTITION OF users
    FOR VALUES IN ('UK', 'DE', 'FR', 'ES', 'IT');

CREATE TABLE users_asia PARTITION OF users
    FOR VALUES IN ('CN', 'JP', 'IN', 'KR');

CREATE TABLE users_other PARTITION OF users
    DEFAULT;  -- Catch-all partition

-- Insert automatically goes to correct partition
INSERT INTO users (username, country) VALUES ('john', 'US');
-- Data stored in users_north_america

-- Query only scans relevant partition
SELECT * FROM users WHERE country = 'DE';
-- Only scans users_europe partition
```

---

### **Hash Partitioning (Even Distribution)**

```sql
-- PostgreSQL Hash Partitioning
CREATE TABLE events (
    event_id BIGSERIAL,
    user_id BIGINT,
    event_type VARCHAR(50),
    created_at TIMESTAMP
) PARTITION BY HASH (user_id);

CREATE TABLE events_p0 PARTITION OF events
    FOR VALUES WITH (MODULUS 8, REMAINDER 0);

CREATE TABLE events_p1 PARTITION OF events
    FOR VALUES WITH (MODULUS 8, REMAINDER 1);

-- ... up to events_p7

-- Data automatically distributed evenly
INSERT INTO events (user_id, event_type) VALUES (12345, 'login');
-- hash(12345) % 8 = partition number
```

**When to use which**:

- **Range**: Time-series, sequential IDs, dates
- **List**: Geography, categories, status values
- **Hash**: Uniform distribution, no natural key

---

## **Sharding Examples**

### **Application-Level Sharding (Python)**

```python
import pymysql

# Configure shards
SHARDS = {
    'shard1': {'host': 'db1.example.com', 'database': 'app_shard1'},
    'shard2': {'host': 'db2.example.com', 'database': 'app_shard2'},
    'shard3': {'host': 'db3.example.com', 'database': 'app_shard3'},
    'shard4': {'host': 'db4.example.com', 'database': 'app_shard4'}
}

# Connection pool per shard
shard_connections = {}
for shard_name, config in SHARDS.items():
    shard_connections[shard_name] = pymysql.connect(
        host=config['host'],
        database=config['database'],
        user='app_user',
        password='password'
    )

# Shard routing logic
def get_shard_for_user(user_id):
    shard_count = len(SHARDS)
    shard_id = user_id % shard_count
    return f'shard{shard_id + 1}'

# CRUD operations
def get_user(user_id):
    shard_name = get_shard_for_user(user_id)
    conn = shard_connections[shard_name]

    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE user_id = %s", (user_id,))
    return cursor.fetchone()

def create_user(user_id, username, email):
    shard_name = get_shard_for_user(user_id)
    conn = shard_connections[shard_name]

    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO users (user_id, username, email) VALUES (%s, %s, %s)",
        (user_id, username, email)
    )
    conn.commit()

def get_all_users():
    # Cross-shard query - query all shards and combine
    all_users = []

    for shard_name, conn in shard_connections.items():
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users")
        all_users.extend(cursor.fetchall())

    return all_users

# Usage
create_user(1001, 'alice', 'alice@example.com')  # Goes to shard2 (1001 % 4 = 1)
create_user(2002, 'bob', 'bob@example.com')      # Goes to shard3 (2002 % 4 = 2)

user = get_user(1001)  # Queries only shard2
```

---

### **Vitess (MySQL Sharding - Production Grade)**

**What is Vitess**: Database clustering system for MySQL, used by YouTube, Slack, GitHub.

```python
# Vitess Python Client
from vitess import vtgate

# Connect to VTGate (routing layer)
conn = vtgate.connect('localhost:15991', 30)
cursor = conn.cursor()

# VTGate automatically routes to correct shard
cursor.execute(
    "INSERT INTO users (user_id, username) VALUES (:user_id, :username)",
    {'user_id': 1001, 'username': 'alice'}
)

# Query automatically routed
cursor.execute(
    "SELECT * FROM users WHERE user_id = :user_id",
    {'user_id': 1001}
)
user = cursor.fetchone()

# Cross-shard query (scatter-gather)
cursor.execute("SELECT COUNT(*) FROM users")
total_users = cursor.fetchone()
```

**Vitess Architecture**:

```
Application
    ↓
VTGate (routing layer)
    ↓
├── VTTablet (Shard 1) → MySQL Instance 1
├── VTTablet (Shard 2) → MySQL Instance 2
├── VTTablet (Shard 3) → MySQL Instance 3
└── VTTablet (Shard 4) → MySQL Instance 4
```

**Features**:

- Automatic shard routing
- Connection pooling
- Query rewriting
- Resharding (move data between shards)
- Horizontal scaling
- Backup and recovery

---

### **Citus (PostgreSQL Sharding)**

**What is Citus**: Distributed PostgreSQL extension, turns Postgres into distributed database.

```sql
-- On coordinator node, create distributed table
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP
);

-- Distribute table across worker nodes
SELECT create_distributed_table('users', 'user_id');

-- Citus automatically shards across workers
INSERT INTO users (username, email) VALUES ('alice', 'alice@example.com');
-- Data distributed based on hash(user_id)

-- Query automatically routed
SELECT * FROM users WHERE user_id = 1001;
-- Queries only relevant shard

-- Joins work if co-located
CREATE TABLE orders (
    order_id BIGSERIAL,
    user_id BIGINT,
    amount DECIMAL(10,2)
);

SELECT create_distributed_table('orders', 'user_id');
-- orders and users co-located by user_id

-- This works efficiently (same shard)
SELECT u.username, o.amount
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.user_id = 1001;
```

**Architecture**:

```
Coordinator Node (PostgreSQL + Citus)
    ↓
├── Worker Node 1 (PostgreSQL)
│   ├── users (shard 1)
│   └── orders (shard 1)
├── Worker Node 2 (PostgreSQL)
│   ├── users (shard 2)
│   └── orders (shard 2)
└── Worker Node 3 (PostgreSQL)
    ├── users (shard 3)
    └── orders (shard 3)
```

---

## **Sharding Challenges and Solutions**

### **Challenge 1: Cross-Shard Queries**

**Problem**: User 1001 on shard1, their orders on multiple shards.

```sql
-- This requires querying ALL shards
SELECT u.username, COUNT(o.order_id) as order_count
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.username;
```

**Solutions**:

**1. Denormalization**

```python
# Store user data with each order
orders = {
    'order_id': 5001,
    'user_id': 1001,
    'username': 'alice',  # Denormalized
    'amount': 99.99
}
# Now can query orders without joining users
```

**2. Application-Level Join**

```python
def get_user_orders(user_id):
    # Get user from user shard
    user_shard = get_shard_for_user(user_id)
    user = user_shard.query("SELECT * FROM users WHERE user_id = ?", user_id)

    # Get orders from order shard
    order_shard = get_shard_for_order(user_id)  # Shard by user_id
    orders = order_shard.query("SELECT * FROM orders WHERE user_id = ?", user_id)

    return {'user': user, 'orders': orders}
```

**3. Co-location** (Best approach)

```python
# Shard both tables by same key (user_id)
def get_shard(user_id):
    return shards[user_id % len(shards)]

# users and orders for same user_id on same shard
# Joins work normally!
```

---

### **Challenge 2: Distributed Transactions**

**Problem**: Transfer $100 from user 1001 (shard1) to user 2002 (shard2).

```python
# Naive approach (WRONG - not atomic)
def transfer(from_user, to_user, amount):
    from_shard = get_shard(from_user)
    to_shard = get_shard(to_user)

    from_shard.execute("UPDATE accounts SET balance = balance - ? WHERE user_id = ?", amount, from_user)
    # What if crash here? Money deducted but not added!
    to_shard.execute("UPDATE accounts SET balance = balance + ? WHERE user_id = ?", amount, to_user)
```

**Solutions**:

**1. Two-Phase Commit (2PC)**

```python
def transfer_2pc(from_user, to_user, amount):
    from_shard = get_shard(from_user)
    to_shard = get_shard(to_user)

    # Phase 1: Prepare
    from_shard.execute("BEGIN")
    to_shard.execute("BEGIN")

    from_shard.execute("UPDATE accounts SET balance = balance - ? WHERE user_id = ?", amount, from_user)
    to_shard.execute("UPDATE accounts SET balance = balance + ? WHERE user_id = ?", amount, to_user)

    # Phase 2: Commit
    try:
        from_shard.execute("PREPARE TRANSACTION 'txn_123'")
        to_shard.execute("PREPARE TRANSACTION 'txn_123'")

        # Both prepared successfully, commit both
        from_shard.execute("COMMIT PREPARED 'txn_123'")
        to_shard.execute("COMMIT PREPARED 'txn_123'")
    except:
        # Rollback both
        from_shard.execute("ROLLBACK PREPARED 'txn_123'")
        to_shard.execute("ROLLBACK PREPARED 'txn_123'")
```

**2. Saga Pattern** (Eventual consistency)

```python
def transfer_saga(from_user, to_user, amount):
    transaction_id = generate_uuid()

    # Step 1: Deduct from source
    from_shard = get_shard(from_user)
    from_shard.execute(
        "UPDATE accounts SET balance = balance - ? WHERE user_id = ?",
        amount, from_user
    )

    # Log compensation action
    log_compensation(transaction_id, 'refund_to_source', from_user, amount)

    try:
        # Step 2: Add to destination
        to_shard = get_shard(to_user)
        to_shard.execute(
            "UPDATE accounts SET balance = balance + ? WHERE user_id = ?",
            amount, to_user
        )

        # Success - mark transaction complete
        mark_transaction_complete(transaction_id)
    except:
        # Failure - execute compensation (refund)
        from_shard.execute(
            "UPDATE accounts SET balance = balance + ? WHERE user_id = ?",
            amount, from_user
        )
        mark_transaction_failed(transaction_id)
```

**3. Avoid Cross-Shard Transactions** (Best)

```python
# Design: Keep user's
balance on same shard as user
# Transfers only within same shard (same bank branch model)
# Cross-shard transfers go through central clearing house
```

---

### **Challenge 3: Rebalancing (Adding New Shards)**

**Problem**: Started with 4 shards, now need 8. How to move data?

**Naive Rehash (BAD)**:

```python
# Old: user_id % 4
# user 1001 → shard 1 (1001 % 4 = 1)

# New: user_id % 8
# user 1001 → shard 1 (1001 % 8 = 1) # Different shard!

# Problem: ALL keys need to move!
```

**Solutions**:

**1. Consistent Hashing** (shown earlier)

- Only ~12.5% of keys move (1/8) instead of 100%

**2. Virtual Shards (Bucket Mapping)**

```python
# Create 1000 virtual shards mapped to 4 physical shards
virtual_to_physical = {
    0: 'shard1',    # virtual shards 0-249 → shard1
    250: 'shard2',  # virtual shards 250-499 → shard2
    500: 'shard3',  # virtual shards 500-749 → shard3
    750: 'shard4'   # virtual shards 750-999 → shard4
}

def get_shard(user_id):
    virtual_shard = user_id % 1000
    for threshold, physical_shard in sorted(virtual_to_physical.items()):
        if virtual_shard < threshold + 250:
            return physical_shard

# Adding new shard: remap some virtual shards
# virtual shards 200-249 move from shard1 to shard5
# Only 5% of data moves
virtual_to_physical[200] = 'shard5'
```

**3. Range Splitting**

```python
# Old ranges
# shard1: 0-25M
# shard2: 25M-50M

# Split shard2 into shard2 and shard5
# shard1: 0-25M (unchanged)
# shard2: 25M-37M (half of original)
# shard5: 37M-50M (new shard)
```

---

## **On-Premises Sharding Setup**

### **MySQL with ProxySQL**

```bash
# Install ProxySQL (query router)
apt-get install proxysql

# Configure ProxySQL
mysql -u admin -padmin -h 127.0.0.1 -P6032

# Add MySQL shards
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES
(1, '192.168.1.10', 3306),  -- Shard 1
(2, '192.168.1.11', 3306),  -- Shard 2
(3, '192.168.1.12', 3306),  -- Shard 3
(4, '192.168.1.13', 3306);  -- Shard 4

# Configure routing rules
INSERT INTO mysql_query_rules (rule_id, match_pattern, destination_hostgroup) VALUES
(1, 'SELECT.*user_id.*[0-9]{1,7}$', 1),  -- user_id < 10M → shard1
(2, 'SELECT.*user_id.*[1-2][0-9]{7}$', 2); -- user_id 10M-30M → shard2

LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
```

**Application connects to ProxySQL, it routes queries**:

```python
import pymysql

# App connects to ProxySQL, not MySQL directly
conn = pymysql.connect(
    host='proxysql-host',
    port=6033,
    user='app_user',
    password='password'
)

# ProxySQL routes to correct shard automatically
cursor = conn.cursor()
cursor.execute("SELECT * FROM users WHERE user_id = 5000000")  # → shard1
cursor.execute("SELECT * FROM users WHERE user_id = 15000000") # → shard2
```

---

## **AWS Sharding Solutions**

### **Amazon Aurora with Read Replicas** (Simple Scaling)

```python
import pymysql

# Write to master
master_conn = pymysql.connect(
    host='mydb-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com',
    database='myapp'
)

# Read from replicas (up to 15 replicas)
replica_conn = pymysql.connect(
    host='mydb-cluster.cluster-ro-xxxxx.us-east-1.rds.amazonaws.com',
    database='myapp'
)

# Write
master_conn.cursor().execute("INSERT INTO users ...")

# Read (load balanced across replicas)
replica_conn.cursor().execute("SELECT * FROM users WHERE ...")
```

**Not true sharding, but handles read scaling for many use cases.**

---

### **DynamoDB** (Managed Sharding)

DynamoDB automatically shards data across partitions.

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# Put item (DynamoDB automatically shards by partition key)
table.put_item(
    Item={
        'user_id': 1001,  # Partition key
        'username': 'alice',
        'email': 'alice@example.com'
    }
)

# Get item (DynamoDB routes to correct partition)
response = table.get_item(Key={'user_id': 1001})

# Query all items with same partition key (single partition)
response = table.query(
    KeyConditionExpression='user_id = :uid',
    ExpressionAttributeValues={':uid': 1001}
)
```

**DynamoDB Partitioning**:

- Automatically creates partitions based on storage (10GB per partition) and throughput (3000 RCU or 1000 WCU)
- **Partition key** determines which partition stores data
- **Sort key** (optional) allows range queries within partition

**Best Practices**:

```python
# GOOD: High cardinality partition key
partition_key = user_id  # Millions of unique values

# BAD: Low cardinality partition key
partition_key = country  # Only ~200 unique values → hotspots

# GOOD: Composite key for even distribution
partition_key = f"{country}#{user_id}"  # Combines both
```

---

## **GCP Sharding Solutions**

### **Cloud Spanner** (Globally Distributed, Auto-Sharded)

```python
from google.cloud import spanner

# Connect to Spanner
spanner_client = spanner.Client()
instance = spanner_client.instance('my-instance')
database = instance.database('my-database')

# Insert (Spanner automatically shards by primary key)
def insert_user(user_id, username, email):
    with database.batch() as batch:
        batch.insert(
            table='users',
            columns=('user_id', 'username', 'email'),
            values=[(user_id, username, email)]
        )

# Query (Spanner routes to correct shard)
def get_user(user_id):
    with database.snapshot() as snapshot:
        results = snapshot.execute_sql(
            "SELECT * FROM users WHERE user_id = @user_id",
            params={'user_id': user_id},
            param_types={'user_id': spanner.param_types.INT64}
        )
        return list(results)

# Usage
insert_user(1001, 'alice', 'alice@example.com')
user = get_user(1001)
```

**Spanner Sharding**:

- Automatically splits data into **splits** (similar to shards)
- Globally distributed across regions
- Strong consistency (uses TrueTime for global clock synchronization)
- Handles up to petabytes of data

**Schema Design for Even Distribution**:

```sql
CREATE TABLE users (
    user_id INT64 NOT NULL,
    username STRING(100),
    created_at TIMESTAMP
) PRIMARY KEY (user_id);

-- GOOD: Avoid monotonically increasing keys
-- Use UUID or hash instead of auto-increment

-- BAD (creates hotspot):
user_id = AUTO_INCREMENT  -- All new writes go to same split

-- GOOD (distributes evenly):
user_id = HASH(uuid)  -- New writes distributed across splits
```

---

### **Cloud Bigtable** (NoSQL, Automatic Sharding)

```python
from google.cloud import bigtable

# Connect to Bigtable
client = bigtable.Client(project='my-project', admin=True)
instance = client.instance('my-instance')
table = instance.table('users')

# Write (Bigtable automatically shards by row key)
row_key = 'user#1001'.encode()
row = table.direct_row(row_key)
row.set_cell('profile', 'username', 'alice')
row.set_cell('profile', 'email', 'alice@example.com')
row.commit()

# Read (routes to correct tablet)
row = table.read_row(row_key)
username = row.cells['profile']['username'][0].value.decode()
```

**Bigtable Tablets** (shards):

- Data split into **tablets** (continuous ranges of row keys)
- Each tablet ~100MB-10GB
- Automatically splits/merges as data grows/shrinks

**Row Key Design** (critical for even distribution):

```python
# BAD: Sequential keys create hotspot
row_key = f"user#{user_id}"  # user#1, user#2, user#3... (all go to same tablet)

# GOOD: Salted/hashed keys distribute evenly
row_key = f"{hash(user_id) % 100}#user#{user_id}"  # Prefix with bucket 0-99

# GOOD: Reverse timestamp for time-series
row_key = f"{LONG_MAX - timestamp}#event#{event_id}"  # Recent events spread across tablets
```

---

## **Azure Sharding Solutions**

### **Azure Cosmos DB** (Globally Distributed, Auto-Partitioned)

```python
from azure.cosmos import CosmosClient

# Connect to Cosmos DB
client = CosmosClient(url, credential=key)
database = client.get_database_client('mydb')
container = database.get_container_client('users')

# Create item (Cosmos auto-partitions by partition key)
container.create_item(body={
    'id': '1001',
    'user_id': 1001,  # Partition key
    'username': 'alice',
    'email': 'alice@example.com'
})

# Read item (routes to correct partition)
item = container.read_item(item='1001', partition_key=1001)

# Query within partition (efficient)
items = container.query_items(
    query="SELECT * FROM c WHERE c.user_id = @user_id",
    parameters=[{'name': '@user_id', 'value': 1001}],
    partition_key=1001
)
```

**Cosmos DB Partitioning**:

- Automatically creates **physical partitions** (up to 50GB each)
- **Logical partitions**: All items with same partition key
- Partition key cannot be changed after creation

**Partition Key Design**:

```python
# GOOD: High cardinality
partition_key = "/user_id"  # Millions of unique values

# BAD: Low cardinality (creates hot partitions)
partition_key = "/country"  # Only ~200 values

# GOOD: Synthetic key for even distribution
partition_key = "/user_bucket"  # user_id % 100 → 100 logical partitions

# Item structure
{
    "id": "1001",
    "user_id": 1001,
    "user_bucket": 1,  # 1001 % 100 = 1
    "username": "alice"
}
```

**Cross-Partition Query** (expensive):

```python
# Queries ALL partitions (slow, expensive RU consumption)
items = container.query_items(
    query="SELECT * FROM c WHERE c.username = @username",
    parameters=[{'name': '@username', 'value': 'alice'}]
    # No partition_key specified → fan-out query
)
```

---

### **Azure SQL Database Elastic Pools** (Sharding)

```python
import pyodbc

# Shard map manager (tracks which data is on which shard)
shard_map = {
    'shard1': 'server1.database.windows.net',
    'shard2': 'server2.database.windows.net',
    'shard3': 'server3.database.windows.net'
}

def get_shard_connection(user_id):
    shard_id = user_id % len(shard_map)
    shard_server = shard_map[f'shard{shard_id + 1}']

    return pyodbc.connect(
        f'DRIVER={{ODBC Driver 17 for SQL Server}};'
        f'SERVER={shard_server};'
        f'DATABASE=myapp_shard{shard_id + 1};'
        f'UID=app_user;PWD=password'
    )

# Usage
user_id = 1001
conn = get_shard_connection(user_id)
cursor = conn.cursor()
cursor.execute("SELECT * FROM users WHERE user_id = ?", user_id)
```

**Azure Elastic Database Tools**:

- Shard map manager (tracks shard locations)
- Multi-shard query execution
- Split-merge tool for rebalancing

---

## **Monitoring Sharded Databases**

### **Key Metrics to Track**

```python
# Per-shard metrics
metrics = {
    'shard1': {
        'queries_per_sec': 5000,
        'cpu_usage': 45,
        'disk_usage': 60,
        'avg_query_time': 25,  # ms
        'connection_count': 200,
        'row_count': 25_000_000
    },
    'shard2': {
        'queries_per_sec': 12000,  # HOTSPOT!
        'cpu_usage': 85,             # OVERLOADED!
        'disk_usage': 90,            # NEARLY FULL!
        'avg_query_time': 150,       # SLOW!
        'connection_count': 500,
        'row_count': 45_000_000
    }
}

# Alert conditions
def check_shard_health(shard_metrics):
    alerts = []

    if shard_metrics['cpu_usage'] > 70:
        alerts.append('High CPU - consider splitting shard')

    if shard_metrics['disk_usage'] > 80:
        alerts.append('Low disk space - expand or archive')

    if shard_metrics['avg_query_time'] > 100:
        alerts.append('Slow queries - check indexes or split shard')

    # Check for imbalance across shards
    qps_variance = calculate_variance([s['queries_per_sec'] for s in all_shards])
    if qps_variance > threshold:
        alerts.append('Uneven load distribution - rebalance needed')

    return alerts
```
