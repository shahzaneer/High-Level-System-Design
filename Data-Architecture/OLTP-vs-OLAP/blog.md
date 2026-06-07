# OLTP vs OLAP

## Introduction
OLTP (Online Transaction Processing) and OLAP (Online Analytical Processing) represent the two fundamental database workload paradigms that underpin nearly every business application. OLTP systems handle real-time transactional workloads—creating orders, updating inventory, processing payments. OLAP systems handle analytical workloads—generating reports, running business intelligence queries, training machine learning models.

The architectural divergence between these workloads is profound: OLTP optimizes for low-latency, high-concurrency, row-oriented access; OLAP optimizes for high-throughput, complex aggregations, column-oriented access. Trying to serve both workloads from the same database inevitably leads to performance degradation in both. Understanding this distinction—and designing appropriate data architectures for each—is foundational to system design.

## Definition

**OLTP (Online Transaction Processing)** systems are optimized for managing transaction-oriented applications, typically involving large numbers of short, simple queries (INSERT, UPDATE, DELETE, simple SELECT) from many concurrent users. Characteristics:

- High concurrency (thousands of simultaneous users)
- Low latency (sub-millisecond to single-digit millisecond queries)
- Row-oriented storage (optimized for accessing entire rows)
- ACID transactions (atomicity, consistency, isolation, durability)
- Normalized schemas (minimize data duplication, maximize integrity)

**OLAP (Online Analytical Processing)** systems are optimized for complex analytical queries, typically involving aggregations, joins across large datasets, and multi-dimensional analysis. Characteristics:

- Low concurrency (tens of simultaneous analysts, or batch jobs)
- Higher latency (seconds to minutes per query)
- Column-oriented storage (optimized for aggregating specific columns across millions of rows)
- Eventually consistent or snapshot-based consistency
- Denormalized schemas (star/snowflake schemas, pre-aggregated)

## Concept Explanation

### Row vs Column-Oriented Storage

```
ROW-ORIENTED (OLTP: PostgreSQL, MySQL):
Data stored row-by-row on disk.

Row 1: [id=1, name="Alice", age=30, city="NYC", salary=75000]
Row 2: [id=2, name="Bob",   age=25, city="LA",  salary=65000]
Row 3: [id=3, name="Carol", age=35, city="NYC", salary=85000]

SELECT * FROM users WHERE id = 1;  -- Single disk seek → entire row
INSERT INTO users VALUES (4, 'Dave', 28, 'SF', 70000);  -- Append to end
→ OLTP optimized: Single-row access is fast


COLUMN-ORIENTED (OLAP: Redshift, BigQuery, Snowflake, ClickHouse):
Data stored column-by-column on disk.

id column:     [1,  2,  3]
name column:   ["Alice", "Bob", "Carol"]
age column:    [30, 25, 35]
city column:   ["NYC", "LA", "NYC"]
salary column: [75000, 65000, 85000]

SELECT AVG(salary) FROM users WHERE city = 'NYC';
→ Only reads salary column (1 seek) + city column (1 seek)
→ Doesn't read name, age, id columns at all
→ OLAP optimized: Column aggregations are fast
```

### Data Models

#### OLTP: Normalized (Third Normal Form)
```sql
-- Normalized: minimize redundancy
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date TIMESTAMP DEFAULT NOW(),
    status VARCHAR(20)
);

CREATE TABLE order_items (
    item_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL
);

-- Writing an order: 3 simple, fast INSERTs
BEGIN;
INSERT INTO orders (customer_id) VALUES (42) RETURNING order_id;
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (1, 100, 2, 29.99);
COMMIT;
```

#### OLAP: Denormalized (Star Schema)
```sql
-- Denormalized: optimized for querying, not writing
CREATE TABLE fact_orders (
    order_id INTEGER,
    date_key INTEGER REFERENCES dim_date(date_key),
    customer_key INTEGER REFERENCES dim_customer(customer_key),
    product_key INTEGER REFERENCES dim_product(product_key),
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    total_amount DECIMAL(10,2),  -- Pre-calculated (denormalized)
    order_count INTEGER DEFAULT 1
);

CREATE TABLE dim_date (
    date_key INTEGER PRIMARY KEY,
    full_date DATE,
    year INTEGER,
    quarter INTEGER,
    month INTEGER,
    day_of_week VARCHAR(10),
    is_holiday BOOLEAN
);

CREATE TABLE dim_customer (
    customer_key INTEGER PRIMARY KEY,
    customer_name VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    segment VARCHAR(50)
);

-- Analytical query: Revenue by product category by month
SELECT 
    d.year, d.month,
    p.category,
    SUM(f.total_amount) as revenue,
    COUNT(DISTINCT f.customer_key) as unique_customers
FROM fact_orders f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
WHERE d.year = 2024
GROUP BY d.year, d.month, p.category
ORDER BY d.month, revenue DESC;
-- Scans millions of rows, aggregates across all of them
-- Columnar storage: reads only year, month, category, total_amount, customer_key columns
```

