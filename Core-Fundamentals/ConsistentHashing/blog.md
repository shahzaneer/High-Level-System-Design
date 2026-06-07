# Consistent Hashing

## Introduction
Consistent Hashing is one of the most elegant algorithms in distributed systems design. It was introduced by David Karger et al. at MIT in 1997 to solve a fundamental problem: how to distribute data across a dynamic set of servers without reshuffling everything when servers are added or removed. Before consistent hashing, systems used simple modulo-based hashing (`hash(key) % N`), which required remapping nearly every key whenever the number of servers N changed. This made horizontal scaling operationally painful and expensive.

Today, consistent hashing powers the core of distributed databases like Amazon DynamoDB, Apache Cassandra, distributed caches like Memcached and Redis Cluster, CDNs, and load balancers. It is the invisible backbone that enables elastic scaling—adding and removing nodes seamlessly without massive data migration.

## Definition
**Consistent Hashing** is a distributed hashing scheme that operates independently of the number of servers or objects in a distributed hash table. It maps both keys and servers (nodes) onto a circular hash space (the "hash ring"). A key is assigned to the nearest server clockwise on the ring. When a server is added or removed, only `K/N` keys (on average, where K = total keys, N = number of servers) need to be remapped, compared to traditional modulo hashing where nearly all keys get remapped.

## Concept Explanation

### The Hash Ring
The core idea is a virtual ring where hash values wrap around from 0 to 2^m - 1 (typically m = 32 or 64 bits).

1. **Map Servers to the Ring**: Hash each server's identifier (e.g., IP address, instance ID) to a position on the ring. For example, `hash(server_A) → 25`, `hash(server_B) → 170`, `hash(server_C) → 300`.

2. **Map Keys to the Ring**: Hash each data key to a position on the same ring. For example, `hash(key_foo) → 90`, `hash(key_bar) → 280`.

3. **Assign Keys**: Each key is stored on the first server found by moving clockwise around the ring from the key's position. So `key_foo (90)` goes to `server_B (170)`, and `key_bar (280)` goes to `server_C (300)`.

### Why It's "Consistent"
When you add or remove a server:

**Adding Server D** with hash position 120:
- Only keys in the range (90, 120] move from server_B to server_D
- `key_foo (90)` would now map to `server_D (120)` instead of `server_B (170)`
- All other keys remain unaffected

**Removing Server B**:
- Only keys previously mapped to server_B move to server_C
- Everything else stays put

This minimizes data redistribution during scaling operations.

### Virtual Nodes (VNodes)
The basic approach has two problems: non-uniform distribution (some servers may get disproportionately more keys) and inability to handle heterogeneous server capacities.

**Solution**: Each physical server is represented by multiple virtual nodes on the ring:
- Server A gets 100 virtual nodes (e.g., `hash(server_A_v1)`, `hash(server_A_v2)`, ...)
- Server B with double capacity gets 200 virtual nodes

Virtual nodes provide:
- **Better load distribution**: More points on the ring create a more uniform distribution
- **Weighted distribution**: Servers with more capacity get more virtual nodes
- **Graceful degradation**: When a server fails, its load spreads evenly across remaining servers

### Data Replication in Consistent Hashing
For fault tolerance, data is replicated to the next R servers clockwise on the ring (R = replication factor). Amazon DynamoDB uses R=3 by default, writing to the next 3 nodes on the ring.

## Python Implementation Example

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, nodes=None, virtual_nodes=150):
        self.ring = {}
        self.sorted_keys = []
        self.virtual_nodes = virtual_nodes
        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key):
        return int(hashlib.md5(str(key).encode()).hexdigest(), 16)

    def add_node(self, node):
        for i in range(self.virtual_nodes):
            vnode_key = f"{node}:vnode:{i}"
            hash_val = self._hash(vnode_key)
            self.ring[hash_val] = node
            bisect.insort(self.sorted_keys, hash_val)

    def remove_node(self, node):
        for i in range(self.virtual_nodes):
            vnode_key = f"{node}:vnode:{i}"
            hash_val = self._hash(vnode_key)
            if hash_val in self.ring:
                del self.ring[hash_val]
                self.sorted_keys.remove(hash_val)

    def get_node(self, key):
        if not self.ring:
            return None
        hash_val = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, hash_val)
        if idx == len(self.sorted_keys):
            idx = 0  # wrap around
        return self.ring[self.sorted_keys[idx]]

