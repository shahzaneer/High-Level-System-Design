# Circuit Breakers

## Introduction
Circuit Breakers are a resilience pattern that prevents cascading failures in distributed systems. The concept was popularized by Michael Nygard in his book "Release It!" (2007), drawing an analogy to electrical circuit breakers that stop current flow when detecting a fault to prevent fire. In software, the circuit breaker detects when a downstream service is failing and "opens the circuit" to stop sending requests, giving the failing service time to recover while protecting upstream services from resource exhaustion.

This pattern became essential with the rise of microservices, where a single service failure can cascade across dozens of dependent services. Companies like Netflix built entire resilience frameworks (Hystrix) around this concept. Today, circuit breakers are built into service meshes (Istio, Linkerd), API gateways, and cloud load balancers.

## Definition
A **Circuit Breaker** is a software design pattern that wraps a potentially failing function call with a monitor that tracks failures. When failures reach a threshold, the circuit "opens" and subsequent calls fail immediately without executing the actual function. After a timeout, the circuit transitions to "half-open," allowing a limited number of test calls. If they succeed, the circuit "closes" and normal operation resumes. If they fail, the circuit re-opens.

### Circuit Breaker States

```
     ┌──────────────────────────────────┐
     │                                  │
     ▼                                  │
 [CLOSED] ──failures > threshold──→ [OPEN]
     ▲                                  │
     │                           timeout expires
     │                                  │
     │                                  ▼
     └───successes > threshold── [HALF-OPEN]
                                 test calls fail ──→ back to OPEN
```

### State Details

**CLOSED (Normal Operation)**:
- Requests flow normally to the downstream service
- Failure counter tracks consecutive or recent failures
- If failures exceed threshold within a window, transition to OPEN

**OPEN (Failure State)**:
- Requests are immediately rejected with an error (fail-fast)
- No calls are made to the failing downstream service
- After a configured timeout (e.g., 30 seconds), transitions to HALF-OPEN
- The downstream service gets time to recover without additional load

**HALF-OPEN (Recovery Testing)**:
- A limited number of probe requests are allowed through
- If probes succeed → CLOSED (service has recovered)
- If any probe fails → OPEN (service still failing, reset timeout)

## Concept Explanation

### Why Circuit Breakers Are Essential

Without a circuit breaker, a typical cascading failure scenario:

```
[Service A] ──call──→ [Service B] (slow due to DB issue)
     │
     ├── Each call to B blocks a thread in A for 30 seconds (timeout)
     ├── Incoming requests accumulate in A's thread pool
     ├── A's thread pool exhausted → A can't process any requests
     ├── [Service C] ──call──→ [Service A] (now failing)
     ├── C's thread pool also exhausts
     └── Cascade continues through entire system
```

With a circuit breaker:

```
[Service A] ──call──→ [Circuit Breaker] ──call──→ [Service B] (failing)
     │                      │
     │                    failures > 5 in 10 seconds → OPEN
     │                      │
     ├── CB fails immediately, thread released in microseconds
     ├── A remains responsive, serves fallback responses
     └── No cascading failure
```

### Implementation

```python
import time
import functools
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30,
                 half_open_max_calls=3, failure_window=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls
        self.failure_window = failure_window
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0
        self.half_open_calls = 0
    
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
            else:
                raise CircuitBreakerOpenError("Circuit is OPEN")
        
        if self.state == CircuitState.HALF_OPEN:
            if self.half_open_calls >= self.half_open_max_calls:
                raise CircuitBreakerOpenError("Half-open test limit reached")
            self.half_open_calls += 1
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED
        self.failure_count = 0
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if (self.state == CircuitState.HALF_OPEN or
            self.failure_count >= self.failure_threshold):
            self.state = CircuitState.OPEN

class CircuitBreakerOpenError(Exception):
    pass

# Usage as a decorator
def circuit_breaker(failure_threshold=5, recovery_timeout=30):
    cb = CircuitBreaker(failure_threshold, recovery_timeout)
    
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            return cb.call(func, *args, **kwargs)
        return wrapper
    return decorator

@circuit_breaker(failure_threshold=3, recovery_timeout=15)
def call_payment_service(amount):
    response = requests.post(
        "http://payment-service/charge",
        json={"amount": amount},
        timeout=5
    )
    if response.status_code >= 500:
        raise Exception(f"Payment service error: {response.status_code}")
    return response.json()
```

### Fallback Strategies

When the circuit is OPEN, you need fallback behavior:

