# Consensus Algorithms (Raft & Paxos)

## Introduction
Consensus Algorithms solve one of the hardest problems in distributed systems: getting a group of independent nodes to agree on a single value or sequence of operations, even when some nodes fail or messages are lost. This is the foundation for building reliable systems out of unreliable components—a requirement for any production distributed database, lock service, or configuration store.

Leslie Lamport introduced Paxos in 1989, but its complexity led Diego Ongaro and John Ousterhout to create Raft in 2014 with the explicit goal of being understandable. Today, consensus algorithms power the core of etcd (Kubernetes' brain), Apache Zookeeper, Consul, CockroachDB, TiDB, and cloud services like AWS Aurora and Google Spanner.

## Definition
A **Consensus Algorithm** enables a collection of distributed nodes to reliably agree on a single value (or sequence of values in a replicated log) despite node failures, network partitions, and message delays. The algorithm guarantees:

- **Safety**: Nothing bad ever happens. No two nodes decide on different values. Once a decision is made, it is never changed.
- **Liveness**: Something good eventually happens. The algorithm eventually reaches agreement if a majority of nodes are functioning and communicating.

## Concept Explanation

### Why Consensus is Hard

The fundamental challenge: in an asynchronous network (where messages can be arbitrarily delayed), it's theoretically impossible to guarantee consensus in bounded time if even one node can fail (FLP Impossibility Theorem, 1985). Real systems work around this by using timeouts and randomization.

Consider three nodes trying to agree on who leads:
- Node A proposes itself as leader
- Node B proposes itself as leader simultaneously
- They can't hear each other due to a network delay
- Without consensus, you get split-brain (both think they're leader)

### Raft: Understandable Consensus

Raft breaks consensus into three sub-problems:

#### 1. Leader Election
Nodes are in one of three states: **Leader**, **Follower**, or **Candidate**.

```
[Follower] ──timeout, no heartbeat──→ [Candidate] ──wins election──→ [Leader]
                   ↑                                                      │
                   └──────── discovers higher term leader ────────────────┘
```

- All nodes start as followers
- If a follower doesn't hear from a leader within a randomized election timeout (150-300ms), it becomes a candidate, increments its term, and requests votes
- If it receives votes from a majority of nodes, it becomes the leader
- The leader sends heartbeats (AppendEntries) at regular intervals to maintain authority
- If a candidate receives a message from a node with a higher term, it becomes a follower

Leadership is determined by **terms** (monotonically increasing logical clocks). Each term can have at most one leader.

#### 2. Log Replication
The leader accepts client commands, appends them to its log, and replicates to followers:

```
Leader receives: SET x = 42

1. Leader appends to local log: [term=1, index=5, cmd="SET x=42"]
2. Leader sends AppendEntries RPCs to all followers
3. When a majority acknowledges, the entry is "committed"
4. Leader applies the entry to its state machine
5. Leader responds to client
6. Leader notifies followers of commit in subsequent AppendEntries
```

A log entry is **committed** when replicated to a majority. This is the key safety property: a committed entry will never be lost or overwritten, even after leader changes.

#### 3. Safety
Raft enforces safety through additional rules:

- **Election Restriction**: A candidate must have all committed entries in its log. Voters won't vote for candidates with less up-to-date logs.
- **Commitment from Current Term**: A leader can't commit entries from previous terms by counting replicas—it must commit an entry from its own current term first, which implicitly commits all prior entries.
- **Leader Append-Only**: Leaders never overwrite or delete entries in their own logs. Followers overwrite conflicting entries to match the leader.

### Paxos (The Classic)

Paxos is more general but harder to understand. It separates consensus into roles:

- **Proposers**: Propose values
- **Acceptors**: Accept or reject proposals
- **Learners**: Learn the agreed-upon value

Paxos uses two-phase voting:

**Phase 1 (Prepare)**:
- Proposer selects a proposal number N and sends Prepare(N) to a majority of acceptors
- Each acceptor promises to reject proposals with numbers less than N and responds with the highest-numbered proposal it has accepted (if any)

**Phase 2 (Accept)**:
- If proposer receives promises from a majority, it sends Accept(N, V) where V is either the value from the highest-numbered response or its own value
- If acceptors receive Accept(N, V) and haven't promised to reject N, they accept it
- Value is chosen when a majority accepts

### Paxos vs Raft

| Aspect | Paxos | Raft |
|--------|-------|------|
| Understandability | Very difficult | Designed for understandability |
| Leader role | Not explicit (multi-proposer) | Strong leader model |
| Log replication | Separate Multi-Paxos protocol | Built-in via AppendEntries |
| Membership changes | Complex | Joint consensus, simple |
| Implementations | Chubby, Spanner | etcd, Consul, CockroachDB, TiKV |

### Applications of Consensus

- **Distributed Lock Services**: etcd, ZooKeeper, Chubby—ensure only one process holds a lock at a time
- **Service Discovery**: Consul, etcd—maintain consistent registry of available services
- **Configuration Management**: Distribute configuration changes atomically across a cluster
- **Leader Election**: Elect a single coordinator for tasks like cron job scheduling or database write coordination
- **Replicated State Machines**: The foundation of strongly consistent distributed databases (CockroachDB, TiDB, Spanner)

## Layman's Explanation

### The Raft Captain Analogy
Imagine a ship crew that must operate even if the captain falls overboard. The crew needs a clear protocol (Raft):

**Leader Election**: If no captain gives orders for 5 seconds (timeout), any crew member can shout "I volunteer as captain!" (become a candidate). If more than half the crew agrees, that person becomes the new captain (leader elected by majority).

**Log Replication**: The captain writes orders in a numbered logbook. Before an order is official, the captain reads it to the crew, and at least half must write it in their own logbooks (committed after majority acknowledgment). Once committed, the order will be carried out even if the captain falls overboard.

**Safety**: The protocol prevents two captains simultaneously because a captain needs majority support. It prevents lost orders because new captains must have the most complete logbook (election restriction).

### The Two-Generals Problem
Two generals must coordinate an attack via messengers that might be captured (unreliable network). How do they agree on a time? If General A sends "attack at dawn" and waits for confirmation, General B might not confirm because the confirmation messenger was captured. It's unsolvable in pure form—hence why consensus algorithms require a **majority** (not unanimity) quorum to make progress despite failures.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Database Consistency Model**: Understanding Raft/Paxos explains why strongly consistent databases (CockroachDB, Spanner) can guarantee correctness but have higher write latency than eventually consistent databases (Cassandra, DynamoDB as default)
- **Kubernetes Operations**: etcd uses Raft. Understanding Raft leader election and log replication is essential for diagnosing etcd performance issues and cluster instability in Kubernetes
- **Service Discovery Architecture**: Consul uses Raft. Incorrect deployment (e.g., too many nodes across high-latency WAN links) causes constant leader elections and service discovery instability
- **Distributed Locking**: Understanding why consensus is required for correct distributed locking prevents subtle race conditions in microservice orchestration

### Business Impact
- **Data Correctness**: Consensus-based databases prevent split-brain scenarios that cause data corruption. Financial systems, inventory management, and any system where correctness matters depends on this
- **System Reliability**: Kubernetes itself depends on etcd's Raft consensus. Understanding quorum requirements helps architects correctly size control planes (odd number of nodes, 3 or 5, spread across failure domains)
- **Cost Optimization**: Consensus requires majority quorum for writes, which means 3 nodes tolerate 1 failure, 5 nodes tolerate 2. Over-provisioning consensus nodes wastes resources; under-provisioning causes outages

### When NOT to Use Consensus
- High-write-throughput systems: Consensus adds latency (majority acknowledgment). For throughput-bound systems, accept eventual consistency.
- Geographically distributed with high latency: Consensus requires frequent heartbeats. If latency between nodes exceeds 50-100ms, use asynchronous replication instead.
- Simple caching layers: A Redis cache doesn't need consensus—losing cached data is acceptable.

## On-Premises Examples

### etcd (Raft-based)
Kubernetes' backing store:

```bash
# 3-node etcd cluster
etcd --name etcd-1 \
  --initial-cluster etcd-1=http://10.0.0.1:2380,etcd-2=http://10.0.0.2:2380,etcd-3=http://10.0.0.3:2380 \
  --initial-cluster-state new \
  --listen-client-urls http://0.0.0.0:2379 \
  --listen-peer-urls http://0.0.0.0:2380

# Write (leader handles it, replicates to majority)
etcdctl put /config/db/host "db-internal.example.com"

# Watch for changes (linearizable read with quorum)
etcdctl watch /config/ --prefix
```

### Apache ZooKeeper (ZAB - ZooKeeper Atomic Broadcast)
ZooKeeper uses a custom consensus protocol (ZAB) inspired by Paxos:

```bash
# zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper

server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

```java
// Leader election, distributed locks, configuration management
CuratorFramework client = CuratorFrameworkFactory.newClient("zk1:2181,zk2:2181,zk3:2181", ...);
LeaderLatch latch = new LeaderLatch(client, "/service-leader");
latch.start();
latch.await(); // blocks until this instance is the leader
```

### HashiCorp Consul (Raft-based)
```bash
consul agent -server -bootstrap-expect=3 \
  -node=consul-1 -data-dir=/var/consul \
  -retry-join=consul-2 -retry-join=consul-3

# Key-value with consensus
consul kv put service/config '{"port": 8080}'

# Health checking with consensus-driven service catalog
consul services register -name=web -port=8080
```

## AWS Examples

### Amazon Aurora Storage Layer (Quorum-based Consensus)
Aurora uses a quorum model across 6 storage nodes in 3 AZs:

- **Write quorum**: 4 out of 6 nodes must acknowledge (tolerates 1 AZ + 1 node failure)
- **Read quorum**: 3 out of 6 nodes must respond (ensures read gets latest version)

This is essentially a consensus protocol at the storage layer, enabling Aurora's strong consistency without the overhead of full Raft/Paxos at the database layer.

### AWS Managed etcd (Amazon EKS)
```bash
# EKS manages etcd for the control plane
# For self-managed etcd on EC2:

aws ec2 run-instances --count 3 --instance-type t3.medium \
  --subnet-id subnet-abc --security-group-ids sg-xyz \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=etcd-node}]'
