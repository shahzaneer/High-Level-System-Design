# Rate Limiting

## Introduction
Rate limiting is a fundamental traffic control mechanism that protects systems from being overwhelmed by excessive requests. While the concept has existed since the early days of networking, it gained critical importance with the rise of public APIs (Twitter's infamous 2007 rate limit implementation), the proliferation of DDoS attacks, and the shift to microservices where cascading failures can bring down entire systems.

Today, rate limiting is deployed at every layer of the stack: API gateways enforce per-user quotas, load balancers throttle per-IP connection rates, and application-level rate limiters protect individual services from abusive or buggy clients. It's not just about security—it's about ensuring fair resource allocation, predictable system behavior, and cost control in cloud environments where you pay per request.

## Definition
**Rate Limiting** is the practice of controlling the rate at which requests are processed by a system. It imposes a constraint on how many operations a client can perform within a specified time window. When the limit is exceeded, the system rejects or queues the excess requests, typically returning HTTP 429 (Too Many Requests) to the client.

Key aspects:
- **Limit**: Maximum number of requests allowed in a time window (e.g., 100 requests per minute)
- **Window**: The time period over which the limit applies (fixed window, sliding window, or token bucket)
- **Identifier**: What identifies a client (IP address, API key, user ID, session token)
- **Action**: What happens when the limit is exceeded (rejection with 429, queuing, or throttling)

## Concept Explanation

### Rate Limiting Algorithms

#### Token Bucket
The most widely used algorithm due to its ability to handle bursts gracefully.

```
[Bucket capacity: 100 tokens]
[Refill rate: 10 tokens/second]

Request arrives → Need 1 token
    If bucket has token → Remove token, process request
    If bucket empty → Reject (429)
```

Tokens are added at a steady rate. The bucket can accumulate unused tokens up to a maximum capacity, allowing bursts up to the bucket size. This is ideal for APIs that experience traffic spikes.

```python
import time

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity        # max tokens
        self.refill_rate = refill_rate  # tokens per second
        self.tokens = capacity
        self.last_refill = time.time()
    
    def consume(self, tokens=1):
        self._refill()
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False
    
    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now

# Usage: 100 requests per minute with burst capacity of 20
limiter = TokenBucket(capacity=20, refill_rate=100/60)
```

#### Fixed Window
Simple counter that resets at fixed intervals (e.g., start of each minute):

```
[Window: 00:00 - 00:59] Limit: 100 requests
Client A sends 100 requests at 00:58 → All allowed
Client A sends 100 requests at 01:01 → All allowed (new window)
```

**Problem**: Burst at window boundaries can allow 2x the limit. In the example above, 200 requests in 3 seconds across two windows—effectively doubling the intended rate.

#### Sliding Window Log
Tracks timestamps of all requests. For each new request, count how many requests occurred in the last window duration:

```python
from collections import deque
import time

class SlidingWindowLog:
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.timestamps = deque()
    
    def allow_request(self):
        now = time.time()
        # Remove timestamps outside the window
        while self.timestamps and self.timestamps[0] <= now - self.window_seconds:
            self.timestamps.popleft()
        
        if len(self.timestamps) < self.max_requests:
            self.timestamps.append(now)
            return True
        return False
```

Accurate but memory-intensive for high volumes (storing every timestamp).

#### Sliding Window Counter (Approximate)
A memory-efficient approximation combining fixed window counters with a weighted previous window:

```
Current window: 60 requests so far (40% elapsed)
Previous window: 80 requests

Estimated count = 80 * (1 - 0.40) + 60 = 108
If limit = 100 → Reject
```

Used by Redis-based implementations (AWS API Gateway, Cloudflare).

#### Leaky Bucket
Processes requests at a constant rate, queuing excess:

```
[Requests] ──→ [Queue] ──→ [Processor (constant rate)]
                  │
              overflow → dropped
```

Unlike token bucket (which allows bursts), leaky bucket smooths traffic to a uniform rate. Good for traffic shaping but adds latency for queued requests.

### Distributed Rate Limiting

Rate limiting across multiple server instances requires shared state:

```python
import redis

class DistributedRateLimiter:
    def __init__(self, redis_client, key_prefix, max_requests, window):
        self.redis = redis_client
        self.key = f"ratelimit:{key_prefix}"
        self.max_requests = max_requests
        self.window = window
    
    def allow_request(self, client_id):
        key = f"{self.key}:{client_id}"
        current = self.redis.get(key)
        
        if current is None:
            pipe = self.redis.pipeline()
            pipe.set(key, 1)
            pipe.expire(key, self.window)
            pipe.execute()
            return True
        
        if int(current) < self.max_requests:
            self.redis.incr(key)
            return True
        
        return False

# Usage
import redis
r = redis.Redis(host='localhost', port=6379)
limiter = DistributedRateLimiter(r, 'api', 100, 60)

if limiter.allow_request('client-api-key-xyz'):
    process_request()
else:
    return {"error": "Rate limit exceeded"}, 429
```

### Rate Limiting Headers (RFC Standard)
Modern APIs should return rate limit information in response headers:

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 73
X-RateLimit-Reset: 1700000000
Retry-After: 30
```

When rate limited:
```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1700000060
Retry-After: 60
```

### Client-Side Best Practices
Clients should implement exponential backoff and jitter:

```python
import time
import random

def call_api_with_retry(url, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url)
        
        if response.status_code != 429:
            return response
        
        retry_after = int(response.headers.get('Retry-After', 1))
        # Exponential backoff with jitter
        wait = retry_after * (2 ** attempt) + random.uniform(0, 1)
        time.sleep(wait)
    
    raise Exception("Max retries exceeded")
```

## Layman's Explanation

### The Nightclub Door Policy
A popular nightclub (the API) can only hold 200 people inside (server capacity). The bouncer at the door (rate limiter) enforces a rate at which people can enter:

- **Token Bucket**: Every minute, 20 new admission tickets (tokens) become available. During slow times, tickets accumulate up to a stack of 50 (burst capacity). A VIP group of 30 can all enter at once if tickets are available. During peak time, only 20 per minute get in.

- **Fixed Window**: The bouncer resets the counter at the top of each hour. Smart clubbers arrive at 11:58 PM, get in (low current count), and at 12:01 AM, bring more friends (new window). The bouncer doesn't notice that 2x capacity entered in 3 minutes.

- **Leaky Bucket**: Like a revolving door that only lets through exactly one person per second, regardless of the crowd size. Slow and steady.

### Why Not Just Add More Club Capacity?
The $0.01 API call that processes image uploads. An abusive client or buggy script could send 10,000 images per second, racking up $100/minute in compute and storage costs. Rate limiting protects your cloud bill as much as your server health.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Algorithm Selection**: Token bucket for APIs with burst tolerance, sliding window for strict fairness, leaky bucket for traffic shaping. Wrong algorithm causes either unnecessary rejections or allows abuse at window boundaries.
- **Throttle Placement**: Rate limit at the API gateway (centralized), at individual services (defense in depth), or at the CDN edge (before traffic reaches origin)? Each has latency, granularity, and bypass risk implications.
- **Identifier Choice**: Rate limit by IP (not behind NAT/proxy), API key (reliable), user ID (requires authentication), or combination? Overly broad identifiers penalize legitimate users behind shared IPs (corporate NAT, mobile carrier CGNAT).
- **Shared State**: Distributed rate limiting requires a shared counter store (Redis, Memcached) with atomic operations. The store becomes a single point of failure and adds latency to every request.

### Business Impact
- **Cost Protection**: In serverless and cloud-native architectures, every request costs money (Lambda invocations, DynamoDB capacity units, API Gateway calls). Rate limiting directly controls cloud spend.
- **Fairness & Monetization**: Tiered rate limits drive API monetization (Free: 100/month, Pro: 10,000/month, Enterprise: unlimited). This is the business model for most SaaS APIs (Stripe, Twilio, OpenAI).
- **DDoS Mitigation**: Application-layer rate limiting at CDN/WAF level is often the first line of defense against volumetric attacks. Properly configured rate limits can absorb a 100x traffic spike without reaching origin servers.
- **Customer Trust**: Predictable, well-documented rate limits (with clear 429 responses and Retry-After headers) create a better developer experience than silently throttling or dropping requests.

## On-Premises Examples

### Nginx Rate Limiting
```nginx
# Define rate limiting zones
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_req_zone $http_x_api_key zone=api_key_limit:10m rate=100r/m;

server {
    location /api/ {
        # Per-IP burstable rate limit
        limit_req zone=api_limit burst=20 nodelay;
        
        # Per-API-key strict limit
        limit_req zone=api_key_limit burst=5;
        
        limit_req_status 429;
        proxy_pass http://backend;
    }
}
```

### HAProxy Rate Limiting
```haproxy
frontend api
    bind *:443 ssl crt /etc/ssl/certs/api.pem
    
    # Track per-IP request rate over 10 seconds
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }
    
    default_backend api_servers