**Graceful Degradation**: Return a degraded but usable response:
```python
@circuit_breaker(failure_threshold=3)
def get_product_recommendations(user_id):
    response = requests.get(f"http://recommendation-service/user/{user_id}")
    return response.json()

# Fallback: return generic popular products
def get_recommendations_with_fallback(user_id):
    try:
        return get_product_recommendations(user_id)
    except CircuitBreakerOpenError:
        return {"recommendations": get_popular_products(), "fallback": True}
```

**Cache Fallback**: Return stale cached data:
```python
def get_user_profile_with_fallback(user_id):
    try:
        profile = get_user_profile_from_service(user_id)
        cache.set(f"profile:{user_id}", profile, ttl=300)
        return profile
    except CircuitBreakerOpenError:
        cached = cache.get(f"profile:{user_id}")
        if cached:
            return {**cached, "stale": True}
        return {"error": "Service unavailable"}
```

**Default Values**: Return sensible defaults:
```python
def get_shipping_cost_with_fallback(order):
    try:
        return shipping_service.calculate(order)
    except CircuitBreakerOpenError:
        return {"cost": order.total * 0.10, "method": "standard", "estimated": True}
```

### Bulkhead Pattern (Companion to Circuit Breaker)
Isolate resources into separate pools so one failing component doesn't consume all resources:

```python
from concurrent.futures import ThreadPoolExecutor

class BulkheadExecutor:
    """Isolate service calls into separate thread pools"""
    def __init__(self):
        self.executors = {
            'payment': ThreadPoolExecutor(max_workers=10),
            'inventory': ThreadPoolExecutor(max_workers=10),
            'shipping': ThreadPoolExecutor(max_workers=5),
        }
    
    def call(self, service_name, func, *args, **kwargs):
        executor = self.executors.get(service_name)
        if not executor:
            raise ValueError(f"No executor for {service_name}")
        return executor.submit(func, *args, **kwargs)
```

### Monitoring Circuit Breaker Health
Expose circuit breaker state for observability:

```python
class MonitoredCircuitBreaker(CircuitBreaker):
    def __init__(self, name, metrics_client, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.name = name
        self.metrics = metrics_client
    
    def _on_success(self):
        super()._on_success()
        self.metrics.increment(f"circuit_breaker.{self.name}.success")
        self.metrics.gauge(f"circuit_breaker.{self.name}.state", 0)  # CLOSED
    
    def _on_failure(self):
        super()._on_failure()
        self.metrics.increment(f"circuit_breaker.{self.name}.failure")
        if self.state == CircuitState.OPEN:
            self.metrics.gauge(f"circuit_breaker.{self.name}.state", 1)  # OPEN
            self.metrics.event(f"Circuit breaker {self.name} OPEN")
```

## Layman's Explanation

### The Electrical Circuit Breaker Analogy
Your home has a circuit breaker panel. When you plug too many devices into one outlet and they draw too much current:
1. The circuit breaker detects the overload (failure threshold exceeded)
2. It "trips" (opens the circuit), cutting power to that outlet
3. This prevents the wires from overheating and starting a fire (cascading failure)
4. After you unplug some devices and wait, you manually flip the breaker back on (half-open testing)
5. If the wiring is fine, power resumes (closes)
6. If it trips again immediately, something is still wrong (re-opens)

Without the breaker: the wires overheat, the fire spreads to the walls, then to the whole house. That's a cascading failure in a distributed system.

### The Restaurant Kitchen Analogy
The head chef (Service A) sends orders to the pastry chef (Service B). One day:
- The pastry chef's oven breaks, and every order takes 45 minutes instead of 5
- The head chef keeps sending orders, each occupying a chef for 45 minutes waiting
- Soon, all chefs are waiting on pastry. No appetizers, no mains, no desserts go out.
- The restaurant is effectively closed because of one broken oven.

With a circuit breaker (the kitchen manager):
- After 3 pastry orders take too long, the manager says "Pastry section is CLOSED"
- Waiters are told immediately, "Sorry, no dessert tonight"
- They stop sending orders, and pastry chef has time to fix the oven
- After 20 minutes, the manager sends one test order (half-open)
- If it's fast, the pastry section reopens. If not, it stays closed.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Threshold Calibration**: Too sensitive (low threshold) and transient network blips trip the circuit unnecessarily. Too insensitive and the protection is ineffective before resource exhaustion occurs. Must be tuned based on service characteristics.
- **Timeout Strategy**: Circuit breaker should have its own timeout that is shorter than the overall request timeout. If the downstream service typically responds in 100ms but can take up to 5 seconds, the circuit breaker timeout should trip at 2 seconds of failures.
- **Fallback Design**: Every service dependency needs a planned fallback. No fallback means the breaker just returns an error faster (fail-fast) rather than gracefully degrading. Users prefer stale data over no data.
- **Recovery Behavior**: HALF-OPEN probing must be gentle—sending too many probe requests can keep a struggling service down. Gradual ramp-up of traffic is better than an on-off switch.