```

```hcl
# Place etcd nodes in different AZs for fault tolerance
resource "aws_instance" "etcd" {
  count              = 3
  ami                = data.aws_ami.ubuntu.id
  instance_type      = "t3.medium"
  subnet_id          = element(aws_subnet.private[*].id, count.index)
  availability_zone  = element(data.aws_availability_zones.available.names, count.index)
}
```

### Amazon ECS Service Discovery (Powered by AWS Cloud Map with consensus-like guarantees)

## GCP Examples

### Cloud Spanner (TrueTime-based Consensus)
Spanner achieves external consistency using TrueTime (GPS + atomic clocks) instead of traditional consensus for ordering:

- TrueTime provides a globally synchronized clock with bounded uncertainty (typically < 7ms)
- Transactions are assigned timestamps within the uncertainty window
- Spanner waits out the uncertainty before committing to guarantee ordering

This is consensus without explicit voting rounds, trading clock dependency for transaction performance.

```sql
-- Spanner automatically handles consensus for every write
INSERT INTO Orders (OrderID, CustomerID, Total)
VALUES ('ORD-001', 'CUST-789', 99.99);
-- This write is replicated across zones with synchronous consensus before acknowledgment
```

### GKE (Kubernetes) etcd
GKE manages the control plane etcd cluster. For regional clusters, etcd is replicated across zones within the region using Raft.

### Memorystore Cluster Management
Redis Cluster uses gossip protocol for membership and failure detection rather than strong consensus—a conscious tradeoff for lower latency at the cost of potential split-brain in certain failure modes.

## Azure Examples

### Azure Cosmos DB (Quorum-based Replication)
Cosmos DB uses a custom consensus protocol across its replica sets:

```
Consistency Level    | Reads from  | Writes to
Strong               | Majority     | Majority
Bounded Staleness    | 2 replicas   | Majority
Session              | 2 replicas   | Majority
Consistent Prefix    | 2 replicas   | Majority
Eventual             | Any          | Any
```

### Azure Kubernetes Service (AKS)
AKS manages etcd for the control plane:

```bash
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --enable-cluster-autoscaler \
  --min-count 1 --max-count 10
