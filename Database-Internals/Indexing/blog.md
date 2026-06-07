# Database Indexing

Indexes are the single most impactful performance optimization for databases. They turn O(n) full table scans into O(log n) index lookups. A query that scans 10 million rows might read just 3-4 index pages with the right index. The cost: indexes consume storage (typically 10-30% of table size) and slow down writes (every INSERT/UPDATE must also update indexes).

The art of indexing is choosing which queries to optimize (based on frequency and impact), which columns to index, and which index type to use. Wrong indexes waste space and slow writes. Missing indexes cause production outages when data grows beyond what full scans can handle.

## B-Tree Index (Default, Most Common)

```
B-TREE STRUCTURE (PostgreSQL, MySQL InnoDB):
  Balanced tree where each node contains ordered key values.
  All leaf nodes are at the same depth.
  
       [100, 200, 300]         ← Root node (keys)
      /    |     |    \
  [50,80] [150] [220,250] [350]  ← Internal nodes
  /  |  \   |    /  |  \    |
  ...leaf nodes with actual row pointers...

Lookup: WHERE id = 150
  1. Root: 100 ≤ 150 < 200 → follow pointer between 100 and 200
  2. Internal node: find 150 → follow pointer
  3. Leaf: read the actual row
  Total: 3 page reads (vs millions for full scan)
```

```sql
-- Standard B-Tree index
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Composite index (multi-column)
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);

-- Partial index (index only rows matching condition)
CREATE INDEX idx_pending_orders ON orders(created_at) WHERE status = 'pending';

-- Covering index (include extra columns to avoid table lookup)
CREATE INDEX idx_orders_cover ON orders(customer_id) INCLUDE (total, status);

-- Unique index (enforces uniqueness + indexing)
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

## Composite Index Column Order Matters

```sql
-- Index: (customer_id, status, created_at)

-- CAN use index (matches leftmost columns):
SELECT * FROM orders WHERE customer_id = 42;                              -- Uses (customer_id)
SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending';      -- Uses (customer_id, status)
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at;         -- Uses all three columns

-- CANNOT use index (skips leftmost column):
SELECT * FROM orders WHERE status = 'pending';                           -- No customer_id = no index!
SELECT * FROM orders WHERE created_at > '2024-01-01';                    -- No customer_id = no index!
```

Rule: Leading columns must appear in WHERE clause for the index to be used. Order columns by: most frequently filtered → most selective → range queries LAST.

## Hash Index

```sql
-- PostgreSQL Hash index: ONLY for equality (=), not ranges (<, >, BETWEEN)
CREATE INDEX idx_orders_id_hash ON orders USING HASH (order_id);

-- Fastest for: SELECT * FROM orders WHERE order_id = 'abc-123'
-- Useless for: SELECT * FROM orders WHERE order_id > 'abc-123'
-- Smaller than B-Tree; use when you ONLY do equality lookups
```

## GIN (Generalized Inverted Index)

```sql
-- For: arrays, JSONB, full-text search
CREATE INDEX idx_orders_items ON orders USING GIN (items);

-- Query array containment:
SELECT * FROM orders WHERE items @> ARRAY['SKU-001'];  -- Orders containing SKU-001

-- Full-text search:
CREATE INDEX idx_products_fts ON products USING GIN (to_tsvector('english', description));
SELECT * FROM products WHERE to_tsvector('english', description) @@ to_tsquery('laptop & gaming');
```

## GiST (Generalized Search Tree)

```sql
-- For: geometric data, range types, full-text (alternative to GIN)
CREATE INDEX idx_locations ON stores USING GiST (location);
SELECT * FROM stores WHERE location <@ box '(0,0),(100,100)';
```

## When Indexes Don't Work

```sql
-- Functions on indexed column PREVENT index usage!
CREATE INDEX idx_orders_date ON orders(created_at);

-- Index NOT used:
SELECT * FROM orders WHERE DATE(created_at) = '2024-06-15';  -- Function on column

-- Index USED:
SELECT * FROM orders WHERE created_at >= '2024-06-15' AND created_at < '2024-06-16';

-- LIKE with leading wildcard PREVENTS index usage:
SELECT * FROM users WHERE email LIKE '%@gmail.com';  -- Index NOT used
SELECT * FROM users WHERE email LIKE 'alice%';       -- Index USED

-- Negation often skips index:
SELECT * FROM orders WHERE status != 'cancelled';    -- May or may not use index
```

## Checking Index Usage

```sql
-- PostgreSQL: EXPLAIN ANALYZE
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
-- Look for: Index Scan (good) vs Seq Scan (bad)
-- Look for: rows=10 vs rows=1000000 (estimated vs actual mismatch = bad statistics)

-- MySQL:
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- Look for: key='idx_orders_customer' (index used) vs NULL (no index)
```

## Best Practices

1. Index every foreign key column (joins need indexes on both sides)
2. Index columns in WHERE, JOIN, ORDER BY clauses
3. Composite indexes beat multiple single-column indexes for combined queries
4. Don't over-index: each index slows writes. 5-10 indexes per table is typical
5. Monitor: unused indexes waste space and slow writes; drop them
6. Rebuild/reindex periodically for heavily updated tables (index bloat)

Indexing is the highest-leverage database optimization. One well-chosen index can reduce query time from 30 seconds to 3 milliseconds. The architect must understand query patterns to specify indexes, and must verify index usage with EXPLAIN rather than assuming.
