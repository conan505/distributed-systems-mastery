# Circuit Breaker Pattern

## What It Is

The **Circuit Breaker Pattern** prevents an application from repeatedly trying to execute an operation that's likely to fail. Like an electrical circuit breaker, it "trips" when failures reach a threshold, stopping requests until the downstream service recovers.

---

## The Analogy ⚡

Think of **electrical circuit breakers** in your home:
- Normal: Electricity flows freely
- Overload: Too much current (short circuit)
- Breaker trips: Stops electricity flow
- Manual reset: After fixing the problem, flip the switch

Without breakers: Wires overheat → Fire → House burns down

In software:
- Normal: Requests flow to service
- Failures: Service is slow/down
- Circuit opens: Stop sending requests
- Recovery: Gradually resume after service recovers

---

## Why It Exists

### The Problem: Cascading Failures
```
Service A → Service B (down)
    │
    └── Keeps retrying...
    └── Thread pool exhausted
    └── A becomes slow
    └── Service C → A (slow)
    └── C becomes slow
    └── Entire system collapses!
```

### Without Circuit Breaker:
- Threads blocked waiting for timeouts
- Resources exhausted (connections, memory)
- Latency spreads across the system
- Recovery takes longer (thundering herd on restart)

### What Circuit Breaker Solves:
- **Fail fast** - Don't wait for timeout
- **Prevent resource exhaustion** - Free up threads immediately
- **Allow recovery** - Give failing service breathing room
- **Graceful degradation** - Return fallback response

---

## How It Works

### Three States:
```
┌─────────────────────────────────────────────────────────────────┐
│                    Circuit Breaker States                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│        ┌──────────────┐                                         │
│        │    CLOSED    │ ◀─── Normal operation                   │
│        │  (Healthy)   │      All requests go through            │
│        └──────┬───────┘                                         │
│               │                                                  │
│               │ Failure threshold exceeded                       │
│               ▼                                                  │
│        ┌──────────────┐                                         │
│        │     OPEN     │ ◀─── Fail immediately                   │
│        │   (Tripped)  │      No requests sent                   │
│        └──────┬───────┘                                         │
│               │                                                  │
│               │ Timeout expires                                  │
│               ▼                                                  │
│        ┌──────────────┐                                         │
│        │  HALF-OPEN   │ ◀─── Test with limited requests         │
│        │  (Testing)   │      If success → CLOSED                │
│        └──────────────┘      If failure → OPEN                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### State Transitions:
| From | Trigger | To |
|------|---------|-----|
| CLOSED | Failure rate > threshold | OPEN |
| OPEN | Wait timeout expires | HALF-OPEN |
| HALF-OPEN | Test request succeeds | CLOSED |
| HALF-OPEN | Test request fails | OPEN |

---

## Implementation Example (Resilience4j)

```java
// Configure circuit breaker
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)              // Open at 50% failures
    .waitDurationInOpenState(Duration.ofSeconds(30))  // Wait 30s before half-open
    .slidingWindowSize(10)                 // Evaluate last 10 calls
    .minimumNumberOfCalls(5)               // Need 5 calls before evaluation
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("paymentService", config);

// Use the circuit breaker
Supplier<String> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> paymentService.process());

// With fallback
String result = Try.ofSupplier(decoratedSupplier)
    .recover(throwable -> "Payment unavailable, try later")
    .get();
```

---

## When to Use It

### ✅ Good Fit:
- **External API calls** - Third-party services
- **Database connections** - When DB is overloaded
- **Microservice communication** - Service-to-service calls
- **Any remote call** - Network can fail

### ❌ Less Useful When:
- Local in-memory operations
- Operations that must complete (use queues instead)
- Fire-and-forget async operations

---

## How to Use It Effectively

### Best Practices:

1. **Configure thresholds carefully**
   - Too sensitive: Opens too often (false positives)
   - Too lenient: Doesn't protect (cascading failures)

2. **Implement meaningful fallbacks**
   ```java
   // Good fallback examples:
   - Return cached data
   - Return default/empty response
   - Queue for later processing
   - Return degraded functionality
   ```

3. **Monitor circuit state**
   - Alert when circuit opens
   - Track open/closed transitions
   - Measure fallback usage

4. **Combine with other patterns**
   - Retry (before circuit trips)
   - Timeout (don't wait forever)
   - Bulkhead (isolate failures)

5. **Test circuit breaker behavior**
   - Chaos engineering
   - Simulate downstream failures

---

## Circuit Breaker vs Retry

| Aspect | Retry | Circuit Breaker |
|--------|-------|-----------------|
| **Purpose** | Handle transient failures | Prevent cascading failures |
| **When** | Before giving up | After repeated failures |
| **Duration** | Short-term | Longer-term |
| **Scope** | Single request | All requests |

**Use together:**
```
Request → Retry (3 times) → Circuit Breaker → Fallback
```

---

## Implementation Deep Dive

### Production-Ready Circuit Breaker (Python)

```python
import time
from enum import Enum
from threading import Lock
from typing import Callable, Optional, TypeVar
from functools import wraps

T = TypeVar('T')

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing fast
    HALF_OPEN = "half_open"  # Testing recovery