```

### Kong API Gateway
```yaml
# Kong rate limiting plugin configuration
services:
  - name: order-service
    url: http://order-service:8080
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          hour: 5000
          policy: redis
          redis_host: redis-cluster
          redis_port: 6379
          fault_tolerant: true  # allow requests if Redis is down
          hide_client_headers: false
```

## AWS Examples

### AWS API Gateway Usage Plans
Tiered rate limiting per API key:

```yaml
# SAM template
UsagePlan:
  Type: AWS::ApiGateway::UsagePlan
  Properties:
    ApiStages:
      - ApiId: !Ref MyApi
        Stage: prod
    Throttle:
      BurstLimit: 200    # max burst (token bucket capacity)
      RateLimit: 100     # sustained rps
    Quota:
      Limit: 1000000     # per month
      Period: MONTH

ApiKey:
  Type: AWS::ApiGateway::ApiKey
  Properties:
    Enabled: true
    StageKeys:
      - RestApiId: !Ref MyApi
        StageName: prod

# Associate key with plan
UsagePlanKey:
  Type: AWS::ApiGateway::UsagePlanKey
  Properties:
    KeyId: !Ref ApiKey
    KeyType: API_KEY
    UsagePlanId: !Ref UsagePlan
```

### AWS WAF Rate-Based Rules
Layer 7 firewall rate limiting to block abusive IPs:

```hcl
resource "aws_wafv2_web_acl" "api" {
  name  = "api-rate-limit"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "rate-limit"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }
}
```

### ElastiCache (Redis) Distributed Rate Limiting
```python
import redis
import time

