# SQL vs NoSQL

## Introduction
The SQL vs NoSQL debate is one of the most consequential decisions in system architecture. For decades, relational databases (SQL) were the only choice for persistent storage. Then, in the late 2000s, NoSQL databases emerged from the need to handle web-scale workloads that relational databases struggled with—massive data volumes, horizontal scalability, flexible schemas, and high write throughput. Companies like Google (Bigtable), Amazon (DynamoDB), and Facebook (Cassandra) pioneered the movement.

Today, the landscape is nuanced. Most enterprises use a polyglot persistence approach, selecting different database types for different workloads within the same application. Understanding when to use each is fundamental to system design.

## Definition

### SQL (Relational Databases)
SQL databases store data in **structured, predefined schemas** with tables, rows, and columns. They enforce relationships through foreign keys and provide ACID transactions. Data is normalized to reduce redundancy.

Examples: PostgreSQL, MySQL, Oracle, SQL Server, Amazon Aurora

### NoSQL (Non-Relational Databases)
NoSQL databases use **flexible or schema-less** data models designed for specific access patterns and scalability requirements. They sacrifice some ACID guarantees for horizontal scalability and performance.

Examples: MongoDB (document), Cassandra (wide-column), Redis (key-value), Neo4j (graph), Elasticsearch (search)

## Concept Explanation

### SQL Database Characteristics

**Schema-on-Write**: Structure is defined before data is inserted. Every row in a table has the same columns with predefined types.

```sql
-- Schema must exist before data insertion
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    total DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Data must conform to schema
INSERT INTO orders (customer_id, total) VALUES (42, 199.99);
```

**Strengths**:
- **ACID Transactions**: Guaranteed consistency across multiple tables and rows
- **Joins**: Express complex relationships across tables efficiently
- **Mature Ecosystem**: Decades of tooling, ORMs, migrations, backup/restore, monitoring
- **Standardization**: SQL is universal; skills, tools, and libraries work across databases
- **Data Integrity**: Foreign keys, unique constraints, check constraints enforced at database level

**Weaknesses**:
- **Horizontal Scaling is Hard**: Sharding adds significant operational complexity
- **Schema Rigidity**: Schema changes require migrations, can lock tables, and cause downtime
- **Impedance Mismatch**: Object-oriented code must be mapped to relational tables (ORMs help but add overhead)

### NoSQL Database Types

#### Document Stores (MongoDB, Couchbase, Firestore)
Store data as JSON-like documents. Each document can have different fields.

```javascript
// Collection: users (schema-less)
db.users.insertMany([
  {
    _id: "user1",
    name: "Alice",
    email: "alice@example.com",
    preferences: { theme: "dark", language: "en" }
  },
  {
    _id: "user2",
    name: "Bob",
    phone: "+1234567890",  // Different fields allowed
    address: { city: "NYC", zip: "10001" }
  }
])
```

Best for: Content management, catalogs, user profiles, game state

#### Key-Value Stores (Redis, DynamoDB, etcd)
Simple hash table: key maps to value. Fastest for simple lookups.

```bash
redis-cli SET "session:abc123" '{"user_id":42,"expires":1700000000}'
redis-cli GET "session:abc123"
```

Best for: Caching, session storage, real-time counters, feature flags

#### Wide-Column Stores (Cassandra, HBase, Bigtable)
Tables with rows that can have different columns. Optimized for high write throughput.