### Business Impact
- **System Resilience**: Netflix's Hystrix library processed billions of requests daily, preventing countless cascading failures. Without circuit breakers, a minor database slowdown could take down the entire streaming platform during peak hours.
- **User Experience During Failures**: A credit card processing outage shouldn't prevent users from browsing products. Circuit breakers with fallbacks keep the core experience alive even when dependent services fail.
- **Reduced MTTR (Mean Time To Recovery)**: Fail-fast circuit breakers prevent resource exhaustion, which means when the downstream service recovers, the upstream service is still healthy and can resume immediately without restart or manual intervention.
- **Operational Confidence**: Teams can deploy, experiment, and scale services knowing that failures in one service won't cascade. This enables continuous deployment and faster innovation cycles.

### Anti-Patterns
- **No circuit breaker**: The original sin of microservices. Every synchronous call to an external service should be wrapped.
- **Global circuit breaker**: One breaker for all downstream calls. A failing payment service would also block shipping service calls. Breakers should be scoped per dependency.
- **Silent fail-fast**: Circuit open with no logging, metrics, or alerting. The operations team has no idea a breaker is open until customers complain. Every state transition should generate an event.
- **Infinite retries with no backoff**: Retrying a failing service without backoff or circuit breaking creates a "retry storm" that can amplify the original failure.

## On-Premises Examples

### Python (pybreaker library)
```python
import pybreaker
import requests

# Configure circuit breaker
db_breaker = pybreaker.CircuitBreaker(
    fail_max=5,              # open after 5 failures
    reset_timeout=30,        # try half-open after 30 seconds
    exclude=[requests.exceptions.HTTPError]  # don't count 4xx as failures
)

@db_breaker
def query_database(query):
    response = requests.post("http://db-proxy/query", json={"sql": query})
    if response.status_code == 503:
        raise pybreaker.CircuitBreakerError("Database unavailable")
    return response.json()

# With fallback listener
def on_open(cb, exc):
    logger.error(f"Circuit breaker OPEN: {exc}")
    alerting.send_alert("DB circuit breaker tripped")

db_breaker.add_on_open_listener(on_open)
```

### Resilience4j (Java)
```java
// CircuitBreaker configuration
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)           // 50% failure rate opens circuit
    .slowCallRateThreshold(50)          // 50% slow calls also count
    .slowCallDurationThreshold(Duration.ofSeconds(2))
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .permittedNumberOfCallsInHalfOpenState(3)
    .slidingWindowType(SlidingWindowType.COUNT_BASED)
    .slidingWindowSize(10)              // evaluate last 10 calls
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("paymentService", config);

// Decorate function with circuit breaker
Supplier<PaymentResponse> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> paymentService.charge(amount));

// Execute with fallback
Try<PaymentResponse> result = Try.ofSupplier(decoratedSupplier)
    .recover(throwable -> fallbackResponse());
```

### Istio Service Mesh (Envoy Sidecar)
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-circuit-breaker
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      # Ejects unhealthy hosts from load balancing pool
```

### Nginx Circuit Breaker (Health Checks)
```nginx
upstream backend {
    server backend1.example.com max_fails=3 fail_timeout=30s;
    server backend2.example.com max_fails=3 fail_timeout=30s;
}
# After 3 failures, Nginx stops sending traffic to that server for 30 seconds
```

## AWS Examples

### AWS App Mesh (Envoy-based Circuit Breaking)
```json
{
  "spec": {
    "listener": {
      "connectionPool": {
        "http": {
          "maxConnections": 100,
          "maxPendingRequests": 50
        }
      },
      "outlierDetection": {
        "maxServerErrors": 5,
        "interval": {
          "unit": "s",
          "value": 10
        },
        "baseEjectionDuration": {
          "unit": "s",
          "value": 30
        },
        "maxEjectionPercent": 50
      }
    }
  }
}
```

### AWS Lambda with Circuit Breaker (DynamoDB Parameter Store)
```python
import boto3
import json

ssm = boto3.client('ssm')

