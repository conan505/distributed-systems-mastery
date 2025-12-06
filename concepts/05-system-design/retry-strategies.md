# Timeouts, Retries, and Backoff

## What They Are

**Timeouts** limit how long to wait for a response. **Retries** repeat failed requests. **Backoff** adds increasing delays between retries to prevent overwhelming systems.

---

## The Analogy 📞

Think of **calling someone who doesn't answer**:
- **Timeout**: Hang up after 30 seconds of ringing
- **Retry**: Call again
- **Backoff**: Wait longer between each attempt
- **Jitter**: Don't call at exactly the same intervals

---

## Why They Matter

### The Problem: Network Failures
```
Without proper handling:

Request times out → Retry immediately
1000 clients do this simultaneously
Server recovers → Receives 1000 retries at once
Server crashes again → More retries
→ System never recovers (thundering herd)

With proper handling:
Exponential backoff + jitter spreads load
Server recovers gradually
```

---

## Timeout Strategies

### Types of Timeouts
```
┌─────────────────────────────────────────────────────────────────┐
│                     Timeout Types                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Connection timeout: Time to establish connection             │
│   └── Usually short: 1-5 seconds                               │
│                                                                  │
│   Read/Socket timeout: Time waiting for response                │
│   └── Depends on operation: 5-30 seconds                       │
│                                                                  │
│   Request timeout: Total time for entire request               │
│   └── Sum of all operations: 30-60 seconds                     │
│                                                                  │
│   Idle timeout: Time before closing idle connection            │
│   └── Longer: 60-300 seconds                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Setting Timeouts
```python
import requests

response = requests.get(
    'https://api.example.com/data',
    timeout=(3.05, 27)  # (connect_timeout, read_timeout)
)

# Or single timeout for both
response = requests.get(url, timeout=30)
```

---

## Retry Strategies

### When to Retry
```
✅ Retry:
- 408 Request Timeout
- 429 Too Many Requests (with backoff)
- 500 Internal Server Error
- 502 Bad Gateway
- 503 Service Unavailable
- 504 Gateway Timeout
- Network errors (connection refused, DNS failure)

❌ Don't retry:
- 400 Bad Request (client error, won't help)
- 401/403 (auth issues)
- 404 Not Found
- 409 Conflict
- Non-idempotent operations (without idempotency key)
```

### Retry Budget
```python
class RetryBudget:
    """Limit total retries to prevent amplification."""
    
    def __init__(self, budget_percent: float = 10):
        self.budget_percent = budget_percent
        self.requests = 0
        self.retries = 0
    
    def can_retry(self) -> bool:
        if self.requests == 0:
            return True
        retry_rate = self.retries / self.requests
        return retry_rate < (self.budget_percent / 100)
    
    def record_request(self, is_retry: bool):
        self.requests += 1
        if is_retry:
            self.retries += 1
```

---

## Backoff Strategies

### 1. Fixed Delay
```python
def fixed_backoff(attempt: int) -> float:
    return 1.0  # Always 1 second

# Attempts: 1s, 1s, 1s, 1s...
# ❌ Doesn't adapt, can cause thundering herd
```

### 2. Linear Backoff
```python
def linear_backoff(attempt: int) -> float:
    return attempt * 1.0  # 1s, 2s, 3s, 4s...

# ❌ Still predictable, can synchronize retries
```

### 3. Exponential Backoff
```python
def exponential_backoff(attempt: int, base: float = 1.0) -> float:
    return base * (2 ** attempt)  # 1s, 2s, 4s, 8s, 16s...

# ✅ Increases delay rapidly
# ❌ Clients still synchronized
```

### 4. Exponential Backoff with Jitter (Best)
```python
import random

def exponential_backoff_with_jitter(
    attempt: int,
    base: float = 1.0,
    max_delay: float = 60.0
) -> float:
    """Full jitter as recommended by AWS."""
    exp_delay = base * (2 ** attempt)
    capped_delay = min(exp_delay, max_delay)
    return random.uniform(0, capped_delay)

# ✅ Spreads retries over time
# ✅ Prevents thundering herd
```

---

## Jitter Strategies

```python
def no_jitter(delay: float) -> float:
    return delay

def full_jitter(delay: float) -> float:
    """Random between 0 and delay."""
    return random.uniform(0, delay)

def equal_jitter(delay: float) -> float:
    """Half fixed, half random."""
    return delay / 2 + random.uniform(0, delay / 2)

def decorrelated_jitter(delay: float, prev_delay: float, base: float) -> float:
    """Based on previous delay."""
    return random.uniform(base, prev_delay * 3)
```

### Comparison
```
Full jitter: Best for many clients
Equal jitter: Good balance
Decorrelated: Best for single client
```

---

## Complete Implementation

```python
import time
import random
from typing import TypeVar, Callable

T = TypeVar('T')

def retry_with_backoff(
    func: Callable[[], T],
    max_attempts: int = 5,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    retryable_exceptions: tuple = (Exception,)
) -> T:
    """
    Retry function with exponential backoff and jitter.
    """
    for attempt in range(max_attempts):
        try:
            return func()
        except retryable_exceptions as e:
            if attempt == max_attempts - 1:
                raise  # Last attempt, re-raise
            
            # Calculate delay with full jitter
            exp_delay = base_delay * (2 ** attempt)
            delay = random.uniform(0, min(exp_delay, max_delay))
            
            print(f"Attempt {attempt + 1} failed: {e}")
            print(f"Retrying in {delay:.2f} seconds...")
            
            time.sleep(delay)

# Usage
result = retry_with_backoff(
    lambda: requests.get(url, timeout=10),
    max_attempts=5,
    retryable_exceptions=(requests.RequestException,)
)
```

---

## Library Examples

### Python (tenacity)
```python
from tenacity import retry, stop_after_attempt, wait_exponential_jitter

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=1, max=60)
)
def call_api():
    return requests.get(url)
```

### Java (Resilience4j)
```java
RetryConfig config = RetryConfig.custom()
    .maxAttempts(5)
    .waitDuration(Duration.ofSeconds(1))
    .retryOnException(e -> e instanceof IOException)
    .build();
```

---

## Best Practices

```
1. Always set timeouts (never infinite wait)
2. Use exponential backoff with jitter
3. Cap maximum delay (60s typical)
4. Limit total attempts (3-5 typical)
5. Only retry idempotent operations
6. Implement retry budget
7. Log retry attempts for debugging
8. Respect Retry-After headers
```

---

## Interview Tips

When discussing Retry Strategies:
1. Explain the thundering herd problem
2. Describe exponential backoff with jitter
3. Know when to retry (5xx) vs not (4xx)
4. Discuss idempotency requirement
5. Mention retry budgets

