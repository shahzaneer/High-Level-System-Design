# Retry & Backoff Patterns

Retry and backoff are the fundamental resilience patterns for handling transient failures in distributed systems. Network blips, database connection timeouts, throttled API requests—these are normal in distributed systems, not exceptions. Retry handles them gracefully. Backoff prevents retries from overwhelming struggling services (the "retry storm" problem).

The key insight: not all failures should be retried. Timeouts and 503s (Service Unavailable) are retryable. 400s (Bad Request) and 401s (Unauthorized) are not—retrying them just repeats the same error. And retries must be combined with idempotency, circuit breakers, and timeouts to form a complete resilience strategy.

## Retry Strategies

### 1. Immediate Retry (Anti-Pattern)

```python
# BAD: Hammering a failing service makes things worse
for attempt in range(5):
    try:
        return call_service()
    except:
        pass  # Immediate retry with zero delay
```

### 2. Fixed Delay Retry

```python
import time

def retry_with_fixed_delay(func, max_retries=3, delay=1.0):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(delay)
```

### 3. Exponential Backoff (Recommended)

```python
import random

def retry_with_backoff(func, max_retries=5, base_delay=1.0, max_delay=60.0):
    """
    Exponential backoff with jitter prevents thundering herd.
    Delay: 1s → 2s → 4s → 8s → 16s (with random jitter)
    """
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError as e:
            if attempt == max_retries - 1:
                raise MaxRetriesExceededError(str(e))
            
            # Exponential: base_delay * 2^attempt
            delay = min(base_delay * (2 ** attempt), max_delay)
            
            # Jitter: randomize within 50-100% of calculated delay
            # Prevents all retrying clients from hammering simultaneously
            jittered = delay * (0.5 + random.random() * 0.5)
            
            time.sleep(jittered)
```

### 4. Decorator Pattern

```python
import functools
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=1, max=60),
    retry=retry_if_exception_type((ConnectionError, TimeoutError, ServiceUnavailableError)),
    before_sleep=lambda retry_state: log.warning(
        f"Retrying {retry_state.fn.__name__}, attempt {retry_state.attempt_number}"
    )
)
def call_payment_service(order_id, amount):
    response = requests.post(
        "http://payment-service/charge",
        json={"order_id": order_id, "amount": amount},
        timeout=5
    )
    response.raise_for_status()
    return response.json()
```

## Retryable vs Non-Retryable Errors

```python
class ErrorClassifier:
    RETRYABLE = [
        408,  # Request Timeout
        429,  # Too Many Requests (with Retry-After header)
        500,  # Internal Server Error
        502,  # Bad Gateway
        503,  # Service Unavailable
        504,  # Gateway Timeout
    ]
    
    NON_RETRYABLE = [
        400,  # Bad Request (your fault, retrying won't help)
        401,  # Unauthorized (fix credentials)
        403,  # Forbidden (fix permissions)
        404,  # Not Found
        409,  # Conflict (business logic conflict)
        422,  # Unprocessable Entity
    ]
    
    @classmethod
    def should_retry(cls, response_or_exception):
        if isinstance(response_or_exception, (ConnectionError, TimeoutError)):
            return True
        
        if hasattr(response_or_exception, 'status_code'):
            return response_or_exception.status_code in cls.RETRYABLE
        
        return False
```

## Idempotency (Critical for Safe Retries)

```python
# Retrying a non-idempotent operation is dangerous:
# POST /orders → retry due to timeout → duplicate order created!

# Solution: Idempotency keys
import uuid

def create_order(order_data):
    # Client generates idempotency key BEFORE sending request
    idempotency_key = order_data.get('idempotency_key', str(uuid.uuid4()))
    
    # Server checks: have I already processed this key?
    existing = db.fetch_one(
        "SELECT result FROM idempotency WHERE key = ?", [idempotency_key]
    )
    if existing:
        return existing['result']  # Return cached result, don't re-process
    
    # Process the request
    order = actually_create_order(order_data)
    
    # Store result with idempotency key (atomic transaction)
    with db.transaction():
        db.execute(
            "INSERT INTO idempotency (key, result, created_at) VALUES (?, ?, ?)",
            [idempotency_key, json.dumps(order), datetime.utcnow()]
        )
    
    return order
```

## Combined Resilience Pattern

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, max=10),
    retry=retry_if_exception_type(TransientError)
)
def resilient_service_call(request):
    """
    Layers:
    1. Retry: handle transient failures (timeouts, 503s)
    2. Circuit Breaker: if service is consistently failing, fast-fail
    3. Timeout: don't wait forever for a response
    4. Idempotency: safe to retry without side effects
    """
    response = requests.post(
        "http://service/endpoint",
        json=request,
        timeout=5  # Per-request timeout
    )
    return response.json()
```

## Summary

| Pattern | Purpose | Key Parameter |
|---------|---------|---------------|
| Fixed delay retry | Simple retry | Delay seconds |
| Exponential backoff | Avoid overwhelming service | Base delay, max delay |
| Jitter | Prevent thundering herd | Random factor (0.5-1.0) |
| Circuit breaker | Stop retrying failing services | Failure threshold, recovery timeout |
| Idempotency | Safe retries without duplicates | Idempotency key |
| Timeout | Don't wait forever | Per-request timeout |

Retry without backoff is dangerous. Backoff without jitter creates thundering herds. Retry without idempotency creates duplicates. The complete pattern is exponential backoff + jitter + circuit breaker + idempotency + timeouts. Together, these form the resilience foundation of any distributed system.