# Usage
ring = ConsistentHashRing(nodes=["server-A", "server-B", "server-C"])
print(ring.get_node("user_session_12345"))  # deterministic assignment
ring.add_node("server-D")
print(ring.get_node("user_session_12345"))  # may or may not change
```

## Layman's Explanation

Imagine a giant clock (the ring) with 360 degree markings. Instead of hour numbers, you place your storage boxes (servers) at various positions around the clock.

When you have a document to store (a key), you look at its "time" (hash value), find where that time falls on the clock, and then move clockwise until you hit the nearest storage box. That's where the document goes.

**Without consistent hashing**: If you add a new storage box, you'd need to check every single document and potentially move it to a new box—like reorganizing an entire library because you added one new shelf.

**With consistent hashing**: Adding a new box only affects documents that naturally fall between that new box and the previous one moving clockwise. Most documents stay exactly where they were—like adding a new shelf to your library at one specific spot; only the books near that spot get shifted, while the rest of the library remains untouched.

**Virtual nodes**: Instead of placing one big box at the 3 o'clock position, you place 100 tiny boxes spread around the clock. This way, every part of the clock has roughly even coverage, and no single box gets overwhelmed.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Elastic Scaling Strategy**: Consistent hashing enables adding/removing database and cache nodes with minimal data migration, critical for cloud-native, auto-scaling architectures
- **Distributed Cache Architecture**: Memcached and Redis clusters use consistent hashing for sharding. Without it, cache node changes cause massive cache invalidation storms (the "thundering herd" problem)
- **Database Partitioning**: NoSQL databases like Cassandra and DynamoDB use consistent hashing under the hood. Understanding it explains their performance characteristics and failure modes
- **CDN Routing**: CDNs use consistent hashing to map content URLs to edge servers, ensuring requests for the same content consistently hit caches that already have it
- **Load Balancing**: Layer 7 load balancers can use consistent hashing (source IP hash) for session affinity without central state storage

### Business Impact
- **Cost Savings**: Minimizes data transfer costs during scaling operations in cloud environments (inter-AZ and cross-region data transfer is expensive)
- **Performance**: Prevents cache invalidation cascades that can crash databases during rolling restarts or auto-scaling events
- **Availability**: Virtual nodes enable graceful degradation—when a node fails, its load distributes evenly instead of overwhelming a single neighbor
- **Operational Simplicity**: Reduces the complexity and risk of scaling operations, enabling more frequent, safer infrastructure changes

## On-Premises Examples

### Redis Cluster with Consistent Hashing
Redis Cluster uses a form of consistent hashing with hash slots (16384 slots distributed across master nodes). On-premises deployment:

```bash
# Create 6 Redis instances (3 masters, 3 replicas)
redis-server --port 7000 --cluster-enabled yes --cluster-config-file nodes-7000.conf
redis-server --port 7001 --cluster-enabled yes --cluster-config-file nodes-7001.conf
# ... up to 7005

# Form the cluster
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1
```

### Apache Cassandra On-Premises
Cassandra uses a ring-based consistent hashing with virtual nodes (vnodes) enabled by default:

```yaml
# cassandra.yaml
num_tokens: 256           # number of virtual nodes per physical node
partitioner: org.apache.cassandra.dht.Murmur3Partitioner
endpoint_snitch: GossipingPropertyFileSnitch
```

Adding a node: Cassandra automatically redistributes token ranges with `nodetool decommission` for removals and `nodetool bootstrap` for additions.

### HAProxy with Consistent Hashing
```haproxy
backend web_servers
    balance hash-type consistent
    hash-type consistent
    server web1 192.168.1.10:80 weight 100 check
    server web2 192.168.1.11:80 weight 100 check
    server web3 192.168.1.12:80 weight 200 check