### HTAP (Hybrid Transactional/Analytical Processing)

Modern databases are blurring the line:

```
Traditional:        Modern HTAP:
┌──────┐ ┌──────┐   ┌──────────────────┐
│ OLTP │ │ OLAP │   │  Single Database  │
│      │ │      │   │  (TiDB, Cockroach │
│ Post │ │Redsh│   │   DB, SingleStore,│
│ greSQL│ │ift  │   │   AlloyDB)        │
└──┬───┘ └──┬───┘   │                   │
   │        │       │  Row store (OLTP) │
   └───┬────┘       │  Column store     │
       │            │  (OLAP)           │
   ETL Pipeline     └──────────────────┘
   (hours of delay)
```

```sql
-- CockroachDB: OLTP + OLAP in the same database
-- Row-oriented primary index for OLTP
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID,
    total DECIMAL,
    order_date TIMESTAMP
);

-- Column-oriented secondary index for OLAP (inverted index)
CREATE INVERTED INDEX orders_analytics 
    ON orders (order_date, customer_id, total);
```

## Layman's Explanation

### The Filing Cabinet vs The Spreadsheet
**OLTP (Row-Oriented = Filing Cabinet)**: Each customer has a folder (row) with all their documents (columns): application form, ID copy, contract, payment history. When a customer calls, you pull ONE folder and see everything about them. Fast for individual lookups and updates. Slow if you want to know "what's the average age of all customers?" because you'd have to open every folder and check the birth date field.

**OLAP (Column-Oriented = Spreadsheet)**: All customer data is in a spreadsheet. The "age" column is stored together, so calculating average age reads one contiguous block of data (fast). The "total purchases" column is stored together, so summing purchases is fast. But updating one customer's phone number requires finding their row across all 50 columns—slow.

### The Restaurant Analogy
**OLTP**: The point-of-sale system at a busy restaurant. Waiters enter orders constantly (INSERT), modify them (UPDATE), check statuses (SELECT). Hundreds of small, fast transactions per hour. The system must never lose an order (ACID).

**OLAP**: The restaurant's monthly business report. The owner wants to know: "Which dishes sold best on weekends last quarter?" and "What's the average check size by server?" These queries scan ALL orders from the last 3 months, aggregate, group, and sort. The owner runs this once a month (low concurrency) and waits 30 seconds (acceptable latency).

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Separation of Concerns**: Putting analytical queries on the OLTP database is the #1 cause of production database performance issues. A single unoptimized reporting query can lock tables and block all transactional traffic. The architect must design separate systems: OLTP for transactions, OLAP for analytics, with an ETL/CDC pipeline between them.
- **ETL/ELT Pipeline Design**: Getting data from OLTP to OLAP involves: Extraction (CDC from transaction logs, avoiding table locks), Transformation (cleaning, denormalizing, enriching), and Loading (batch or streaming into the analytical store). Pipeline latency determines data freshness—batch ETL (hourly) vs streaming ETL (seconds).
- **Data Model Design**: OLTP demands normalized schemas (3NF) for write performance and data integrity. OLAP demands denormalized schemas (star/snowflake) for read performance. Designing both models and keeping them synchronized is a core architectural challenge.
- **Technology Selection**: The OLTP (PostgreSQL, MySQL, RDS) and OLAP (Redshift, BigQuery, Snowflake, ClickHouse) technology choices are independent. You might run OLTP on RDS PostgreSQL and OLAP on BigQuery, with data flowing between them via CDC + Dataflow.

### Business Impact
- **User Experience**: A checkout page backed by a database that's also running monthly reports results in 10-second checkout times. Users abandon carts. Separating OLTP and OLAP keeps the checkout fast.
- **Data-Driven Decisions**: Without an OLAP system, business intelligence requires "please run this query against production during off-hours" emails to DBAs. With OLAP, business analysts self-serve with sub-second to sub-minute queries.
- **Cost**: OLTP databases run 24/7 on provisioned capacity. OLAP databases can be serverless (BigQuery, Redshift Serverless, Snowflake) that charge per query—you pay for analytics only when someone's actually analyzing.

## On-Premises Examples

### PostgreSQL + ClickHouse (OLTP + OLAP)
```bash
# OLTP: PostgreSQL for transactions
psql -c "INSERT INTO orders VALUES (...)"

# CDC to OLAP: Debezium captures PostgreSQL WAL changes
# → Kafka → ClickHouse consumer

# OLAP: ClickHouse for analytics
clickhouse-client --query "
SELECT 
    toStartOfMonth(order_date) AS month,
    product_category,
    sum(total_amount) AS revenue
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY month, product_category
ORDER BY month, revenue DESC
"
```

### MySQL + Apache Druid
```bash
# OLTP: MySQL
# Streaming ingestion via Kafka → Druid (real-time OLAP)
# Sub-second queries on billions of rows
```

