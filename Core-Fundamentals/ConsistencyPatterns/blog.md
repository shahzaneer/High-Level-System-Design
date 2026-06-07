# Consistency Patterns in Distributed Systems

## Introduction
Consistency patterns define the guarantees a distributed system provides about when and how data changes become visible to clients. As systems move from single-node databases to distributed architectures, maintaining the illusion of a single, coherent copy of data becomes a central challenge. Different applications tolerate different levels of consistency, and the choice profoundly impacts system architecture, performance, scalability, and user experience.

The spectrum ranges from Strong Consistency (every read sees the latest write) to Eventual Consistency (reads may see stale data temporarily), with several practical middle-ground patterns used by real-world systems. Understanding these patterns is essential for choosing the right database, designing replication strategies, and predicting system behavior during failures.

## Definition
**Consistency** in distributed systems refers to the guarantee that all nodes (or replicas) see the same data at the same time (or within a bounded timeframe). It's distinct from the "C" in ACID (which refers to transaction invariants, not replication). In the context of distributed data stores, consistency patterns include:

- **Strong Consistency**: After a write completes, all subsequent reads will see that write (or a later one). The system behaves as if there's a single copy of data.
- **Eventual Consistency**: If no new writes are made, eventually all reads will return the same value. During normal operation, different replicas may return different values.
- **Causal Consistency**: Writes that are causally related must be seen in the same order by all nodes. Writes that are not causally related (concurrent) may be seen in different orders.
- **Read-Your-Writes Consistency**: A client always sees its own writes immediately, even if reads from other replicas might return stale data.
- **Monotonic Reads**: A client never sees data go "backward in time"—once it sees a value, subsequent reads never return an older value.

## Concept Explanation

### The Consistency Spectrum

```
Strong ──────────────────────────────────────────────────── Eventual
  │                                                           │
  │  Bounded Staleness    Causal     Read-Your-Writes        │
  │       │                 │              │                    │
  ▼       ▼                 ▼              ▼                    ▼
Linearizable   │           Session       Monotonic       Eventually
Consistency    │         Consistency      Reads         Consistent
```

### Strong Consistency (Linearizability)

The strongest guarantee: the system behaves as if there's exactly one copy of data, and all operations are atomic. Once a write completes, all readers see it. Reads never return stale data. This is the "gold standard" but comes with performance costs.

```
Timeline:
t0: Write(x=1) → Acknowledged
t1: Read(x) from Node A → 1
t2: Read(x) from Node B → 1  (guaranteed)
t3: Read(x) from Node C → 1  (guaranteed)
```

Implementation requires consensus algorithms (Paxos, Raft) or synchronous replication. Every write must be acknowledged by a majority of replicas before being considered committed.

```python
# Strongly consistent read in DynamoDB
response = dynamodb.get_item(
    TableName='Orders',
    Key={'OrderID': 'ORD-001'},
    ConsistentRead=True  # reads from leader, guarantees latest write
)
```

**Use cases**: Financial transactions, inventory management, user authentication (you must see the password change immediately after setting it)

### Eventual Consistency

The weakest guarantee: if no new writes occur, eventually all replicas converge to the same value. During normal operation, reads may return stale data. The system prioritizes availability and performance.

```
Timeline:
t0: Write(x=1) → Acknowledged
t1: Read(x) from Node A → 1  (new value, happened to read updated replica)
t2: Read(x) from Node B → 0  (stale value, replica not yet updated)
t3: (replication completes)
t4: Read(x) from Node B → 1  (converged)
```

```python
# Eventually consistent read in DynamoDB (default)
response = dynamodb.get_item(
    TableName='Orders',
    Key={'OrderID': 'ORD-001'},
    ConsistentRead=False  # default, may return stale data
)
```

**Use cases**: Social media feeds, product recommendations, analytics dashboards, DNS

### Bounded Staleness