# AKS control plane runs etcd in a highly available configuration
```

### Azure Service Fabric
Uses a custom replicated state machine with consensus-based leader election for reliable services:

```csharp
// Reliable service with state managed via consensus-backed replication
protected override async Task RunAsync(CancellationToken cancellationToken)
{
    var myDictionary = await this.StateManager
        .GetOrAddAsync<IReliableDictionary<string, long>>("myDictionary");

    using (var tx = this.StateManager.CreateTransaction())
    {
        await myDictionary.AddOrUpdateAsync(tx, "counter", 1, (key, value) => ++value);
        await tx.CommitAsync();
    }
}
```

## Summary

| Consensus System | Algorithm | Primary Use | Kubernetes Integration |
|-----------------|-----------|-------------|----------------------|
| etcd | Raft | Config, service discovery | Native (control plane) |
| ZooKeeper | ZAB (Paxos-variant) | Coordination, locking | Optional (older clusters) |
| Consul | Raft | Service mesh, KV store | Optional (service mesh) |
| Spanner | TrueTime + Paxos | Globally distributed DB | N/A |
| CockroachDB | Raft | Distributed SQL DB | Can run on K8s |

Consensus algorithms are the invisible backbone of distributed system reliability. They solve the fundamental coordination problems that arise when you move from a single machine to a cluster. While you may never implement Raft yourself, understanding its guarantees, quorum requirements, and failure modes is essential for designing systems on top of etcd, ZooKeeper, and consensus-based databases.
