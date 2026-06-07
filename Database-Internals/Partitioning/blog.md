# Database Partitioning

Partitioning splits a large table into smaller, more manageable pieces while presenting a single logical table to applications. The database routes queries to the relevant partitions automatically (partition pruning). The two main strategies: horizontal partitioning (split rows across partitions—what most people mean by "partitioning") and sharding (horizontal partitioning across multiple database instances).

Partitioning solves operational problems: queries scan only relevant partitions, bulk deletes become partition drops, and old data can be moved to cheaper storage. It does NOT solve write throughput scaling for a single database—that requires sharding.

## Declarative Partitioning (PostgreSQL)

```sql
-- Range partitioning by date (most common)
CREATE TABLE orders (
    order_id UUID,
    customer_id UUID,
    total DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Automatic partition creation (PostgreSQL 17+ or pg_partman extension)
-- Or use pg_partman for auto-partition management
```

```sql
-- List partitioning (by discrete values)
CREATE TABLE orders PARTITION BY LIST (status);
CREATE TABLE orders_active PARTITION OF orders FOR VALUES IN ('pending', 'confirmed');
CREATE TABLE orders_completed PARTITION OF orders FOR VALUES IN ('shipped', 'delivered');
CREATE TABLE orders_cancelled PARTITION OF orders FOR VALUES IN ('cancelled', 'refunded');

-- Hash partitioning (uniform distribution, no logical grouping)
CREATE TABLE orders PARTITION BY HASH (customer_id);
CREATE TABLE orders_p0 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_p1 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_p2 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_p3 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

## Partition Pruning

```sql
-- Query automatically prunes irrelevant partitions
SELECT * FROM orders WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16';
-- Database scans ONLY orders_2024_01 partition (not all partitions)

EXPLAIN SELECT * FROM orders WHERE created_at = '2024-06-15';
-- Shows: "Scan on orders_2024_06" only

-- Partitioning wins when:
--   Table has 500M rows across 2 years
--   Most queries filter by date (last 30 days)
--   Partition pruning means scanning 12M rows instead of 500M
```

## Operational Benefits

```sql
-- Bulk delete old data: DROP partition (instant) vs DELETE (hours)
DROP TABLE orders_2022_01;  -- Instant, no VACUUM needed

-- DELETE FROM orders WHERE created_at < '2022-02-01';  -- Same effect, but hours

-- Detach partition (keep data, remove from logical table)
ALTER TABLE orders DETACH PARTITION orders_2024_01;
-- Now orders_2024_01 is a standalone table; can be archived/exported

-- Move old partitions to cheaper storage (tablespaces)
CREATE TABLESPACE slow_storage LOCATION '/mnt/archive';
ALTER TABLE orders_2022_01 SET TABLESPACE slow_storage;
```

## Sub-Partitioning

```sql
-- Range partition by month, sub-partition by status (hash)
CREATE TABLE orders (
    order_id UUID,
    created_at TIMESTAMP NOT NULL,
    status VARCHAR(20) NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_06 PARTITION OF orders
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01')
    PARTITION BY HASH (status);

CREATE TABLE orders_2024_06_p0 PARTITION OF orders_2024_06
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- ... p1, p2, p3
```

## Partitioning vs Sharding

| Aspect | Partitioning | Sharding |
|--------|-------------|----------|
| Where | Single database instance | Multiple database instances |
| Goal | Manageability, query performance | Write throughput scaling |
| Complexity | Low (built-in SQL) | High (application routing, cross-shard queries) |
| Transaction | ACID across all partitions | No cross-shard ACID |
| Cross-partition queries | Transparent | Must be done at application level |
| When to use | Table > 100GB, most queries date-filtered | Write throughput > single instance capacity |

## Best Practices

1. **Partition by the most common query filter**—usually date
2. **Don't create too many partitions**—10,000+ partitions degrade planning time
3. **Index each partition identically**—partitioning doesn't eliminate the need for indexes
4. **Monitor for partition skew**—one partition 100x bigger than others defeats the purpose
5. **Automate partition creation**—pg_partman or application-level cron jobs
6. **Test partition pruning**—EXPLAIN to verify queries hit only needed partitions

Partitioning is the first step when tables grow beyond manageable size. It improves query performance through partition pruning, simplifies data lifecycle management through partition DROP/DETACH, and enables tiered storage. When a single database instance can no longer handle write throughput even with partitioning, that's the signal to consider sharding.