class CircuitBreaker:
    """
    Thread-safe circuit breaker with sliding window failure tracking.
    """

    def __init__(
        self,
        failure_threshold: int = 5,      # Failures to trip
        success_threshold: int = 3,       # Successes to close
        timeout: float = 30.0,            # Seconds before half-open
        expected_exceptions: tuple = (Exception,)
    ):
        self.failure_threshold = failure_threshold
        self.success_threshold = success_threshold
        self.timeout = timeout
        self.expected_exceptions = expected_exceptions

        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._success_count = 0
        self._last_failure_time: Optional[float] = None
        self._lock = Lock()

    @property
    def state(self) -> CircuitState:
        with self._lock:
            if self._state == CircuitState.OPEN:
                # Check if timeout expired → transition to half-open
                if time.time() - self._last_failure_time >= self.timeout:
                    self._state = CircuitState.HALF_OPEN
                    self._success_count = 0
            return self._state

    def call(self, func: Callable[[], T], fallback: Optional[Callable[[], T]] = None) -> T:
        """Execute function with circuit breaker protection."""

        if self.state == CircuitState.OPEN:
            if fallback:
                return fallback()
            raise CircuitOpenError("Circuit is OPEN")

        try:
            result = func()
            self._on_success()
            return result
        except self.expected_exceptions as e:
            self._on_failure()
            if fallback:
                return fallback()
            raise

    def _on_success(self):
        with self._lock:
            if self._state == CircuitState.HALF_OPEN:
                self._success_count += 1
                if self._success_count >= self.success_threshold:
                    self._state = CircuitState.CLOSED
                    self._failure_count = 0
            elif self._state == CircuitState.CLOSED:
                self._failure_count = 0  # Reset on success

    def _on_failure(self):
        with self._lock:
            self._failure_count += 1
            self._last_failure_time = time.time()

            if self._state == CircuitState.HALF_OPEN:
                self._state = CircuitState.OPEN
            elif self._failure_count >= self.failure_threshold:
                self._state = CircuitState.OPEN

class CircuitOpenError(Exception):
    pass

# Decorator usage
def circuit_breaker(
    failure_threshold: int = 5,
    timeout: float = 30.0,
    fallback: Optional[Callable] = None
):
    cb = CircuitBreaker(failure_threshold=failure_threshold, timeout=timeout)

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            return cb.call(lambda: func(*args, **kwargs), fallback)
        wrapper.circuit_breaker = cb  # Expose for monitoring
        return wrapper
    return decorator

# Usage
@circuit_breaker(failure_threshold=3, timeout=60, fallback=lambda: {"status": "cached"})
def call_payment_service(order_id: str):
    return requests.post(f"https://payment.api/charge/{order_id}").json()
```

### Sliding Window Implementation

```python
from collections import deque
from dataclasses import dataclass

@dataclass
class CallResult:
    success: bool
    timestamp: float

class SlidingWindowCircuitBreaker:
    """
    Circuit breaker with time-based sliding window for more accurate failure rate.
    """

    def __init__(
        self,
        failure_rate_threshold: float = 0.5,  # 50% failure rate
        window_size_seconds: float = 60.0,
        minimum_calls: int = 10,
        timeout: float = 30.0
    ):
        self.failure_rate_threshold = failure_rate_threshold
        self.window_size = window_size_seconds
        self.minimum_calls = minimum_calls
        self.timeout = timeout

        self._calls: deque[CallResult] = deque()
        self._state = CircuitState.CLOSED
        self._last_failure_time: Optional[float] = None
        self._lock = Lock()

    def _cleanup_old_calls(self):
        """Remove calls outside the sliding window."""
        cutoff = time.time() - self.window_size
        while self._calls and self._calls[0].timestamp < cutoff:
            self._calls.popleft()

    def _calculate_failure_rate(self) -> float:
        self._cleanup_old_calls()
        if len(self._calls) < self.minimum_calls:
            return 0.0
        failures = sum(1 for c in self._calls if not c.success)
        return failures / len(self._calls)

    def _record_call(self, success: bool):
        with self._lock:
            self._calls.append(CallResult(success, time.time()))

            if not success:
                self._last_failure_time = time.time()

            if self._state == CircuitState.CLOSED:
                if self._calculate_failure_rate() >= self.failure_rate_threshold:
                    self._state = CircuitState.OPEN
```

### Metrics & Monitoring

```python
from dataclasses import dataclass, field
from typing import Dict

@dataclass
class CircuitBreakerMetrics:
    """Metrics for observability."""
    total_calls: int = 0
    successful_calls: int = 0
    failed_calls: int = 0
    rejected_calls: int = 0  # When circuit is open
    state_transitions: Dict[str, int] = field(default_factory=dict)

    def record_success(self):
        self.total_calls += 1
        self.successful_calls += 1

    def record_failure(self):
        self.total_calls += 1
        self.failed_calls += 1

    def record_rejection(self):
        self.rejected_calls += 1

    def record_state_change(self, from_state: str, to_state: str):
        key = f"{from_state}->{to_state}"
        self.state_transitions[key] = self.state_transitions.get(key, 0) + 1

    @property
    def failure_rate(self) -> float:
        if self.total_calls == 0:
            return 0.0
        return self.failed_calls / self.total_calls

# Export to Prometheus
def export_metrics(cb: CircuitBreaker, service_name: str):
    return {
        f"circuit_breaker_state{{service=\"{service_name}\"}}": cb.state.value,
        f"circuit_breaker_failure_count{{service=\"{service_name}\"}}": cb._failure_count,
    }
```

---

## Real-World Examples

| Library/Tool | Language | Notes |
|--------------|----------|-------|
| **Resilience4j** | Java | Modern, lightweight, functional |
| **Polly** | .NET | Comprehensive resilience library |
| **Hystrix** | Java | Netflix (deprecated, use Resilience4j) |
| **Istio** | Service mesh | Sidecar-based, language-agnostic |
| **Envoy** | Service mesh | High-performance proxy |

---

## Interview Tips

When discussing Circuit Breaker:
1. Use the electrical circuit analogy
2. Explain the three states clearly (CLOSED → OPEN → HALF-OPEN)
3. Discuss cascading failure prevention
4. Mention combining with retry and timeout
5. Talk about fallback strategies (cached data, default response)
6. Explain sliding window vs count-based failure tracking

