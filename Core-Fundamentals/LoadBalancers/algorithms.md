# Load Balancing Algorithms

---

## 1. Round Robin

### How It Works

The simplest algorithm. Maintains a circular list of servers and assigns each new request to the next server in sequence, wrapping back to the first server after the last.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (wraps around)
Request 5 → Server B
```

### When to Use

- All backend servers have identical specs (CPU, memory, capacity)
- Requests have similar processing requirements
- Servers are in same location or network
- Simple stateless applications

### Advantages

- Simple to implement and understand
- Evenly distributes requests numerically
- Low overhead, very fast decision making
- Predictable distribution pattern

### Disadvantages

- Doesn't consider server capacity or current load
- Inefficient if servers have different capabilities
- Can overload slow servers while fast ones are underutilized
- No awareness of request complexity

### Real World Example

A content delivery serving static images. All servers identical, all requests similar (serve an image). Round robin perfectly distributes the load.

**Layer Support:** L4 and L7

---

## 2. Weighted Round Robin

### How It Works

Like round robin, but assigns weights to servers based on capacity. Higher weight means more requests.

```
Server A weight = 3
Server B weight = 2
Server C weight = 1

Request 1 → Server A (count 1)
Request 2 → Server A (count 2)
Request 3 → Server A (count 3)
Request 4 → Server B (count 1)
Request 5 → Server B (count 2)
Request 6 → Server C (count 1)
Request 7 → Server A (cycle repeats)
```

Over 6 requests:

- Server A: 50%
- Server B: 33%
- Server C: 17%

### When to Use

- Servers have different capacities
- Gradual traffic shifting (canary deployments)
- Cost optimization
- Phased rollouts of new versions

### Advantages

- Matches traffic to server capacity
- Simple yet flexible
- Useful for blue green and canary deployments
- Can gradually shift traffic

### Disadvantages

- Doesn't consider real time load
- Manual weight configuration required
- Requires tuning

### Real World Example

Mixed instances:
3 x m5.2xlarge (weight 4)
2 x m5.xlarge (weight 2)

Canary deployment:
Old version weight 95
New version weight 5

**Layer Support:** L4 and L7

---

## 3. Least Connections

### How It Works

Routes new requests to the server with the fewest active connections.

```
Server A: 45 connections
Server B: 32 connections
Server C: 38 connections

New request → Server B
```

### When to Use

- Highly variable request durations
- Long lived connections
- Similar server capacity
- Interactive applications

### Advantages

- Dynamically adapts to load
- Prevents server overload
- Better than round robin for uneven workloads

### Disadvantages

- More complex than round robin
- Connection count may not reflect actual load
- Tracking overhead

### Real World Example

WebSocket based chat applications or database connection pooling.

**Layer Support:** Mostly L4

---

## 4. Weighted Least Connections

### How It Works

Uses the ratio of active connections divided by server weight.

```
Server A: 60 / weight 3 = 20
Server B: 30 / weight 2 = 15
Server C: 20 / weight 1 = 20

New request → Server B
```

### When to Use

- Different server capacities
- Persistent connections
- Cost and performance optimization

### Advantages

- Adapts to capacity and load
- Best for heterogeneous servers

### Disadvantages

- Complex to implement
- Requires accurate weights
- Higher overhead

### Real World Example

Video streaming platforms with mixed server specs.

**Layer Support:** L4 and L7

---

## 5. Least Response Time (Least Latency)

### How It Works

Routes traffic to the server with the fastest average response time.

```
Server A: 120ms
Server B: 85ms
Server C: 150ms