A middle ground: reads may return stale data, but the staleness has a known bound (e.g., "at most 5 seconds behind"). Azure Cosmos DB offers bounded staleness as an explicit consistency level.

```
Guarantee: Any read will return data no more than T seconds or K versions behind
```

```python
# Azure Cosmos DB bounded staleness (configurable at account level)
# Staleness bounded by either time (seconds) or version count
consistency_policy = {
    'defaultConsistencyLevel': 'BoundedStaleness',
    'maxStalenessPrefix': 100,       # max 100 versions behind
    'maxIntervalInSeconds': 5        # max 5 seconds behind
}
```

### Causal Consistency

Preserves the "happens-before" relationship between operations. If operation A causally precedes operation B (B depends on or follows from A), then all nodes see A before B. Unrelated concurrent operations can be seen in any order.

```
If Alice posts a photo (A), Bob comments on it (B):
  B causally depends on A → everyone sees A before B

If simultaneously, Charlie posts a photo (C), and Dave posts a photo (D):
  C and D are concurrent → different nodes may see them in different orders
  But within each photo, comments still appear after the photo
```

Causal consistency is the strongest consistency achievable without sacrificing availability during network partitions (it's the strongest "available" consistency in the sense of CAP).

**Implementation**: Vector clocks track causality. Each node maintains a vector of counters, one per node, representing the logical clock of each node as known to this node.

```python
# Vector clock example
class VectorClock:
    def __init__(self):
        self.clock = {}  # {node_id: counter}
    
    def increment(self, node_id):
        self.clock[node_id] = self.clock.get(node_id, 0) + 1
    
    def merge(self, other_clock):
        for node, counter in other_clock.items():
            self.clock[node] = max(self.clock.get(node, 0), counter)
    
    def happens_before(self, other_clock):
        """Check if this clock causally precedes other_clock"""
        at_least_one_less = False
        for node in set(self.clock.keys()) | set(other_clock.keys()):
            a = self.clock.get(node, 0)
            b = other_clock.get(node, 0)
            if a > b:
                return False
            if a < b:
                at_least_one_less = True
        return at_least_one_less

# Usage
vc1 = VectorClock()
vc1.increment('A')
# {A: 1}

vc2 = VectorClock()
vc2.clock = vc1.clock.copy()  # B's version depends on A's version
vc2.increment('B')
# {A: 1, B: 1} → vc1 happens-before vc2 (True)
```

### Read-Your-Writes (Read-Your-Own-Writes) Consistency

A client always sees its own writes immediately. This is a session-level guarantee. Doesn't require all replicas to agree—just that the replica serving the client has seen the client's writes.

```
Alice writes her profile bio:
  t0: Write(bio="Hello World") → Acknowledged
  t1: Read(bio) from same session → "Hello World"  (guaranteed)

Bob reads Alice's profile from a different replica:
  t2: Read(bio) → may still see old value  (not guaranteed for Bob)
```

Implementation strategies:
- Route reads for recently modified data to the leader (primary)
- Track last write timestamp and read from leader if within staleness window
- Client includes its last write timestamp; replica waits until it has caught up

```python
class SessionConsistentClient:
    def __init__(self, db):
        self.db = db
        self.last_write_time = 0
        self.written_keys = set()
    
    def write(self, key, value):
        self.db.put(key, value)
        self.last_write_time = time.time()
        self.written_keys.add(key)
    
    def read(self, key):
        if key in self.written_keys and \
           time.time() - self.last_write_time < 5:
            # Read from leader to guarantee read-your-writes
            return self.db.leader_read(key)
        return self.db.read(key)  # may go to any replica
```

### Monotonic Reads

A client never sees data go "backward in time." If a client reads value V1, then later reads the same key, it will never see a value older than V1. Prevents the confusing experience of refreshing a page and seeing data disappear.

```
Without monotonic reads:
  t0: Read(x) → 2  (saw updated value from replica A)
  t1: Read(x) → 1  (switched to replica B which is behind → confusing!)

With monotonic reads:
  t0: Read(x) → 2
  t1: Read(x) → 2 or 3 (never 1)
```

Implementation: Route a given user's reads to the same replica (sticky sessions) or track the highest version seen and ensure subsequent reads are at least that version.

### Consistent Prefix Reads

Guarantees that reads see a causally consistent snapshot—specifically, that the sequence of writes is seen in order with no "gaps." If writes occur in order W1, W2, W3, a reader never sees W1 and W3 without W2. This is essential for systems where ordering matters (collaborative documents, chat applications).

## Layman's Explanation

### The Shared Whiteboard Analogy
A team of 5 people in different rooms, each with a copy of the same whiteboard. They can write on their copy and changes propagate to others via a messenger.

**Strong Consistency**: Every time someone writes, the messenger runs to ALL rooms and updates everyone's whiteboard before the writer can write again. You always see the latest. Slow if rooms are far apart. This is like a bank—the balance must be correct for every teller, every time.

**Eventual Consistency**: People write freely. The messenger delivers updates at their own pace. For a while, different rooms see different things. Eventually, all whiteboards match. This is like social media—different people might see different like counts for a few seconds. Nobody dies.

**Causal Consistency**: If Alice writes "Meeting at 3pm" and Bob writes "+1" underneath it, Bob's "+1" always appears under Alice's message, never before it, regardless of which room you're in. But if Charlie writes "Meeting at 4pm" independently in another room, that might appear in any position relative to Alice's message.

**Read-Your-Writes**: Alice always sees "Meeting at 3pm" on HER whiteboard immediately after writing it. She doesn't need to wait for the messenger to come back.

**Monotonic Reads**: Once Alice sees "Meeting at 3pm" with 5 attendees, refreshing her whiteboard never shows 3 attendees again. The count only goes up.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **User Experience Impact**: The consistency level directly affects what users see. A banking app showing wrong balances (weak consistency) is catastrophic. A Twitter feed showing slightly stale follower counts is acceptable.
- **Database Selection**: Different databases offer different consistency models. MongoDB defaults to eventual consistency (secondary reads). CockroachDB provides serializable isolation (strong). DynamoDB offers tunable consistency per request. The architect must match the database to the use case's consistency requirements.
- **CAP Theorem Trade-offs**: During network partitions, you choose between consistency and availability. Understanding the consistency spectrum helps make nuanced decisions—you rarely need absolute strong consistency for everything. Different parts of the same application often need different guarantees.
- **Microservice Data Ownership**: When each service owns its data, cross-service data views are eventually consistent. The architect must design for this—CQRS with eventual consistency between command and query models, or sagas for distributed transactions.

### Business Impact
- **Revenue**: Amazon's study showed every 100ms of latency cost 1% in sales. Strong consistency adds latency (waiting for quorum). Choosing eventual consistency where appropriate directly impacts revenue.
- **Compliance**: GDPR's "right to erasure" requires that deleted data stops appearing. With eventual consistency, a deleted record might still be visible from stale replicas for seconds or minutes. This has legal implications.
- **Customer Trust**: Apps that show inconsistent data erode user trust. A user who changes their email, refreshes, and sees the old email thinks the app is broken. Read-your-writes consistency is the minimum acceptable for user-facing applications.
- **Operational Simplicity**: Strongly consistent systems are simpler to reason about. Eventually consistent systems require application-level conflict handling, which adds development and debugging complexity.

### Decision Framework
Ask these questions for each data access pattern:
1. Can the user tolerate seeing stale data? For how long?
2. Will inconsistency cause financial or legal harm?
3. Do writes need to be visible across geographic regions? What latency is acceptable?
4. Can concurrent writes to the same data happen? If so, how should they be resolved?

## On-Premises Examples

### PostgreSQL (Strong Consistency)
Synchronous streaming replication for strong consistency across replicas:

```sql
-- Primary: require synchronous acknowledgment from at least 1 standby
ALTER SYSTEM SET synchronous_commit = 'remote_apply';
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (replica1, replica2)';

-- Now every commit waits for at least one standby to apply the change
-- before acknowledging to the client → strong consistency
INSERT INTO orders (customer_id, total) VALUES (42, 99.99);
COMMIT;  -- blocks until standby confirms
```

### Apache Cassandra (Tunable Consistency)
Per-query consistency level allows choosing the strength per operation:

```sql
-- Strong consistency: read+write quorum (R + W > RF)
-- With Replication Factor 3:
CONSISTENCY QUORUM;  -- requires 2 of 3 replicas
SELECT * FROM users WHERE user_id = 42;
-- Guarantees latest write because R(2) + W(2) = 4 > RF(3)

-- Eventual consistency:
CONSISTENCY ONE;  -- read from any single replica
SELECT * FROM users WHERE user_id = 42;
-- May return stale data, but fastest and always available
```

### MongoDB (Tunable Read Preference + Write Concern)
```javascript
// Write concern: majority (strong consistency for writes)
db.orders.insertOne(
  { orderID: "ORD-001", status: "confirmed" },
  { writeConcern: { w: "majority", j: true } }
);

// Read preference: primary (strong consistency for reads)
db.orders.find({}).readPref("primary");

// Read preference: nearest (eventual consistency, lowest latency)
db.orders.find({}).readPref("nearest");
```

### Redis (Strong Consistency via WAIT)
```bash
# Redis synchronous replication with WAIT command
SET order:123 "confirmed"
WAIT 1 1000  # wait for 1 replica to acknowledge within 1000ms
# Combines async replication with synchronous acknowledgement per-command
```

## AWS Examples

### DynamoDB Consistency Models
```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# Eventually consistent read (default, half the cost)
response = table.get_item(
    Key={'OrderID': 'ORD-001'}
)

# Strongly consistent read (2x cost, reads from leader)
response = table.get_item(
    Key={'OrderID': 'ORD-001'},
    ConsistentRead=True
)

# DynamoDB Global Tables: multi-region eventually consistent
# Automatic conflict resolution: last-write-wins
table_asia = boto3.resource('dynamodb', region_name='ap-southeast-1').Table('Orders')
table_asia.put_item(Item={'OrderID': 'ORD-001', 'Status': 'shipped'})
# Eventually visible in other regions
```

### S3 Consistency Model
```python
# S3 now provides strong read-after-write consistency for all operations
# (as of December 2020)
s3.put_object(Bucket='my-bucket', Key='file.txt', Body='hello')
response = s3.get_object(Bucket='my-bucket', Key='file.txt')
# Guaranteed to return 'hello' immediately
```

### Aurora Global Database
```python
# Cross-region replication with bounded staleness (~1 second typical)
# Primary region (us-east-1): reads and writes
# Secondary region (eu-west-1): reads only, typically <1 second behind

# Managed replication lag typically < 1 second
# Useful for disaster recovery and local reads with bounded staleness
```

## GCP Examples

### Cloud Spanner (External Consistency)
Strongest consistency model available—stronger than serializable:

```sql
-- Cloud Spanner: externally consistent reads
-- Guarantees: if T1 commits before T2 starts, T2 sees T1's writes
-- Requires TrueTime API (atomic clocks + GPS synchronized across data centers)

-- Read-write transaction (strong consistency for both reads and writes)
BEGIN TRANSACTION;
SELECT status FROM Orders WHERE OrderID = 'ORD-001';
UPDATE Orders SET status = 'confirmed' WHERE OrderID = 'ORD-001';
COMMIT;

-- Read-only transaction (can specify staleness for performance)
SELECT * FROM Orders
  WHERE OrderID = 'ORD-001'
  WITH (exact_staleness='10s');  -- bounded staleness: at most 10 seconds behind
```

### Firestore Consistency Models
```python
from google.cloud import firestore

db = firestore.Client()

# Strongly consistent reads (default in Firestore)
doc = db.collection('orders').document('ORD-001').get()
# Always returns latest data

# Real-time listeners with strong consistency
def on_snapshot(doc_snapshot, changes, read_time):
    for doc in doc_snapshot:
        print(f'Doc: {doc.id} => {doc.to_dict()}')
    print(f'Read time: {read_time}')  # consistent timestamp

db.collection('orders').on_snapshot(on_snapshot)
```

### Cloud Datastore (Eventually Consistent by Default)
```python
from google.cloud import datastore

client = datastore.Client()

# Eventually consistent query (default)
query = client.query(kind='Orders')
query.add_filter('status', '=', 'pending')
results = list(query.fetch())

# Strongly consistent: ancestor query or key lookup
key = client.key('Customer', 'CUST-789', 'Order', 'ORD-001')
order = client.get(key)  # strongly consistent
```

## Azure Examples

### Cosmos DB (5 Consistency Levels)
```python
from azure.cosmos import CosmosClient, ConsistencyLevel

# Account-level default consistency
client = CosmosClient(
    ENDPOINT, credential=KEY,
    consistency_level=ConsistencyLevel.Session  # default
)

# Override consistency per request (can only relax, not strengthen)
container = database.get_container_client('orders')

# Strong consistency (highest latency, highest cost in RU/s)
container.read_item(
    item='ORD-001',
    partition_key='CUST-789',
    consistency_level=ConsistencyLevel.Strong
)

# Bounded staleness (10 seconds, 100 versions max behind)
container.read_item(
    item='ORD-001',
    partition_key='CUST-789',
    consistency_level=ConsistencyLevel.BoundedStaleness
)

# Eventual consistency (lowest cost, lowest latency)
container.read_item(
    item='ORD-001',
    partition_key='CUST-789',
    consistency_level=ConsistencyLevel.Eventual
)
```

### Azure Cosmos DB Consistency Matrix

| Consistency Level | Reads from | Writes to | Latency | Use Case |
|------------------|------------|-----------|---------|----------|
| Strong | Majority (3/4) | Majority (3/4) | Highest | Financial ledgers |
| Bounded Staleness | 2 replicas | Majority (3/4) | Medium-High | Dashboard with known lag |
| Session | 2 replicas (session) | Majority (3/4) | Medium | User-facing apps |
| Consistent Prefix | 2 replicas | Majority (3/4) | Medium | Event ordering, chat |
| Eventual | Any replica | Majority (3/4) | Lowest | Counters, likes, analytics |

### Azure SQL Database
```sql
-- Read-only replicas are transactionally consistent
-- (Synchronous replication within region)
SELECT * FROM Orders WHERE OrderID = 'ORD-001';
-- Returns committed data, no stale reads from primary replica

-- Active geo-replication (async, eventually consistent)
-- Secondary regions may have replication lag
```

## Summary

| Consistency Pattern | Guarantee | Latency | Availability | Best For |
|-------------------|-----------|---------|-------------|----------|
| Strong / Linearizable | Latest write always visible | Highest | Lower during partitions | Financial, auth, inventory |
| Bounded Staleness | Stale within known time/versions | Medium | Higher | Dashboards, operational views |
| Causal | Causally related writes in order | Medium-Low | Higher | Social media, collaborative editing |
| Read-Your-Writes | Client sees own writes | Low | High | User profile settings, any user-facing write |
| Monotonic Reads | No going backward in time | Low | High | Feeds, timelines, paginated results |
| Eventual | Converges eventually | Lowest | Highest | Analytics, counters, recommendations |

Consistency is not binary (strong vs. eventual)—it's a spectrum. The art of distributed systems design is applying the right consistency model to each data access pattern within the same application. A single application might use strong consistency for payment processing, causal consistency for social features, and eventual consistency for recommendation engines. Understanding this spectrum enables architects to build systems that are both highly available and correctly consistent where it matters.
