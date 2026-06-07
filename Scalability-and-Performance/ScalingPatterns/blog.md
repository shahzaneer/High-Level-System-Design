# Scaling Patterns

## Introduction
Scalability is the ability of a system to handle growing amounts of work by adding resources. It is one of the most fundamental non-functional requirements in system design—a system that cannot scale becomes a victim of its own success. Twitter's "Fail Whale" era (2008-2012) was a scaling failure: the system could not handle its explosive user growth, resulting in frequent outages that became a cultural meme.

Scaling is not monolithic. Different components of a system scale differently: stateless web servers scale horizontally almost infinitely; relational databases scale vertically with limits; distributed databases scale horizontally but add complexity. The architect's job is to identify bottlenecks before they become outages and apply the right scaling pattern to each component.

## Definition

**Scalability** is the property of a system to handle a growing amount of work by adding resources. Two fundamental approaches:

**Vertical Scaling (Scale Up)**: Add more resources to an existing node—more CPU, RAM, disk, network. Like upgrading from a sedan to a truck.

**Horizontal Scaling (Scale Out)**: Add more nodes to the system—more servers, more database replicas. Like adding more trucks to a fleet instead of getting a bigger truck.

## Concept Explanation

### Vertical vs Horizontal: The Trade-Offs

```
VERTICAL SCALING:
  ✓ Simple: no code changes, no distributed systems complexity
  ✓ Consistent: single-node databases provide strong consistency
  ✗ Hard limit: eventually the biggest machine isn't big enough
  ✗ Single point of failure (unless redundant with failover)
  ✗ Expensive at high end: 256-core servers cost exponentially more
  ✗ Downtime to scale: must shutdown, upgrade hardware, restart

HORIZONTAL SCALING:
  ✓ Theoretically infinite: add nodes indefinitely
  ✓ Elastic: scale up AND down based on demand (cloud-native)
  ✓ Commodity hardware: use cheap machines instead of expensive big ones
  ✓ Resilience: failure of one node doesn't take down the system
  ✗ Distributed complexity: consistency, partitioning, coordination
  ✗ Network overhead: inter-node communication adds latency
  ✗ Application must be designed for it: stateless, partitionable
```

### Auto-Scaling Patterns

#### Reactive Auto-Scaling (Metric-Driven)
```python
class ReactiveAutoScaler:
    """
    Scale based on observed metrics. Reacts AFTER load changes.
    """
    
    def __init__(self, target_cpu=70, scale_up_threshold=80, 
                 scale_down_threshold=30, cooldown_seconds=300):
        self.target_cpu = target_cpu
        self.scale_up_threshold = scale_up_threshold
        self.scale_down_threshold = scale_down_threshold
        self.cooldown = cooldown_seconds
        self.last_scale_time = datetime.min
    
    def evaluate(self, current_cpu, current_replicas):
        if (datetime.now() - self.last_scale_time).seconds < self.cooldown:
            return None  # Still in cooldown
        
        desired_replicas = current_replicas * (current_cpu / self.target_cpu)
        
        if current_cpu > self.scale_up_threshold:
            self.last_scale_time = datetime.now()
            return min(ceil(desired_replicas), MAX_REPLICAS)
        
        elif current_cpu < self.scale_down_threshold:
            self.last_scale_time = datetime.now()
            return max(floor(desired_replicas), MIN_REPLICAS)
        
        return None  # No change needed
```

#### Predictive Auto-Scaling (Schedule + ML-Based)
```python
class PredictiveAutoScaler:
    """
    Scale BEFORE load arrives based on historical patterns.
    Critical for applications with predictable daily/weekly patterns.
    """
    
    def __init__(self):
        self.schedule = {
            'weekday_morning': {'time': '08:00', 'replicas': 10},
            'weekday_peak':    {'time': '12:00', 'replicas': 50},
            'weekday_evening': {'time': '18:00', 'replicas': 20},
            'weekend':         {'time': None,    'replicas': 5},
            'black_friday':    {'date': '2024-11-29', 'replicas': 200}
        }
    
    def predict(self, now, current_load):
        # Scheduled scaling
        scheduled = self._get_scheduled_replicas(now)
        
        # ML prediction based on recent trends
        trend = self._predict_from_trends(now)
        
        # Combine: max of schedule and prediction
        return max(scheduled, trend)
```

### The 12-Factor App Stateless Principle

```
Scalable applications must be STATELESS:
  - Any instance can handle any request
  - No session state stored locally (no sticky sessions needed)
  - Shared nothing architecture
  
State must live in backing services:
  - Database (persistent data)
  - Cache (Redis/Memcached for sessions)
  - Message Queue (async work items)
  - Object Storage (files/assets)

Stateless = Horizontally Scalable
```