class AWSDistributedRateLimiter:
    def __init__(self, elasticache_endpoint, key_prefix):
        self.redis = redis.Redis(
            host=elasticache_endpoint,
            port=6379,
            ssl=True,
            decode_responses=True
        )
        self.prefix = key_prefix

    def is_allowed(self, client_id, max_req, window_sec, burst=0):
        """Sliding window rate limiter using sorted sets"""
        key = f"{self.prefix}:{client_id}"
        now = time.time()
        window_start = now - window_sec
        
        pipe = self.redis.pipeline()
        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)
        # Count current window
        pipe.zcard(key)
        # Add current request with jitter to prevent collisions
        pipe.zadd(key, {f"{now}:{random.getrandbits(32)}": now})
        # Set expiry
        pipe.expire(key, window_sec + 1)
        
        _, current_count, _, _ = pipe.execute()
        
        return current_count < max_req
```

## GCP Examples

### Cloud Endpoints / Apigee
```yaml
# OpenAPI spec with rate limiting
swagger: '2.0'
info:
  title: Order API
  version: 1.0.0
host: api.example.com
x-google-management:
  metrics:
    - name: "read-requests"
      displayName: "Read requests"
      valueType: INT64
      metricKind: DELTA
  quota:
    limits:
      - name: "read-limit"
        metric: "read-requests"
        unit: "1/min/{project}"
        values:
          STANDARD: 1000

paths:
  /orders:
    get:
      x-google-quota:
        metricCosts:
          "read-requests": 1
```

### Cloud Armor (WAF Rate Limiting)
```bash
gcloud compute security-policies create api-rate-limit-policy

gcloud compute security-policies rules create 1000 \
  --security-policy=api-rate-limit-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=300 \
  --conform-action=allow \
  --exceed-action=deny-429
```

### Cloud Functions Rate Limiting
```python
from google.cloud import redis_v1

def rate_limit_check(client_ip):
    client = redis_v1.CloudRedisClient()
    # Connect to Memorystore Redis instance
    # Implement token bucket or sliding window
    pass
```

## Azure Examples

### Azure API Management (Rate Limit by Key)
```xml
<policies>
    <inbound>
        <rate-limit-by-key
            calls="100"
            renewal-period="60"
            counter-key="@(context.Subscription.Id)"
            increment-condition="@(context.Response.StatusCode >= 200 && context.Response.StatusCode < 300)" />
    </inbound>
</policies>
```

### Azure Front Door WAF Rate Limiting
```json
{
  "customRules": [
    {
      "name": "rateLimitRule",
      "enabledState": "Enabled",
      "priority": 1,
      "ruleType": "RateLimitRule",
      "rateLimitDurationInMinutes": 1,
      "rateLimitThreshold": 100,
      "matchConditions": [
        {
          "matchVariable": "RemoteAddr",
          "operator": "IPMatch",
          "matchValue": ["*"]
        }
      ]
    }
  ]
}
```

### Azure Redis Cache for Distributed Rate Limiting
```python
import redis

# Connection to Azure Cache for Redis
r = redis.Redis(
    host='mycache.redis.cache.windows.net',
    port=6380,
    password=os.environ['REDIS_KEY'],
    ssl=True
)

# Token bucket implementation
LUA_SCRIPT = """
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

local elapsed = math.max(0, now - last_refill)
tokens = math.min(capacity, tokens + elapsed * rate)

if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 60)
    return 1  -- allowed
end

return 0  -- denied
"""

# Register the Lua script
check_rate = r.register_script(LUA_SCRIPT)

def allow_request(api_key):
    result = check_rate(
        keys=[f"ratelimit:{api_key}"],
        args=[100, 100/60, time.time(), 1]  # capacity, rate/s, now, cost
    )
    return result == 1
```

## Summary Decision Matrix

| Algorithm | Burst Handling | Memory Usage | Accuracy | Best For |
|-----------|---------------|--------------|----------|----------|
| Token Bucket | Excellent (configurable) | O(1) per client | High | APIs, general purpose |
| Fixed Window | Poor (boundary burst) | O(1) | Low (unfair at edges) | Simple, low-throughput |
| Sliding Window Log | Good | O(N) per client | Perfect | Accuracy-critical, low volume |
| Sliding Window Counter | Good | O(1) | Good (approximated) | High-volume, Redis-based |
| Leaky Bucket | None (smoothed) | O(1) per client | High | Traffic shaping, QoS |

Rate limiting is a critical layer in the defense-in-depth strategy for any internet-facing system. The right algorithm, placement, and client communication (headers) determine whether rate limiting protects your system or degrades your user experience. Every solution architect should know token bucket, sliding window log, and how to implement distributed rate limiting with Redis.
