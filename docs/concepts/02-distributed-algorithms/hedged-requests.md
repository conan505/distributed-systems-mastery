# Hedged Requests (Tail Latency Reduction)

## What It Is

**Hedged Requests** is a technique to reduce tail latency by sending the same request to multiple servers and using the first response, canceling the others.

---

## The Analogy 🏇

Think of **betting on multiple horses**:
- Bet on 3 horses in the same race
- You win as soon as ANY of them wins
- Cancel losing bets when first horse finishes
- Higher cost, but guaranteed faster result

---

## Why It Exists

### The Problem: Tail Latency
```
Latency distribution:
- p50: 10ms  (median)
- p99: 100ms (1% of requests)
- p999: 500ms (0.1% of requests)

For a page requiring 100 backend calls:
- Probability ALL are fast: 0.99^100 = 37%
- 63% of page loads hit at least one slow request!

Tail latency dominates user experience
```

### What Hedged Requests Solve:
- **Reduce tail latency** - First response wins
- **Mask slow servers** - Don't wait for stragglers
- **Improve consistency** - More predictable latency

---

## How It Works

### Basic Hedging
```
┌─────────────────────────────────────────────────────────────────┐
│                    Hedged Request Flow                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Client sends request to Server A                              │
│                                                                  │
│   t=0ms:  Request ──▶ [Server A]                                │
│                                                                  │
│   t=10ms: No response yet, hedge!                               │
│           Request ──▶ [Server B]                                │
│                                                                  │
│   t=15ms: [Server B] responds first ✓                           │
│           Cancel request to Server A                            │
│           Return response to client                             │
│                                                                  │
│   Result: 15ms instead of waiting for slow Server A             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation
```python
import asyncio
from typing import Any

async def hedged_request(
    request_func,
    servers: list,
    hedge_delay_ms: float = 10
) -> Any:
    """Send hedged requests with delay between attempts."""
    
    tasks = []
    
    async def make_request(server):
        return await request_func(server)
    
    # Start first request
    tasks.append(asyncio.create_task(make_request(servers[0])))
    
    for server in servers[1:]:
        # Wait for hedge delay or first response
        done, pending = await asyncio.wait(
            tasks,
            timeout=hedge_delay_ms / 1000,
            return_when=asyncio.FIRST_COMPLETED
        )
        
        if done:
            # Got response, cancel pending and return
            for task in pending:
                task.cancel()
            return done.pop().result()
        
        # No response yet, send hedge request
        tasks.append(asyncio.create_task(make_request(server)))
    
    # Wait for first response
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )
    
    for task in pending:
        task.cancel()
    
    return done.pop().result()
```

---

## Strategies

### 1. Delayed Hedging (Most Common)
```
Wait short time before hedging

t=0:   Send to Server A
t=5ms: No response? Send to Server B
t=8ms: Server B responds → Use it

Good balance of latency reduction vs overhead
```

### 2. Immediate Hedging
```
Send to multiple servers simultaneously

t=0: Send to Server A AND Server B
t=5ms: Server B responds → Use it

Maximum latency reduction, but 2x load
```

### 3. Tied Requests
```
Send to both, but include "tied" token

Server receiving first cancels the other
Reduces wasted work

Used by Google's Bigtable
```

---

## When to Hedge

### Good Hedge Delay Calculation
```python
def calculate_hedge_delay(latencies: list) -> float:
    """Use p95 latency as hedge trigger."""
    sorted_latencies = sorted(latencies)
    p95_index = int(len(sorted_latencies) * 0.95)
    return sorted_latencies[p95_index]

# If p95 is 20ms, hedge after 20ms
# 95% of requests won't trigger hedge
# Only slowest 5% get hedged
```

### Adaptive Hedging
```python
class AdaptiveHedger:
    def __init__(self):
        self.latency_window = []
    
    def record_latency(self, latency_ms: float):
        self.latency_window.append(latency_ms)
        if len(self.latency_window) > 1000:
            self.latency_window.pop(0)
    
    def get_hedge_delay(self) -> float:
        if not self.latency_window:
            return 10  # Default
        return np.percentile(self.latency_window, 95)
```

---

## Trade-offs

### Costs
```
1. Increased load
   - Extra requests to servers
   - ~5% overhead if hedging at p95

2. Resource consumption
   - Cancelled requests still consume some resources
   - Network bandwidth

3. Complexity
   - Request cancellation
   - Idempotency requirements
```

### Benefits
```
1. Dramatically reduced tail latency
   - Google: 50% reduction in p99

2. Better user experience
   - More consistent response times

3. Fault tolerance
   - Masks transient failures
```

---

## Real-World Usage

| Company | Implementation |
|---------|----------------|
| **Google** | Tied requests in Bigtable, Spanner |
| **Amazon** | Hedged reads in DynamoDB |
| **Netflix** | Hedged requests for metadata |
| **LinkedIn** | Hedged calls to distributed cache |

---

## Best Practices

```
1. Only hedge read requests (idempotent operations)
2. Set hedge delay at p95-p99 latency
3. Limit hedge attempts (2-3 max)
4. Monitor hedge rate and adjust
5. Implement request cancellation
6. Consider server-side tied requests
```

---

## Related Techniques

| Technique | Description |
|-----------|-------------|
| **Backup requests** | Similar, but wait longer |
| **Speculative execution** | Pre-execute likely paths |
| **Request deflection** | Route away from slow servers |
| **Circuit breaker** | Stop calling failing servers |

---

## Interview Tips

When discussing Hedged Requests:
1. Explain tail latency problem (p99 issue)
2. Describe the hedge delay strategy
3. Discuss the load trade-off
4. Mention idempotency requirement
5. Give real examples (Google, Amazon)