```sql
-- Cassandra CQL
CREATE TABLE sensor_data (
    device_id UUID,
    timestamp TIMESTAMP,
    temperature FLOAT,
    humidity FLOAT,
    PRIMARY KEY (device_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

Best for: Time-series data, IoT telemetry, recommendation engines

#### Graph Databases (Neo4j, Amazon Neptune)
Optimized for highly connected data—nodes (entities) and edges (relationships).

```cypher
// Neo4j Cypher
CREATE (alice:Person {name: 'Alice'})-[:FRIEND_OF]->(bob:Person {name: 'Bob'})
MATCH (a:Person)-[:FRIEND_OF]->(f:Person)-[:FRIEND_OF]->(foaf:Person)
WHERE a.name = 'Alice'
RETURN foaf.name
```

Best for: Social networks, recommendation engines, fraud detection, knowledge graphs

### CAP Theorem Alignment
- **SQL databases** are typically **CP** (Consistent + Partition Tolerant): They prioritize data correctness over availability
- **NoSQL databases** vary:
  - MongoDB: CP (with replica sets), AP (with sharded clusters)
  - Cassandra: AP (tunable consistency)
  - DynamoDB: AP by default, CP with strongly consistent reads
  - Redis: CP (with persistence and replication), AP (pure cache mode)

## Layman's Explanation

### SQL: The Filing Cabinet
Imagine a filing cabinet with labeled drawers (tables), folders (rows), and tabs (columns). Every folder must follow the same template. You can't slip in a post-it note where a standard form is expected. Need to connect customer folders with order folders? You use a cross-reference number (foreign key). Change the form template? You must update every existing folder (schema migration).

This structure makes it incredibly reliable for financial records, inventory, and anything where consistency is paramount. But if you suddenly need to store millions of folders across multiple rooms (sharding), things get complicated fast.

### NoSQL: The Junk Drawer (but organized)
It's more like different storage systems for different needs:

- **Document Store**: Like having expandable folders where each folder can hold whatever papers you need, in whatever format. One customer folder might have a phone number; another might not.
- **Key-Value Store**: Like a valet parking ticket system. You hand over a ticket number (key), and they fetch your car (value) instantly. No fuss, no joins.
- **Graph Database**: Like a conspiracy theorist's corkboard with photos and strings connecting everything—perfect for "who knows whom" questions.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Data Modeling**: SQL requires upfront schema design thinking through all entities and relationships. NoSQL requires thinking through access patterns—how will data be queried? The data model should match the query pattern.
- **Scaling Strategy**: SQL scales vertically (bigger machine) with read replicas for horizontal read scaling. NoSQL scales horizontally (more machines) natively but requires understanding partitioning and consistency trade-offs
- **Transaction Requirements**: If you need multi-object ACID transactions (e.g., transferring money between accounts), SQL is the default choice until proven otherwise. Modern NoSQL databases support transactions but with limitations
- **Query Flexibility**: Ad-hoc queries with complex joins are SQL's strength. NoSQL typically requires designing the data model for known query patterns upfront
- **Operational Maturity**: SQL has 40+ years of tooling. NoSQL tools are catching up but operational expertise is less widespread

### Business Impact
- **Time to Market**: NoSQL's schema flexibility allows faster iteration when requirements are evolving
- **Performance at Scale**: NoSQL can handle workloads that would be prohibitively expensive on SQL (e.g., Facebook's inbox search on Cassandra)
- **Total Cost of Ownership**: SQL licensing (Oracle, SQL Server) can be expensive. Open-source NoSQL options reduce licensing but may increase operational complexity
- **Vendor Lock-in**: SQL standardization makes migration easier. NoSQL databases are more proprietary (though many now support SQL-like query languages)

### Decision Framework
Ask these questions:
1. **Is your data highly structured with relationships?** → SQL
2. **Do you need ACID transactions across multiple records?** → SQL
3. **Is write throughput extreme (millions/sec)?** → NoSQL (Cassandra, DynamoDB)
4. **Is schema rapidly evolving?** → NoSQL (Document stores)
5. **Are queries primarily key-based lookups?** → NoSQL (Key-value)
6. **Is data highly interconnected?** → NoSQL (Graph)
7. **Do you need full-text search?** → NoSQL (Elasticsearch)

## On-Premises Examples

### PostgreSQL (SQL)
The most advanced open-source relational database:

```bash
# Install
apt-get install postgresql

# Create database and user
sudo -u postgres psql -c "CREATE DATABASE ecommerce;"
sudo -u postgres psql -c "CREATE USER app_user WITH PASSWORD 'secure_password';"
```

```sql
-- Schema with constraints and relationships
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    total DECIMAL(10,2) CHECK (total > 0),
    status VARCHAR(20) DEFAULT 'pending'
);

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status) WHERE status = 'pending';
```

### MongoDB (NoSQL - Document)
```bash
# Install and start
mongod --dbpath /var/lib/mongodb --bind_ip 127.0.0.1

# Insert documents
mongosh
```

```javascript
use ecommerce;
db.orders.insertOne({
  customer: { id: 42, email: "alice@example.com" },
  items: [
    { sku: "SKU-001", qty: 2, price: 29.99 },
    { sku: "SKU-002", qty: 1, price: 49.99 }
  ],
  total: 109.97,
  status: "pending",
  createdAt: new Date()
});

db.orders.createIndex({ "customer.id": 1, status: 1 });
```

### Hybrid On-Premises
Run PostgreSQL for transactional data (orders, payments) and Cassandra for high-volume event streams (clickstreams, logs) within the same data center.

## AWS Examples

### Amazon RDS / Aurora (SQL)
Fully managed relational databases:

```hcl
resource "aws_db_instance" "orders" {
  identifier     = "orders-db"
  engine         = "postgres"
  engine_version = "16.1"
  instance_class = "db.r6g.large"
  
  allocated_storage     = 100
  storage_encrypted     = true
  multi_az             = true
  backup_retention_period = 30
  
  db_name  = "ecommerce"
  username = "admin"
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.db.id]
}
```

Aurora provides up to 5x MySQL and 3x PostgreSQL throughput with automatic storage scaling up to 128TB.

### Amazon DynamoDB (NoSQL - Key-Value/Document)
Serverless, auto-scaling NoSQL:

```python
import boto3

dynamodb = boto3.resource('dynamodb')

# Create table with on-demand capacity
table = dynamodb.create_table(
    TableName='Orders',
    KeySchema=[
        {'AttributeName': 'OrderID', 'KeyType': 'HASH'},
        {'AttributeName': 'Timestamp', 'KeyType': 'RANGE'}
    ],
    AttributeDefinitions=[
        {'AttributeName': 'OrderID', 'AttributeType': 'S'},
        {'AttributeName': 'Timestamp', 'AttributeType': 'N'}
    ],
    BillingMode='PAY_PER_REQUEST'
)

