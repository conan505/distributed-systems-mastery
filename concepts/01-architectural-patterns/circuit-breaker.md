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

## Real-World Examples

| Library/Tool | Language |
|--------------|----------|
| **Resilience4j** | Java |
| **Polly** | .NET |
| **Hystrix** | Java (Netflix, deprecated) |
| **Istio** | Service mesh (any language) |
| **Envoy** | Service mesh |

---

## Interview Tips

When discussing Circuit Breaker:
1. Use the electrical circuit analogy
2. Explain the three states clearly
3. Discuss cascading failure prevention
4. Mention combining with retry and timeout
5. Talk about fallback strategies