```python
# BAD: Stateful (not horizontally scalable)
sessions = {}  # In-memory dict lost when pod restarts

@app.route('/cart/add')
def add_to_cart():
    session_id = request.cookies.get('session')
    if session_id not in sessions:
        sessions[session_id] = []
    sessions[session_id].append(request.json['item'])
    return "Added"

# GOOD: Stateless (horizontally scalable)
@app.route('/cart/add')
def add_to_cart():
    session_id = request.cookies.get('session')
    cart = redis.get(f"cart:{session_id}") or []
    cart.append(request.json['item'])
    redis.setex(f"cart:{session_id}", 3600, json.dumps(cart))
    return "Added"
```

### Database Scaling Patterns

```
READ SCALING (Read Replicas):
  [Writer] ──replication──→ [Reader 1]
                         └─→ [Reader 2]
                         └─→ [Reader 3]
  Writes → Writer only
  Reads → Any reader (eventually consistent)
  Good for: read-heavy workloads (90%+ reads)

WRITE SCALING (Sharding):
  [Shard 0: users A-G]
  [Shard 1: users H-N]
  [Shard 2: users O-U]
  [Shard 3: users V-Z]
  Each shard handles its own reads AND writes
  Good for: write-heavy workloads, massive datasets

CACHING:
  [App] → [Redis] → [Database]
  Cache 95%+ of reads; database only handles 5% + all writes
  Most cost-effective scaling technique
```

### Queue-Based Load Leveling

```
Without Queue:
  [Traffic Spike] ──→ [App Server] ──→ Overwhelmed, 503 errors

With Queue:
  [Traffic Spike] ──→ [SQS/Kafka] ──→ [Worker Pool] ──→ Steady processing
  Queue absorbs the spike; workers process at their own pace
  Smooths traffic peaks into steady, manageable throughput
```

## Layman's Explanation

### The Restaurant Kitchen
**Vertical Scaling (Bigger Kitchen)**: The restaurant is getting busier. You buy a bigger stove, a bigger fridge, hire a faster chef. This works until you run out of physical space—you can't fit a 20-burner stove in a 200 sq ft kitchen.

**Horizontal Scaling (More Kitchens)**: Instead of one giant kitchen, you open three kitchens. The host (load balancer) sends customers to whichever kitchen is least busy. If one kitchen catches fire (server failure), the other two keep serving. Need more capacity? Build a fourth kitchen.

**Which is right?**:
- A small family restaurant (low-traffic blog) → vertical scaling is fine
- A 500-seat banquet hall (e-commerce platform) → horizontal scaling is mandatory

### The Highway Analogy
**Vertical**: Adding more lanes to a single highway. Eventually you hit the limit (can't add a 50th lane). Construction (upgrade) causes traffic jams.

**Horizontal**: Building parallel highways. Traffic is distributed. One highway can close for maintenance without stopping all traffic.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Identify the Bottleneck First**: Scaling the web tier when the database is the bottleneck wastes money and doesn't improve performance. Use load testing and metrics to identify the actual bottleneck before scaling.
- **Stateless Design**: The single most impactful architectural decision for scalability. If every instance can handle any request, scaling is trivial (just add instances). If sessions are pinned to specific instances, scaling requires sticky sessions and careful state management.
- **Database Scaling is the Hardest Part**: Application servers scale horizontally almost trivially. Databases are the bottleneck in 90% of systems. The database scaling strategy—read replicas, sharding, caching—is the most consequential architectural decision.
- **Cost of Over-Provisioning vs Under-Provisioning**: Over-provisioning wastes money (idle resources). Under-provisioning causes outages during traffic spikes. Auto-scaling finds the balance—but auto-scaling cold starts add latency, so a baseline of over-provisioning (20-30% headroom) is standard practice.

### Business Impact
- **Black Friday Preparedness**: E-commerce platforms must handle 10x normal traffic on Black Friday. Scaling architecture that handled normal traffic beautifully but crumbles at 10x destroys revenue during the most critical sales period of the year.
- **Cost Efficiency**: Auto-scaling down during off-hours (nights, weekends) can reduce cloud costs by 40-60% compared to static provisioning at peak capacity.

## On-Prem + Cloud Examples

```yaml
# Kubernetes HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 5 min before scaling down
```

```hcl
# AWS Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  min_size         = 3
  max_size         = 20
  desired_capacity = 3

  target_group_arns = [aws_lb_target_group.app.arn]

  tag {
    key                 = "Name"
    value               = "app-server"
    propagate_at_launch = true
  }
}
```

## Summary

| Component | Vertical Scale | Horizontal Scale | Best Practice |
|-----------|---------------|-----------------|---------------|
| Web/App Server | Rarely (16+ cores) | Always | Stateless + Auto-scale groups |
| Cache (Redis) | Cluster mode | Native sharding | Redis Cluster |
| Database | Up to ~64 cores | Read replicas + sharding | Cache heavily before sharding |
| Message Queue | N/A | Partition-based | Kafka/SQS partitions |
| CDN | N/A | Edge locations | Always use CDN for static assets |

Scaling is about removing bottlenecks. The first step is always measurement—identify which component is limiting throughput. The second step is applying the right scaling pattern to that component. The third step is measuring again to find the next bottleneck. Scaling is never "done"—it's a continuous process of monitoring, identifying constraints, and removing them before users notice.