New request → Server B
```

### When to Use

- Multi region deployments
- Latency sensitive applications
- Hybrid or multi cloud setups

### Advantages

- Self optimizing
- Accounts for network and processing latency
- Best user experience

### Disadvantages

- Complex monitoring
- Can cause feedback loops
- Sensitive to transient issues

### Real World Example

Global CDN or geo distributed databases.

**Layer Support:** L7

---

## 6. IP Hash (Source IP Hash)

### How It Works

Uses client IP to deterministically select a server.

```
Hash(203.0.113.45) % 3 = Server 1
Hash(198.51.100.23) % 3 = Server 2
```

### When to Use

- Session affinity without cookies
- Caching scenarios
- Non HTTP protocols

### Advantages

- Stateless
- Deterministic
- Works at L4
- Good cache locality

### Disadvantages

- Poor distribution behind NAT
- Adding servers reshuffles mappings
- No load awareness

### Real World Example

Gaming servers and CDN origin shielding.

**Layer Support:** L4 and L7

---

## 7. URL Hash (Path Based Routing)

### How It Works

Hashes URL path to route similar requests to the same backend.

```
/api/users → Server 1
/api/products → Server 2
/images/logo.png → Server 3
```

### When to Use

- Heavy server side caching
- Microservices
- Content delivery

### Advantages

- Excellent cache hit rate
- Predictable routing

### Disadvantages

- Uneven load possible
- L7 only
- Mapping disruption on scale

### Real World Example

CDN origin routing and API gateways.

**Layer Support:** L7 only

---

## 8. Random

### How It Works

Selects a backend server randomly.

### When to Use

- Homogeneous servers
- Stateless apps
- Testing environments

### Advantages

- Zero overhead
- Simple

### Disadvantages

- Unpredictable short term behavior
- No load awareness

### Real World Example

Mostly dev or fallback scenarios.

**Layer Support:** L4 and L7

---

## 9. Random with Two Choices (Power of Two Choices)

### How It Works

Randomly pick two servers and choose the less loaded one.

```
Server B: 45 connections
Server D: 32 connections

Route → Server D
```

### When to Use

- Distributed systems
- Service mesh
- High throughput environments

### Advantages

- Near optimal load balancing
- Minimal overhead
- No global state

### Disadvantages

- Not fully optimal
- Less predictable

### Real World Example

Istio or Linkerd service mesh.

**Layer Support:** L4 and L7

---

## 10. Least Bandwidth

### How It Works

Routes to server with lowest current bandwidth usage.

```
Server A: 150 Mbps
Server B: 95 Mbps
Server C: 180 Mbps

New request → Server B
```

### When to Use

- File transfers
- Video streaming
- Large payload workloads

### Advantages

- Prevents bandwidth saturation
- Optimizes throughput

### Disadvantages

- Hard to measure accurately
- Ignores CPU and memory

### Real World Example

File hosting platforms.

**Layer Support:** L7

---

## 11. Consistent Hashing

### How It Works

Maps servers and requests on a hash ring to minimize remapping.

### When to Use

- Distributed caches
- Sharded databases
- Dynamic scaling environments

### Advantages

- Minimal disruption
- Excellent cache locality
- Scales well

### Disadvantages

- Complex implementation
- Requires good hashing

### Real World Example

Memcached and DynamoDB.

**Layer Support:** L4 and L7

---

## 12. Geographic / Geolocation Based Routing

### How It Works

Routes based on client geographic location.

### When to Use

- Global applications
- CDN
- Compliance requirements

### Advantages

- Lowest latency
- Data residency compliance

### Disadvantages

- Geo IP inaccuracies
- Complex failover
- Uneven regional load

### Real World Example

Netflix, global ecommerce, online gaming.

**Layer Support:** L7

---

## 13. Priority Based / Custom Policy Routing

### How It Works

Routes traffic based on business rules and request attributes.

### When to Use

- Multi tenant SaaS
- A B testing
- Compliance driven routing

### Advantages

- Extremely flexible
- Business aligned routing

### Disadvantages

- Complex configuration
- Easy to misconfigure
- Debugging is hard

### Real World Example

Tier based SaaS, API gateways, media platforms.

**Layer Support:** L7 only