```

## AWS Examples

### Amazon DynamoDB
DynamoDB partitions data using a consistent hashing ring internally. Each partition holds ~10GB of data and is distributed across multiple AZs:

- **Hash Key**: The partition key is hashed onto the ring
- **Auto-Scaling**: When partitions split due to size or throughput, data redistribution follows consistent hashing semantics
- **Global Tables**: Multi-region replication using vector clocks built on the hash ring foundation

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# DynamoDB automatically hashes the partition key and route to the correct shard
response = table.put_item(
    Item={
        'OrderID': 'ORD-2024-001',  # this key is hashed onto the ring
        'CustomerID': 'CUST-789',
        'Total': 150.00
    }
)
```

### Amazon ElastiCache (Redis)
ElastiCache Redis clusters use consistent hashing with 16384 hash slots:

- **Cluster Mode Disabled**: Single shard, no consistent hashing needed
- **Cluster Mode Enabled**: Up to 500 shards, each responsible for a range of hash slots
- **Online Resharding**: ElastiCache can add/remove shards and redistribute hash slots with minimal downtime, leveraging consistent hashing principles

### AWS Application Load Balancer
ALB uses consistent hashing for sticky sessions:

```hcl
resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = true
  }
}
```

## GCP Examples

### Cloud Memorystore (Redis Cluster)
Memorystore Redis Cluster uses consistent hashing with hash slots:

```bash
gcloud redis clusters create my-redis-cluster \
  --region=us-central1 \
  --replica-count=2 \
  --shard-count=3 \
  --node-type=REDIS_HIGHMEM_MEDIUM
```

When you scale shards, consistent hashing ensures only keys in affected hash slots migrate.

### Cloud CDN
Google Cloud CDN uses consistent hashing to distribute content requests across edge caches:

```bash
gcloud compute backend-services create my-backend \
  --load-balancing-scheme=EXTERNAL \
  --protocol=HTTP \
  --enable-cdn

gcloud compute backend-services update my-backend \
  --consistent-hash-http-header-name="X-Session-ID"
```

### Cloud Spanner
While Spanner uses a different architecture (TrueTime-based), its split management for data distribution follows consistent hashing-like principles for load balancing splits across nodes automatically.

## Azure Examples

### Azure Cosmos DB
Cosmos DB partitions data using a combination of partition keys and logical partitions that are mapped to physical partitions:

```json
{
  "id": "doc-001",
  "userId": "user-123",
  "partitionKey": "user-123"
}
```

The partition key is hashed to determine placement. When storage or throughput exceeds 10GB or 10,000 RU/s, the logical partition splits—following consistent hashing redistribution principles.

### Azure Redis Cache (Enterprise Tier)
```
az redisenterprise create \
  --cluster-name "myRedisCluster" \
  --resource-group "myResourceGroup" \
  --location "East US" \
  --sku "Enterprise_E10" \
  --capacity 2
```

Enterprise tier supports clustering with hash slot-based consistent hashing for sharded data distribution.

### Azure Application Gateway
Uses consistent hashing for cookie-based session affinity:

```json
{
  "backendAddressPools": [...],
  "backendHttpSettingsCollection": [
    {
      "cookieBasedAffinity": "Enabled",
      "affinityCookieName": "ApplicationGatewayAffinity",
      "requestTimeout": 30
    }
  ]
}
```

## Summary

| Aspect | Modulo Hashing | Consistent Hashing |
|--------|---------------|-------------------|
| Node addition impact | Remaps ~100% of keys | Remaps ~K/N keys |
| Load distribution | Even (with good hash) | Can be uneven without vnodes |
| Virtual nodes | N/A | Yes, for uniform distribution |
| Implementation complexity | Trivial | Moderate |
| Use cases | Fixed-size pools | Dynamic, elastic clusters |

Consistent hashing is not just an algorithm—it's a design philosophy that enables horizontally scalable, elastic distributed systems. Every solution architect should understand both the ring concept and virtual nodes, as they form the foundation of modern distributed databases, caches, and CDNs.