# Query with Global Secondary Index
table = dynamodb.Table('Orders')
response = table.query(
    IndexName='CustomerID-index',
    KeyConditionExpression='CustomerID = :cid',
    ExpressionAttributeValues={':cid': 'CUST-789'}
)
```

### Amazon DocumentDB (NoSQL - MongoDB compatible)
Managed MongoDB-compatible service:

```python
from pymongo import MongoClient

client = MongoClient('mongodb://user:pass@docdb-cluster.us-east-1.docdb.amazonaws.com:27017')
db = client['ecommerce']
db.orders.find({"status": "pending"}).sort("createdAt", -1).limit(50)
```

### Amazon Neptune (NoSQL - Graph)
Managed graph database for highly connected data:

```python
# Gremlin traversal
g.V().has('Customer', 'email', 'alice@example.com')
  .out('PLACED')
  .has('Order', 'status', 'completed')
  .values('total')
  .sum()
```

## GCP Examples

### Cloud SQL (SQL)
Managed MySQL, PostgreSQL, and SQL Server:

```bash
gcloud sql instances create orders-db \
  --database-version=POSTGRES_16 \
  --cpu=2 --memory=8GB \
  --region=us-central1 \
  --availability-type=REGIONAL \
  --storage-type=SSD --storage-size=100GB
```

### Cloud Spanner (SQL - Globally Distributed)
Unique offering: SQL with horizontal scalability and strong consistency:

```sql
CREATE TABLE Orders (
  OrderID    STRING(36) NOT NULL,
  CustomerID STRING(36) NOT NULL,
  Total      FLOAT64 NOT NULL,
  Status     STRING(20) NOT NULL DEFAULT 'pending',
  CreatedAt  TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true),
) PRIMARY KEY (OrderID);

-- Global strong consistency
SELECT SUM(Total) FROM Orders WHERE CustomerID = 'CUST-789';
```

### Firestore (NoSQL - Document)
Serverless, real-time document database:

```python
from google.cloud import firestore

db = firestore.Client()
doc_ref = db.collection('orders').document('ORD-001')
doc_ref.set({
    'customer_id': 'CUST-789',
    'items': [{'sku': 'SKU-001', 'qty': 2}],
    'total': 59.98,
    'created_at': firestore.SERVER_TIMESTAMP
})

# Real-time listener
def on_snapshot(doc_snapshot, changes, read_time):
    for doc in doc_snapshot:
        print(f'Order update: {doc.id}')

db.collection('orders').where('status', '==', 'pending').on_snapshot(on_snapshot)
```

### Bigtable (NoSQL - Wide Column)
Massive petabyte-scale analytical workloads:

```bash
gcloud bigtable instances create analytics-db \
  --display-name="Analytics" \
  --cluster=primary-cluster \
  --cluster-zone=us-central1-a \
  --cluster-num-nodes=3
```

## Azure Examples

### Azure SQL Database (SQL)
Managed SQL Server with built-in intelligence:

```bash
az sql server create \
  --name order-db-server \
  --resource-group myResourceGroup \
  --location eastus \
  --admin-user admin \
  --admin-password "$DB_PASSWORD"

az sql db create \
  --resource-group myResourceGroup \
  --server order-db-server \
  --name ecommerce \
  --service-objective GP_Gen5_2 \
  --zone-redundant true
```

### Azure Cosmos DB (NoSQL - Multi-Model)
Globally distributed, multi-model database:

```python
from azure.cosmos import CosmosClient

client = CosmosClient('https://my-cosmos.documents.azure.com', credential=KEY)
database = client.create_database_if_not_exists('ecommerce')
container = database.create_container_if_not_exists(
    id='orders',
    partition_key=PartitionKey(path='/customerId')
)

container.create_item({
    'id': 'ORD-001',
    'customerId': 'CUST-789',
    'total': 99.99,
    'status': 'pending'
})

# Cosmos DB offers 5 consistency levels from strong to eventual
```

### Azure Database for PostgreSQL / MySQL
Managed open-source relational databases with built-in high availability, automatic backups, and point-in-time restore.

## Summary Decision Matrix

| Factor | SQL | NoSQL |
|--------|-----|-------|
| Data Model | Fixed schema, normalized | Flexible schema, denormalized |
| Transactions | Full ACID | Varies (some ACID, some BASE) |
| Scaling | Vertical, read replicas | Horizontal, native sharding |
| Query Complexity | Complex joins, ad-hoc queries | Simple queries, pre-planned access patterns |
| Consistency | Strong | Tunable (strong to eventual) |
| Use Case | ERP, banking, inventory | Social media, IoT, real-time analytics |
| Maturity | 40+ years | 15+ years |

The right choice is rarely "SQL or NoSQL" but rather "which database for which component." A modern e-commerce platform might use PostgreSQL for orders and payments, MongoDB for product catalogs, Redis for session caching, and Elasticsearch for product search—each chosen for its strengths in that specific context.