## AWS Examples

### RDS + Redshift (Classic AWS Architecture)
```hcl
resource "aws_db_instance" "oltp" {
  identifier = "orders-db"
  engine     = "postgres"
  instance_class = "db.r6g.xlarge"
}

resource "aws_redshift_cluster" "olap" {
  cluster_identifier = "analytics"
  node_type          = "ra3.xlplus"
  number_of_nodes    = 2
}
```

```python
# AWS DMS: CDC from RDS to Redshift
dms_client.create_replication_task(
    ReplicationTaskIdentifier='oltp-to-olap',
    SourceEndpointArn=rds_endpoint_arn,
    TargetEndpointArn=redshift_endpoint_arn,
    MigrationType='full-load-and-cdc',  # Full + ongoing changes
    TableMappings=json.dumps({
        'rules': [{
            'rule-type': 'selection',
            'rule-id': '1',
            'rule-name': 'orders-tables',
            'object-locator': {
                'schema-name': 'public',
                'table-name': '%'
            },
            'rule-action': 'include'
        }]
    })
)
```

### Aurora + Redshift Spectrum (Data Lake Querying)
```sql
-- Redshift Spectrum queries data directly in S3 (no loading needed)
CREATE EXTERNAL SCHEMA spectrum
FROM DATA CATALOG DATABASE 'orders_data_lake'
IAM_ROLE 'arn:aws:iam::...';

SELECT 
    product_category,
    SUM(total_amount) as revenue
FROM spectrum.orders_parquet  -- Queries Parquet files in S3
WHERE order_date >= '2024-01-01'
GROUP BY product_category;
```

## GCP Examples

### Cloud SQL + BigQuery (Google's Classic Architecture)
```bash
# OLTP: Cloud SQL (PostgreSQL)
gcloud sql instances create orders-db

# OLAP: BigQuery
bq mk analytics

# CDC: Datastream captures Cloud SQL changes → BigQuery
gcloud datastream streams create orders-cdc \
  --source=mysql-source \
  --destination=bigquery-destination

# Query in BigQuery
bq query --nouse_legacy_sql '
SELECT 
    DATE_TRUNC(order_date, MONTH) as month,
    product_category,
    SUM(total_amount) as revenue
FROM analytics.orders
WHERE order_date >= "2024-01-01"
GROUP BY month, product_category
ORDER BY revenue DESC
'
```

### Spanner (HTAP in One Database)
```sql
-- Cloud Spanner: OLTP + OLAP without ETL
-- OLTP: transactional reads/writes
INSERT INTO Orders (OrderID, CustomerID, Total, OrderDate)
VALUES ('ORD-123', 'CUST-789', 99.99, CURRENT_TIMESTAMP());

-- OLAP: analytical queries on the same table
SELECT 
    DATE_TRUNC(OrderDate, MONTH) as month,
    SUM(Total) as revenue
FROM Orders
WHERE OrderDate >= '2024-01-01'
GROUP BY month
ORDER BY month;
-- Spanner handles both workloads in the same database
```

## Azure Examples

### Azure SQL + Synapse Analytics
```bash
# OLTP: Azure SQL
az sql db create --name orders-db --server myServer

# OLAP: Azure Synapse Analytics (formerly SQL Data Warehouse)
az synapse workspace create --name analytics-workspace

# Synapse Link: near real-time CDC from Azure SQL to Synapse
az sql db update --name orders-db --server myServer \
  --enable-synapse-link true
```

### Azure Cosmos DB HTAP
```json
// Cosmos DB: OLTP + OLAP via Synapse Link
// Transactional store (OLTP) + Analytical store (OLAP) in same database
{
  "id": "ORD-123",
  "customerId": "CUST-789",
  "total": 99.99,
  "status": "shipped"  
}
// Automatically synced to columnar analytical store
// No ETL pipeline needed
```

## Summary

| Characteristic | OLTP | OLAP |
|---------------|------|------|
| Primary use | Business transactions | Business analytics |
| Query type | Simple CRUD | Complex aggregations, joins |
| Data model | Normalized (3NF) | Denormalized (Star/Snowflake) |
| Storage | Row-oriented | Column-oriented |
| Concurrency | High (thousands of users) | Low (tens of analysts) |
| Latency | Sub-ms to ms | Seconds to minutes |
| Throughput | High write volume | High read volume |
| Data retention | Current + recent | Historical (years) |
| Examples | PostgreSQL, MySQL, RDS, Cloud SQL | Redshift, BigQuery, Snowflake, ClickHouse |

OLTP and OLAP are complementary, not competitive. Every business needs both: OLTP to run the business in real-time and OLAP to understand how the business is performing. The architect's job is to design the data pipeline that connects them—CDC from OLTP to OLAP—and to ensure that analytical workloads never degrade transactional performance. The trend toward HTAP (single database for both) is promising but still emerging; for most production systems, separate OLTP and OLAP with asynchronous data synchronization remains the proven architecture.
