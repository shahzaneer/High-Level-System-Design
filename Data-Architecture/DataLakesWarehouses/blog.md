# Data Lakes & Warehouses

## Introduction
Data Lakes and Data Warehouses are the two dominant architectural patterns for storing and analyzing large-scale data. The Data Warehouse emerged in the 1990s (Ralph Kimball, Bill Inmon) as a structured, schema-on-write repository optimized for business intelligence. The Data Lake emerged in the 2010s as a response to the explosion of unstructured and semi-structured data—sensor logs, social media feeds, clickstreams—that didn't fit neatly into warehouse schemas.

The modern evolution is the Data Lakehouse—an architecture that combines the schema flexibility and low-cost storage of a data lake with the ACID transactions and performance of a data warehouse. Technologies like Delta Lake (Databricks), Apache Iceberg, and Apache Hudi have made this convergence practical, enabling organizations to store all their data in a single architecture while supporting both BI and ML workloads.

## Definition

**Data Warehouse**: A centralized repository optimized for storing structured, processed data and serving analytical queries. Key characteristics:
- Schema-on-write (data must conform to schema before loading)
- Optimized for SQL-based analytics
- High query performance on structured data
- Business intelligence and reporting focus
- Higher storage cost per GB

**Data Lake**: A centralized repository that stores raw data in its native format (structured, semi-structured, unstructured) at any scale. Key characteristics:
- Schema-on-read (schema applied when querying, not when storing)
- Stores raw, unprocessed data (CSV, JSON, Parquet, Avro, images, logs)
- Supports diverse workloads (SQL, ML, streaming)
- Lower storage cost per GB
- Risk of becoming a "data swamp" without governance

**Data Lakehouse**: An architecture that combines data lake flexibility with warehouse reliability:
- ACID transactions on data lake storage
- Schema enforcement and evolution
- Direct SQL and ML on the same data
- Open formats (Parquet, Iceberg) avoiding vendor lock-in

## Concept Explanation

### Architecture Comparison

```
DATA WAREHOUSE:
┌─────────────────────────────────────┐
│  BI Tools (Tableau, Looker, PowerBI) │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│      DATA WAREHOUSE (SQL)           │
│  ┌────────────────────────────┐    │
│  │ Star Schema (Fact + Dims)  │    │
│  │ Partitioned, Clustered,    │    │
│  │ Aggregated                 │    │
│  └────────────────────────────┘    │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│          ETL PIPELINE               │
│  ┌─────────┐  ┌─────────────────┐  │
│  │ Extract │→│ Transform+Clean  │  │
│  │ (OLTP)  │  │ (Business Logic) │  │
│  └─────────┘  └────────┬────────┘  │
│                        │           │
│               Load into Warehouse  │
└────────────────────────────────────┘


DATA LAKE:
┌─────────────────────────────────────────────────────────┐
│  Consumers: BI │ Data Science │ ML Training │ Streaming │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│               DATA LAKE (Object Storage)                │
│  ┌─────────┐ ┌────────┐ ┌──────────┐ ┌──────────────┐ │
│  │ Bronze  │→│ Silver │→│  Gold    │→│  Sandbox     │ │
│  │ (Raw)   │ │(Clean) │ │(Business)│ │(Experiments) │ │
│  └─────────┘ └────────┘ └──────────┘ └──────────────┘ │
│                                                         │
│  Bronze: Raw CSV, JSON, Parquet, Avro (as-is)          │
│  Silver: Deduplicated, cleaned, validated              │
│  Gold:   Aggregated, business-ready tables             │
│  Sandbox: Data science experiments with copies         │
└─────────────────────────────────────────────────────────┘
```

### Medallion Architecture (Bronze → Silver → Gold)

```python
class MedallionPipeline:
    """
    Databricks Lakehouse pattern: progressive data refinement.
    """
    
    def bronze_layer(self):
        """Ingest raw data as-is. No transformation. Append-only."""
        # Bronze: Raw orders from Kafka
        spark.readStream \
            .format("kafka") \
            .option("subscribe", "orders") \
            .load() \
            .writeStream \
            .format("delta") \
            .option("checkpointLocation", "/checkpoints/orders_bronze") \
            .start("/data-lake/bronze/orders")
        
        # Schema: whatever the producer sent
        # Data quality: whatever the producer sent
        # Purpose: replay source of truth, reprocessing capability
    
    def silver_layer(self):
        """Clean, validate, deduplicate. Enforce schema."""
        bronze_df = spark.read.format("delta").load("/data-lake/bronze/orders")
        
        silver_df = bronze_df \
            .dropDuplicates(["order_id"]) \
            .filter(col("total").isNotNull() & (col("total") > 0)) \
            .withColumn("processed_at", current_timestamp()) \
            .withColumn("order_date", col("created_at").cast("date"))
        
        silver_df.write \
            .format("delta") \
            .mode("append") \
            .save("/data-lake/silver/orders")
    
    def gold_layer(self):
        """Business-level aggregates. Ready for BI and ML."""
        silver_df = spark.read.format("delta").load("/data-lake/silver/orders")
        
        # Daily revenue by product category
        gold_df = silver_df \
            .groupBy("order_date", "product_category") \
            .agg(
                sum("total").alias("revenue"),
                count("order_id").alias("order_count"),
                avg("total").alias("avg_order_value")
            )
        
        gold_df.write \
            .format("delta") \
            .mode("overwrite") \
            .option("replaceWhere", "order_date >= '2024-01-01'") \
            .save("/data-lake/gold/daily_revenue")
```

