# Rate Limiting Algorithms

## What They Are

**Rate Limiting** controls how many requests a client can make in a given time window. Multiple algorithms exist, each with different trade-offs for precision, memory, and burst handling.

---

## The Analogy 🚿

Think of **water flow control**:
- **Leaky Bucket**: Water drips out at constant rate, overflow spills
- **Token Bucket**: Tokens refill over time, spend tokens for requests
- **Fixed Window**: Count drops per minute, reset at minute boundary
- **Sliding Window**: Rolling average over last 60 seconds

---

## Why Rate Limiting Exists

### Problems Without Rate Limiting:
- DoS attacks overload servers
- Noisy neighbors hog resources
- Cascading failures from traffic spikes
- Unfair resource allocation
- API abuse and scraping

### What It Solves:
- **System protection** - Prevent overload
- **Fair usage** - Share resources equitably
- **Cost control** - Limit API consumption
- **Security** - Block brute-force attacks

---

## The Algorithms

### 1. Token Bucket ⭐ (Most Popular)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Token Bucket                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Tokens refill at rate R             ┌─────────────┐           │
│           │                           │  ● ● ● ●    │ Bucket    │
│           ▼                           │  ● ● ●      │ (max B)   │
│       ┌───────┐                       └──────┬──────┘           │
│       │ + + + │ ──────────────────────────▶  │                  │
│       └───────┘                              │                  │
│                                              ▼                  │
│                                    Request consumes 1 token     │
│                                    No token? Request denied     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Allows bursts up to bucket size, then rate-limited to R.
```

**Properties:**
- ✅ Allows controlled bursts
- ✅ Smooth rate enforcement
- ✅ Simple to implement
- Parameters: Rate (tokens/sec), Bucket size

**Implementation:**
```python
class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate = rate        # tokens per second
        self.capacity = capacity
        self.tokens = capacity
        self.last_update = time.time()
    
    def allow_request(self):
        now = time.time()
        # Refill tokens
        self.tokens = min(
            self.capacity,
            self.tokens + (now - self.last_update) * self.rate
        )
        self.last_update = now
        
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

---

### 2. Leaky Bucket

```
┌─────────────────────────────────────────────────────────────────┐
│                        Leaky Bucket                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Requests arrive (any rate)    ┌─────────────┐                 │
│           │                     │  ▼ ▼ ▼ ▼    │ Queue           │
│           ▼                     │  ▼ ▼ ▼      │ (size Q)        │
│       ┌───────┐                 └──────┬──────┘                 │
│       │ > > > │ ──────────────────────▶│                        │
│       └───────┘                        │ Leaks at rate R        │
│                                        ▼                        │
│                               ┌────────────────┐                │
│                               │   Processing   │                │
│                               └────────────────┘                │
│                                                                  │
│   Full queue? New requests dropped/rejected                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Requests processed at constant rate R, regardless of arrival rate.
```

**Properties:**
- ✅ Perfectly smooth output rate
- ✅ Simple FIFO queue
- ❌ No bursting allowed
- ❌ May increase latency (queuing)

**Use when**: Need constant processing rate (e.g., network traffic shaping)

---

### 3. Fixed Window Counter

```
┌─────────────────────────────────────────────────────────────────┐
│                    Fixed Window Counter                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Window 1         Window 2         Window 3                     │
│   (0:00-0:59)      (1:00-1:59)      (2:00-2:59)                  │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐                 │
│   │ cnt: 97 │      │ cnt: 23 │      │ cnt: 0  │                 │
│   │ max:100 │      │ max:100 │      │ max:100 │                 │
│   └─────────┘      └─────────┘      └─────────┘                 │
│                                                                  │
│   Problem: At 0:59, 100 requests. At 1:00, 100 more.            │
│            = 200 requests in 1-minute span (edge burst)         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Properties:**
- ✅ Very simple (single counter)
- ✅ Low memory
- ❌ Burst at window boundaries (2x limit!)

---

### 4. Sliding Window Log

```
┌─────────────────────────────────────────────────────────────────┐
│                    Sliding Window Log                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Store timestamp of each request:                               │
│   [1:00:05, 1:00:12, 1:00:23, 1:00:45, 1:00:58, 1:01:02...]     │
│                                                                  │
│   Current time: 1:01:30                                          │
│   Window: last 60 seconds (1:00:30 - 1:01:30)                   │
│                                                                  │
│   Count requests in window: 2                                    │
│   Limit: 100 → Request allowed                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Properties:**
- ✅ Most accurate
- ❌ High memory (store all timestamps)
- Use for: Low-volume, high-precision needs

---

### 5. Sliding Window Counter (Hybrid) ⭐

```
Combines fixed window efficiency with sliding window accuracy.

Current window: 30% elapsed
Previous window count: 100
Current window count: 20

Weighted count = 100 * 0.70 + 20 * 1.0 = 90
```

**Properties:**
- ✅ Memory efficient (2 counters)
- ✅ Smooth limits (no boundary burst)
- ✅ Good approximation

---

## Comparison Table

| Algorithm | Memory | Precision | Bursting | Use Case |
|-----------|--------|-----------|----------|----------|
| Token Bucket | Low | Good | Controlled | APIs, general |
| Leaky Bucket | Low | Perfect | None | Traffic shaping |
| Fixed Window | Very Low | Poor | 2x at edges | Simple counters |
| Sliding Log | High | Perfect | None | High-precision |
| Sliding Window | Low | Good | Limited | Best balance |

---

## When to Use Each

| Scenario | Recommended Algorithm |
|----------|----------------------|
| **API rate limiting** | Token Bucket or Sliding Window |
| **Network traffic shaping** | Leaky Bucket |
| **Simple request counting** | Fixed Window |
| **Financial transactions** | Sliding Window Log |
| **Distributed systems** | Token Bucket + Redis |

---

## Interview Tips

When discussing Rate Limiting:
1. Know at least Token Bucket and Sliding Window
2. Explain trade-offs (memory, precision, bursting)
3. Discuss distributed implementation (Redis, atomic operations)
4. Mention HTTP 429 response code