class DynamoDBCircuitBreaker:
    def __init__(self, service_name):
        self.service_name = service_name
        self.param_name = f"/circuit-breaker/{service_name}"
    
    def get_state(self):
        try:
            response = ssm.get_parameter(Name=self.param_name)
            return json.loads(response['Parameter']['Value'])
        except ssm.exceptions.ParameterNotFound:
            return {'state': 'CLOSED', 'failures': 0}
    
    def record_failure(self):
        state = self.get_state()
        state['failures'] += 1
        if state['failures'] >= 5:
            state['state'] = 'OPEN'
            state['opened_at'] = int(time.time())
        self._save_state(state)
    
    def is_open(self):
        state = self.get_state()
        if state['state'] == 'OPEN':
            if time.time() - state['opened_at'] >= 30:
                state['state'] = 'HALF_OPEN'
                state['failures'] = 0
                self._save_state(state)
                return False
            return True
        return False
    
    def _save_state(self, state):
        ssm.put_parameter(
            Name=self.param_name,
            Value=json.dumps(state),
            Type='String',
            Overwrite=True
        )
```

### AWS RDS Proxy Circuit Breaking (Built-in)
RDS Proxy handles connection pooling with implicit circuit breaking—when database connections are exhausted, it queues or rejects new connections rather than passing failures to the application.

### Route 53 Health Checks (DNS-Level Circuit Breaking)
```hcl
resource "aws_route53_health_check" "app" {
  fqdn              = "app.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  # DNS failover: if health check fails, Route 53 routes to failover endpoint
}
```

## GCP Examples

### Istio on GKE (Service Mesh Circuit Breaker)
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-circuit-breaker
spec:
  host: payment-service.default.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 100
```

### Cloud Load Balancing Health Checks
```bash
gcloud compute health-checks create http app-health-check \
  --port=8080 \
  --request-path=/health \
  --check-interval=10s \
  --timeout=5s \
  --unhealthy-threshold=3 \    # 3 failures → unhealthy
  --healthy-threshold=2        # 2 successes → healthy
```

### Cloud Run Circuit Breaker via Traffic Splitting
```bash
# Progressive rollback if new revision has high error rate
gcloud run services update-traffic my-service \
  --to-revisions=LATEST=100 \
  --to-latest
  
# Manual rollback (circuit breaker opened by operator/monitoring)
gcloud run services update-traffic my-service \
  --to-revisions=PREVIOUS=100
```

## Azure Examples

### Azure Application Gateway Health Probes
```json
{
  "properties": {
    "protocol": "Http",
    "host": "app.example.com",
    "path": "/health",
    "interval": 30,
    "timeout": 30,
    "unhealthyThreshold": 3,
    "pickHostNameFromBackendAddress": false
  }
}
```

### Azure Service Fabric Circuit Breaker
```csharp
// Polly library for circuit breaker
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .Or<TimeoutRejectedException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (ex, ts) => {
            logger.LogError($"Circuit BREAKER OPEN: {ex.Message}");
        },
        onReset: () => {
            logger.LogInformation("Circuit breaker RESET");
        }
    );

var response = await circuitBreakerPolicy.ExecuteAsync(
    () => httpClient.GetAsync("http://payment-service/charge")
);
```

### Azure API Management Circuit Breaker Policy
```xml
<policies>
    <backend>
        <forward-request timeout="10" />
    </backend>
    <on-error>
        <!-- Circuit breaker behavior on failure -->
        <choose>
            <when condition="@(context.LastError.Reason == "Timeout")">
                <return-response>
                    <set-status code="503" reason="Service temporarily unavailable" />
                    <set-body>Service is temporarily unavailable. Please try again later.</set-body>
                    <set-header name="Retry-After" exists-action="override">
                        <value>30</value>
                    </set-header>
                </return-response>
            </when>
        </choose>
    </on-error>
</policies>
```

## Summary Decision Matrix

| Circuit Breaker Implementation | Best For | Complexity | Notes |
|------------------------------|----------|------------|-------|
| Application-level (pybreaker, Resilience4j, Polly) | Fine-grained control, per-dependency | Low-Medium | Most flexible; code coupling |
| Service Mesh (Istio/Envoy) | Infrastructure-level, no code changes | Medium | Sidecar proxy overhead; uniform behavior |
| API Gateway (Kong, Apigee) | North-south traffic, client-facing APIs | Medium | Limited to ingress; not service-to-service |
| Load Balancer Health Checks | TCP/HTTP level, simple pass/fail | Low | Coarse-grained; doesn't differentiate error types |
| Cloud Platform Built-in | AWS RDS Proxy, Cloud Run revision management | Low | Provider-specific; limited customization |

Circuit breakers are a foundational resilience pattern for distributed systems. They prevent cascading failures by failing fast and giving downstream services time to recover. Every synchronous cross-service call should be wrapped with a circuit breaker, a timeout, and a fallback strategy. Without these three layers of defense, microservices architectures become fragile chains where any link failure brings down the entire system.
