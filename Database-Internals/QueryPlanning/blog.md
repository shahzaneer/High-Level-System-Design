# Query Planning & EXPLAIN

The Query Planner is the database's optimizer—it decides HOW to execute a query among many possible strategies. The same SQL can be executed via sequential scan, index scan, bitmap scan, nested loop join, hash join, or merge join. The planner estimates the cost of each strategy and picks the cheapest. Understanding EXPLAIN output is the essential skill for diagnosing and fixing slow queries.

## How the Planner Works

```
1. Parse SQL → syntax tree
2. Rewrite → apply rules (view expansion, constant folding)
3. Plan → generate possible execution strategies
4. Optimize → estimate cost (CPU + I/O) for each strategy, pick minimum
5. Execute → run the chosen plan

The cost model is based on:
- Table statistics (row count, column distribution, most common values)
- Index availability
- Estimated rows returned at each step
- I/O cost (reading pages from disk)
- CPU cost (processing rows)
```

## Reading EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT o.order_id, o.total, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
  AND o.created_at > '2024-01-01'
ORDER BY o.created_at DESC
LIMIT 20;
```

```
QUERY PLAN (read from innermost to outermost):
Limit  (cost=1245.32..1245.37 rows=20 width=68) (actual time=12.456..12.478 rows=20 loops=1)
  → Sort  (cost=1245.32..1248.82 rows=1400 width=68) (actual time=12.454..12.463 rows=20 loops=1)
    Sort Key: o.created_at DESC
    → Hash Join  (cost=89.50..1178.20 rows=1400 width=68) (actual time=1.234..11.890 rows=1400 loops=1)
      Hash Cond: (o.customer_id = c.id)
      → Index Scan using idx_orders_status_date on orders o  (cost=0.42..883.50 rows=1400 width=40) (actual time=0.023..5.678 rows=1400 loops=1)
        Index Cond: (status = 'pending' AND created_at > '2024-01-01')
      → Hash  (cost=60.30..60.30 rows=1530 width=36) (actual time=1.180..1.181 rows=1530 loops=1)
        → Seq Scan on customers c  (cost=0.00..60.30 rows=1530 width=36) (actual time=0.012..0.678 rows=1530 loops=1)
Planning Time: 0.345 ms
Execution Time: 12.543 ms
```

**Key fields**:
- `cost=x..y`: Startup cost (x) and total cost (y) in arbitrary planner units
- `actual time=x..y`: Actual milliseconds (only with ANALYZE)
- `rows=X`: Estimated rows (planner) vs actual rows (ANALYZE)
- `loops=N`: How many times this node executed (1 = not in subquery)

**Red flags**:
- `estimated rows << actual rows` = stale statistics → `ANALYZE table;`
- `Seq Scan` on large table with WHERE clause = missing index
- `Nested Loop` with large inner table = should be Hash Join

## Common Scan Types

```sql
-- Sequential Scan: reads EVERY row. Good for small tables, terrible for large
Seq Scan on orders

-- Index Scan: reads index, then reads table rows
Index Scan using idx_orders_customer on orders

-- Index Only Scan: reads index only (no table access needed) - FASTEST
Index Only Scan using idx_orders_cover on orders

-- Bitmap Scan: combines multiple indexes (reads all matching index entries, 
--   sorts by physical location, reads table in order—good for many rows)
Bitmap Index Scan on idx_orders_status
Bitmap Index Scan on idx_orders_date
BitmapOr → Bitmap Heap Scan on orders
```

## Common Join Types

```sql
-- Nested Loop: For each row in outer table, scan inner table. 
--   Good when outer is small + inner has index.
Nested Loop
  → outer (small result)
  → Index Scan on inner using join_key

-- Hash Join: Build hash table from smaller table, probe with larger.
--   Good for medium/large tables.
Hash Join
  Hash Cond: (o.customer_id = c.id)
  → Seq Scan on orders (large)
  → Hash → Seq Scan on customers (small)

-- Merge Join: Sort both tables on join key, merge sorted streams.
--   Good when both tables already sorted (e.g., by index).
Merge Join
  Merge Cond: (o.customer_id = c.id)
  → Index Scan using idx_orders_customer
  → Index Scan using customers_pkey
```

## Fixing Bad Plans

```sql
-- 1. Fix stale statistics
ANALYZE orders;
ANALYZE customers;

-- 2. Add missing index (look for Seq Scan on large filtered table)
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- 3. Rewrite query to be index-friendly
-- Bad: function prevents index
SELECT * FROM orders WHERE DATE(created_at) = '2024-06-15';
-- Good: range condition uses index
SELECT * FROM orders WHERE created_at >= '2024-06-15' AND created_at < '2024-06-16';

-- 4. Materialized views for expensive aggregations
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT DATE(created_at) as date, SUM(total) as revenue
FROM orders GROUP BY DATE(created_at);
CREATE INDEX ON daily_revenue(date);
REFRESH MATERIALIZED VIEW daily_revenue;  -- Refresh periodically
```

## Summary

| EXPLAIN Finding | Likely Problem | Fix |
|----------------|---------------|-----|
| Seq Scan on large table | Missing index | Create index on WHERE/JOIN columns |
| Estimated rows << actual | Stale statistics | ANALYZE table |
| Nested Loop on large tables | Should be Hash Join | Check if join column is indexed |
| High planning time | Too complex query | Simplify, use views, update stats |
| Repeated Seq Scans in subquery | Inefficient subquery | Rewrite as JOIN or CTE |

Reading EXPLAIN is the database equivalent of reading a profiler for application code. The architect must be able to identify the slow part of a query plan and determine whether it needs an index, updated statistics, or a rewritten query.
