# Connection Pooling

Connection pooling is the single most impactful application-level optimization for database performance. Opening a PostgreSQL connection involves TCP handshake, TLS negotiation, authentication, and backend process forking—50-100ms of work. Doing this for every request kills performance. Connection pooling reuses established connections across requests, reducing connection overhead to near zero.

The pool also protects the database: PostgreSQL spawns one OS process per connection. 10,000 connections = 10,000 processes = context switching hell. A connection pool limits application connections to a manageable number (typically 20-100), multiplexing thousands of application requests through a small set of persistent database connections.

## Server-Side Pooling (PgBouncer)

```ini
# pgbouncer.ini
[databases]
orders = host=db.internal.com port=5432 dbname=orders

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Pool mode: transaction (recommended for most apps)
pool_mode = transaction

# Pool size
default_pool_size = 25
max_client_conn = 500
max_db_connections = 25

# Timeouts
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 30
```

### PgBouncer Pool Modes

| Mode | When connection returned to pool | Best for |
|------|--------------------------------|----------|
| Session | When client disconnects | Stateful sessions (SET, temp tables) |
| Transaction | After each transaction (COMMIT/ROLLBACK) | Most web apps (stateless between transactions) |
| Statement | After each statement | Read-only, fully stateless |

```
Transaction pooling is the sweet spot:
  - Connection reused after each transaction
  - 500 clients → 25 database connections
  - Works with most applications
  - CAVEAT: prepared statements, SET, LISTEN/NOTIFY break with transaction pooling
```

## Client-Side Pooling (Application-Level)

### Python (SQLAlchemy)

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql://user:pass@pgbouncer:6432/orders',
    poolclass=QueuePool,
    pool_size=10,              # Persistent connections
    max_overflow=20,           # Additional connections when pool exhausted
    pool_timeout=30,           # Wait max 30s for connection
    pool_recycle=3600,         # Recycle connections older than 1 hour
    pool_pre_ping=True,        # Verify connection before using (handles stale)
)
```

### Java (HikariCP)

```yaml
spring.datasource.hikari:
  maximumPoolSize: 20
  minimumIdle: 5
  idleTimeout: 600000          # 10 minutes
  maxLifetime: 1800000         # 30 minutes
  connectionTimeout: 30000     # 30 seconds
  validationTimeout: 5000
  leakDetectionThreshold: 60000 # Log connections held > 60 seconds
```

## Connection Pool Sizing

```
Formula: pool_size = (core_count * 2) + effective_spindle_count

PostgreSQL (CPU-bound):
  4 cores → pool_size = 9 (4*2 + 1)
  8 cores → pool_size = 17
  16 cores → pool_size = 33

MySQL (can handle more):
  4 cores → pool_size = 15-25
  8 cores → pool_size = 25-50

BUT: modern SSDs change this. Start with:
  pool_size = 20 per application instance
  Monitor: if connection wait time > 1ms, increase
           if database CPU < 50%, connections are not the bottleneck

PostgreSQL max_connections = 100 (default)
  Keep application connections at 50-80% of this
  Heavy connection usage → increase max_connections (requires restart)
```

## Architecture Patterns

### Two-Tier Pooling (Application + PgBouncer)

```
[App Instance 1]  ──pool(10)──┐
[App Instance 2]  ──pool(10)──┤
[App Instance 3]  ──pool(10)──┼── [PgBouncer: pool(25)] ── [PostgreSQL: max_conn=50]
[App Instance 4]  ──pool(10)──┤
[App Instance 5]  ──pool(10)──┘

40 app connections → PgBouncer → 25 database connections
Even with auto-scaling to 20 instances, database connections stay at 25
```

### Sidecar PgBouncer (Kubernetes)

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: order-service:v1
    env:
    - name: DB_HOST
      value: "localhost"    # Connect to local PgBouncer
    - name: DB_PORT
      value: "6432"
  
  - name: pgbouncer
    image: pgbouncer/pgbouncer:latest
    ports:
    - containerPort: 6432
    volumeMounts:
    - name: pgbouncer-config
      mountPath: /etc/pgbouncer
```

## Pool Exhaustion (Troubleshooting)

```sql
-- PostgreSQL: check active connections
SELECT count(*), state FROM pg_stat_activity 
WHERE datname = 'orders' GROUP BY state;

-- idle in transaction (RED FLAG): application started transaction,
--   did work, never COMMIT. Connection is held.
SELECT pid, query_start, state, query 
FROM pg_stat_activity 
WHERE state = 'idle in transaction' 
  AND query_start < NOW() - INTERVAL '5 minutes';

-- Fix: set idle_in_transaction_session_timeout
ALTER DATABASE orders SET idle_in_transaction_session_timeout = '5min';
```

## Summary

| Pooling Level | Tool | Best For |
|--------------|------|----------|
| Application | HikariCP, SQLAlchemy pool | Reducing connect overhead per request |
| Server-side | PgBouncer, Pgpool-II | Multiplexing many apps to few DB connections |
| Sidecar | PgBouncer per pod | Kubernetes; local fast connections |
| Cloud-managed | RDS Proxy, Cloud SQL Proxy | AWS/GCP/Azure managed pooling |

Connection pooling is a non-negotiable production requirement. Direct application-to-database connections without pooling cause: slow requests (connection overhead), database overload (too many processes), and cascade failures (connection storms during traffic spikes). The recommended architecture: application-level pool (fast local reuse) + server-side pool (PgBouncer, limits total database connections) for robust, production-grade connection management.
