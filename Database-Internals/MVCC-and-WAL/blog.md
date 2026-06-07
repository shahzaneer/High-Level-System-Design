# MVCC & WAL

MVCC (Multi-Version Concurrency Control) and WAL (Write-Ahead Logging) are the two foundational mechanisms that make modern databases reliable and concurrent. MVCC allows readers to never block writers and writers to never block readers—each transaction sees a consistent snapshot of the database at the moment it began. WAL ensures durability: even if the database crashes, committed transactions survive because they were written to the WAL first.

PostgreSQL, MySQL InnoDB, Oracle, and CockroachDB all implement variations of MVCC. Understanding these internals explains isolation levels, vacuum/bloat, replication, and performance characteristics that directly impact application design.

## MVCC: How It Works

```
INSTEAD OF: "UPDATE row → overwrite in place"
MVCC DOES: "UPDATE row → insert new version, mark old version obsolete"

Table: orders (id, customer_id, total, xmin, xmax)
  xmin = transaction ID that created this version
  xmax = transaction ID that deleted/updated this version (NULL = still visible)

Initial state (tx 100 inserts):
  | id | customer_id | total | xmin | xmax |
  |  1 |          42 | 99.99 |  100 | NULL |

Tx 101 updates total:
  |  1 |          42 | 99.99 |  100 |  101 |  ← Old version (marked obsolete)
  |  1 |          42 | 89.99 |  101 | NULL |  ← New version

Tx 101 commits: new version becomes visible to future transactions
Tx 101 rolls back: new version xmax = 101; old version xmax = NULL (restored)

Concurrent readers:
  Tx 102 (started before Tx 101 commit): sees old version (total=99.99)
  Tx 103 (started after Tx 101 commit): sees new version (total=89.99)
  → Readers never block writers; writers never block readers
```

### Transaction Isolation Levels

```sql
-- Read Uncommitted (PostgreSQL treats as Read Committed)
-- Can see uncommitted changes from other transactions (dirty reads)

-- Read Committed (PostgreSQL default)
-- Each statement sees snapshot of committed data at statement start
-- Phantom reads possible: same query returns different rows within transaction

-- Repeatable Read (MySQL InnoDB default)
-- Each transaction sees snapshot of committed data at transaction start
-- Prevents non-repeatable reads; phantom reads possible (MySQL prevents)

-- Serializable
-- Transactions execute as if they ran one at a time
-- PostgreSQL: Serializable Snapshot Isolation (SSI)
-- Highest isolation, lowest concurrency
```

```sql
-- Check current isolation level
SHOW default_transaction_isolation;  -- PostgreSQL
SELECT @@transaction_isolation;       -- MySQL

-- Set isolation level
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
-- All reads in this transaction see data as it existed at this moment
SELECT * FROM orders WHERE customer_id = 42;  -- Returns 5 orders
-- Even if another transaction inserts/deletes, this transaction still sees 5
SELECT * FROM orders WHERE customer_id = 42;  -- Still returns 5
COMMIT;
```

## Vacuum and Bloat (PostgreSQL)

MVCC creates "dead tuples"—old row versions no longer visible to any transaction. VACUUM reclaims this space.

```sql
-- Check bloat
SELECT schemaname, tablename, 
       n_dead_tup, n_live_tup,
       round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;

-- Manual vacuum
VACUUM orders;              -- Marks space reusable; doesn't return to OS
VACUUM FULL orders;         -- Rewrites table; returns space to OS; LOCKS table
VACUUM ANALYZE orders;      -- Vacuum + update statistics

-- Autovacuum (automatic, enabled by default)
-- PostgreSQL auto-vacuums when dead tuples > threshold
```

## WAL: Write-Ahead Logging

```
WAL guarantees durability:
  1. Before modifying data files, write change to WAL
  2. WAL write is sequential (fast) vs data file writes (random)
  3. After crash: replay WAL from last checkpoint → recover to consistent state

WAL entry for INSERT:
  LSN: 0/ABCD1234
  Transaction: 105
  Operation: INSERT
  Table: orders
  New row: (id=1, customer_id=42, total=99.99, ...)

WAL serves multiple purposes:
  - Crash recovery (durability)
  - Point-in-time recovery (PITR): replay WAL to specific moment
  - Streaming replication: standby servers replay WAL in near real-time
  - Logical replication / CDC: decode WAL into logical changes
```

```sql
-- Check WAL settings
SHOW wal_level;         -- replica (default), logical
SHOW max_wal_size;      -- Max WAL size before checkpoint
SHOW checkpoint_timeout;

-- WAL archiving for PITR
-- postgresql.conf:
-- archive_mode = on
-- archive_command = 'aws s3 cp %p s3://backups/wal/%f'
```

## Write Amplification from Indexes

```python
# Single UPDATE can trigger multiple WAL writes:
db.execute("UPDATE orders SET status = 'shipped' WHERE id = 123")

# WAL writes generated:
# 1. Main table: mark old row obsolete (xmax = tx_id)
# 2. Main table: insert new row (xmin = tx_id)
# 3. Primary key index: update pointer
# 4. idx_orders_status: remove from 'pending', add to 'shipped'
# 5. idx_orders_customer: update if relevant columns changed
# 6. Any other indexes on the table
# 
# More indexes = more WAL = more write latency = more replication lag
```

## Why This Matters for Architects

- **Long-running transactions prevent vacuum**: An uncommitted transaction left open holds back xmin horizon. Dead tuples pile up. Table bloat → slow queries. Always set `idle_in_transaction_session_timeout`.
- **Indexes slow writes**: Each index adds to WAL volume and write latency. Index only what queries actually use. Monitor unused indexes.
- **Replication lag = WAL replay lag**: Standby must replay every WAL entry. Heavy write load + many indexes = lag. Design for this.
- **Isolation level trade-off**: Repeatable Read prevents anomalies but holds back vacuum. Choose based on consistency requirements vs operational impact.

MVCC enables concurrent reads and writes without locks. WAL ensures data survives crashes. Together, they are the foundation of modern database reliability.