### Table Formats: Delta Lake, Iceberg, Hudi

```
Problem: Parquet files in a data lake are just files.
         No transactions, no time travel, no schema enforcement.

Solution: Table formats add database-like features on top of files.

┌──────────────────────────────────────────────┐
│               DELTA LAKE                      │
│  ┌──────────────────────────────────────┐    │
│  │  _delta_log/                         │    │
│  │    000.json (Add orders_part1.parquet)│    │
│  │    001.json (Add orders_part2.parquet)│    │
│  │    002.json (Remove orders_part1.parq)│    │
│  │                                      │    │
│  │  Features:                           │    │
│  │  • ACID Transactions                 │    │
│  │  • Time Travel (query as of version) │    │
│  │  • Schema Enforcement                │    │
│  │  • Schema Evolution                  │    │
│  │  • Upserts/Merge (CDC)               │    │
│  │  • Compaction (small files → big)    │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

```sql
-- Delta Lake Time Travel
-- Query data as it existed 7 days ago
SELECT * FROM orders
  TIMESTAMP AS OF current_timestamp() - INTERVAL 7 DAYS;

-- Query data at a specific version
SELECT * FROM orders
  VERSION AS OF 42;

-- Rollback a bad update
RESTORE TABLE orders TO VERSION AS OF 42;
```

### Query Engines

```
QUERY ENGINES (compute) ──query──→ TABLE FORMAT (metadata) ──read──→ STORAGE (data)
                                    │                                    │
                              Delta Lake/Iceberg              S3/ADLS/GCS/Object Store
                              (transaction log)              (Parquet files)
```

```sql
-- Presto/Trino: Federated SQL across data lake + warehouse
SELECT 
    o.order_id,
    o.total,
    c.name,
    c.segment
FROM delta_lake.orders o
JOIN warehouse.customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01';
-- Queries Delta Lake (S3) and Data Warehouse simultaneously
-- No data movement needed
```

## Layman's Explanation

### The Library vs The Self-Storage Facility

**Data Warehouse**: A traditional library. Every book (data) is catalogued, categorized, and placed on a specific shelf according to the Dewey Decimal System (schema-on-write). You can find exactly what you need quickly. But adding a new type of media (video games? sensor data?) requires redesigning the cataloguing system.

**Data Lake**: A self-storage facility. You dump everything in—furniture (structured data), boxes of photos (semi-structured JSON), old VHS tapes (unstructured data). No cataloguing required upfront. But finding that one photo of grandma becomes a massive search expedition (schema-on-read). Without organization, it becomes a "data swamp"—a pile of junk nobody can find anything in.

**Data Lakehouse**: A modern library with climate-controlled self-storage. You store everything in storage containers (S3), but each container has a detailed digital catalogue (Delta Lake transaction log) with ACID guarantees. The librarian (query engine) can find anything quickly, whether it's a book, a VHS tape, or a sensor log.

### Why You Need Both
You wouldn't close your library to store everything in self-storage units (data lake only = slow queries). And you wouldn't build a library wing for every box of photos (warehouse only = expensive for raw data). The Lakehouse gives you the best of both.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Schema-on-Write vs Schema-on-Read**: Warehouse requires knowing the schema upfront (good for well-understood data like orders). Lake allows deferring schema decisions (good for exploratory data like sensor logs where you don't know what questions you'll ask yet).
- **Storage Format**: CSV/JSON (human-readable, slow, no compression) vs Parquet/ORC (columnar, compressed, 5-10x smaller, 10-100x faster queries). Always Parquet/ORC for analytical workloads. CSV is only for human inspection.
- **Partitioning Strategy**: Determines query performance and cost. Common strategies: date-based (`order_date=2024-06-15/`), category-based, or multi-level. Over-partitioning (100,000 tiny partitions) is as bad as no partitioning. Target partition sizes of 100MB-1GB.
- **Governance**: Without governance, a Data Lake becomes a Data Swamp. Need: data catalog (AWS Glue, Dataplex, Purview), access controls (Lake Formation, IAM, Ranger), data lineage, data quality checks, and PII discovery.

### Business Impact
- **Analytics Agility**: A Data Lake allows data scientists to access raw data directly without waiting for ETL pipelines. This reduces time-to-insight from weeks (request → ETL → warehouse → analyze) to hours (directly query raw data in the lake).
- **Cost**: S3/GCS/ADLS storage costs $0.02-0.03/GB/month. Warehouse storage costs $0.25-1.00/GB/month (proprietary, managed, higher performance). Storing raw data in the lake and curated data in the warehouse optimizes cost.
- **ML Enablement**: Machine learning requires raw, granular data—not pre-aggregated warehouse summaries. A Data Lake provides the raw data ML engineers need without duplicating the warehouse data.

## On-Premises Examples

### MinIO (S3-Compatible Data Lake)
```bash
# Deploy MinIO cluster
minio server http://node{1...4}/export{1...4}

# Use with Spark + Delta Lake
spark-submit \
  --packages io.delta:delta-spark_2.12:3.0.0 \
  --conf spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension \
  --conf spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog \
  etl_job.py
```

```python
# Write Delta Lake table to MinIO
df.write \
  .format("delta") \
  .mode("append") \
  .save("s3a://data-lake/gold/daily_revenue")
```

### ClickHouse (Open-Source OLAP)
```sql
-- ClickHouse on-premises warehouse
CREATE TABLE orders (
    order_date Date,
    product_category String,
    revenue Decimal(15,2),
    order_count UInt32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(order_date)
ORDER BY (product_category, order_date);

-- Sub-second queries on billions of rows
SELECT 
    product_category,
    sum(revenue) as total_revenue
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY product_category;
```

## AWS Examples

### S3 Data Lake + Glue Catalog + Athena
```hcl
resource "aws_glue_catalog_database" "data_lake" {
  name = "orders_data_lake"
}

resource "aws_glue_crawler" "orders" {
  name          = "orders-crawler"
  database_name = aws_glue_catalog_database.data_lake.name
  role          = aws_iam_role.glue.arn

  s3_target {
    path = "s3://data-lake/silver/orders/"
  }

  schema_change_policy {
    update_behavior = "UPDATE_IN_DATABASE"
    delete_behavior = "LOG"
  }
}
```

```sql
-- Athena (serverless SQL on S3)
SELECT 
    product_category,
    SUM(total_amount) as revenue,
    COUNT(DISTINCT customer_id) as customers
FROM orders_data_lake.orders
WHERE order_date >= DATE '2024-01-01'
GROUP BY product_category
ORDER BY revenue DESC;
```

### Redshift Spectrum (Warehouse querying Data Lake)
```sql
-- Redshift queries data directly in S3 (no loading)
CREATE EXTERNAL TABLE spectrum.orders_raw
PARTITIONED BY (order_date date)
STORED AS PARQUET
LOCATION 's3://data-lake/bronze/orders/';

-- Query external data like it's in Redshift
SELECT * FROM spectrum.orders_raw WHERE order_date = '2024-06-15';
```

## GCP Examples

### BigQuery (Serverless Data Warehouse)
```bash
# External table querying data directly from GCS
bq mk --external_table_definition=gs://data-lake/orders/*.parquet@PARQUET \
  analytics.orders

# Query directly—no loading needed
bq query '
SELECT product_category, SUM(total) as revenue
FROM analytics.orders
WHERE order_date >= "2024-01-01"
GROUP BY product_category
'
```

### Dataproc + BigLake
```bash
# BigLake: Fine-grained access control on data lake tables
gcloud dataplex lakes create data-lake \
  --location=us-central1

# Governed tables: apply BigQuery access controls to GCS files
```

## Azure Examples

### Azure Data Lake Storage (ADLS) + Synapse
```bash
az storage account create \
  --name datalakestore \
  --enable-hierarchical-namespace true  # ADLS Gen2

# Synapse Serverless SQL
# Query Parquet files in ADLS with SQL—no loading needed
```

```sql
-- Synapse Serverless SQL
SELECT TOP 100 *
FROM OPENROWSET(
    BULK 'https://datalakestore.dfs.core.windows.net/silver/orders/*.parquet',
    FORMAT = 'PARQUET'
) AS orders
WHERE order_date = '2024-06-15';
```

## Summary

| Architecture | Best For | Storage | Query Performance | Governance |
|-------------|----------|---------|------------------|------------|
| Data Warehouse | Structured BI, known questions | $$-$$$ | Fast | Strong |
| Data Lake | Raw, unstructured, ML, exploratory | $ | Variable | Weak (needs governance layer) |
| Data Lakehouse | Best of both: BI + ML on one platform | $ | Fast (with table format) | Strong (with catalog) |

The modern data architecture converges on the Lakehouse pattern: data stored in low-cost object storage (S3/ADLS/GCS) in open formats (Parquet/Iceberg), managed by a table format (Delta Lake/Iceberg/Hudi) providing ACID guarantees, with multiple query engines (Athena, Redshift Spectrum, BigQuery, Synapse, Trino) accessing the same data for different workloads. This eliminates data silos, reduces ETL duplication, and enables both BI analysts and ML engineers to work from the same single source of truth.
