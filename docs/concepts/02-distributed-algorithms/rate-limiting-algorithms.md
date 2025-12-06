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

## Implementation Deep Dive

### Distributed Token Bucket (Redis)

```python
import redis
import time

class DistributedTokenBucket:
    """Thread-safe, distributed rate limiter using Redis"""

    def __init__(self, redis_client, key_prefix="ratelimit"):
        self.redis = redis_client
        self.key_prefix = key_prefix

    def allow_request(self, user_id: str, rate: float, capacity: int) -> bool:
        """
        Atomic token bucket using Redis Lua script.
        rate: tokens per second
        capacity: max burst size
        """
        key = f"{self.key_prefix}:{user_id}"
        now = time.time()

        # Lua script for atomic operation
        lua_script = """
        local key = KEYS[1]
        local rate = tonumber(ARGV[1])
        local capacity = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        local requested = tonumber(ARGV[4])

        local data = redis.call('HMGET', key, 'tokens', 'last_update')
        local tokens = tonumber(data[1]) or capacity
        local last_update = tonumber(data[2]) or now

        -- Refill tokens based on time elapsed
        local elapsed = now - last_update
        tokens = math.min(capacity, tokens + (elapsed * rate))

        local allowed = 0
        if tokens >= requested then
            tokens = tokens - requested
            allowed = 1
        end

        redis.call('HMSET', key, 'tokens', tokens, 'last_update', now)
        redis.call('EXPIRE', key, 3600)  -- Clean up after 1 hour

        return allowed
        """

        result = self.redis.eval(lua_script, 1, key, rate, capacity, now, 1)
        return result == 1

# Usage
redis_client = redis.Redis(host='localhost', port=6379)
limiter = DistributedTokenBucket(redis_client)

# Allow 100 requests/minute with burst of 10
if limiter.allow_request("user_123", rate=100/60, capacity=10):
    process_request()
else:
    return HttpResponse(status=429, headers={"Retry-After": "60"})
```

### Sliding Window Counter (Production Implementation)

```python
class SlidingWindowCounter:
    """Memory-efficient sliding window using two counters"""

    def __init__(self, redis_client, window_size_sec=60):
        self.redis = redis_client
        self.window_size = window_size_sec

    def allow_request(self, user_id: str, limit: int) -> tuple[bool, dict]:
        now = time.time()
        current_window = int(now // self.window_size)
        prev_window = current_window - 1
        window_progress = (now % self.window_size) / self.window_size

        curr_key = f"sw:{user_id}:{current_window}"
        prev_key = f"sw:{user_id}:{prev_window}"

        # Get both window counts atomically
        pipe = self.redis.pipeline()
        pipe.get(prev_key)
        pipe.incr(curr_key)
        pipe.expire(curr_key, self.window_size * 2)
        prev_count, curr_count, _ = pipe.execute()

        prev_count = int(prev_count or 0)

        # Weighted count: more weight on current as window progresses
        weighted_count = prev_count * (1 - window_progress) + curr_count

        allowed = weighted_count <= limit

        return allowed, {
            "limit": limit,
            "remaining": max(0, int(limit - weighted_count)),
            "reset": int((current_window + 1) * self.window_size)
        }
```

### Rate Limit Response Headers

```python
# Always include these headers in API responses
def add_rate_limit_headers(response, info):
    response.headers["X-RateLimit-Limit"] = str(info["limit"])
    response.headers["X-RateLimit-Remaining"] = str(info["remaining"])
    response.headers["X-RateLimit-Reset"] = str(info["reset"])
    if not info.get("allowed", True):
        response.headers["Retry-After"] = str(info["reset"] - time.time())
```

### Multi-Tier Rate Limiting

```python
class TieredRateLimiter:
    """Different limits for different time windows"""

    def __init__(self, redis_client):
        self.limiters = [
            SlidingWindowCounter(redis_client, window_size_sec=1),    # Per-second
            SlidingWindowCounter(redis_client, window_size_sec=60),   # Per-minute
            SlidingWindowCounter(redis_client, window_size_sec=3600), # Per-hour
        ]
        self.limits = [10, 100, 1000]  # Requests per tier

    def allow_request(self, user_id: str) -> bool:
        for limiter, limit in zip(self.limiters, self.limits):
            allowed, _ = limiter.allow_request(user_id, limit)
            if not allowed:
                return False
        return True
```

---

## Interview Tips

When discussing Rate Limiting:
1. Know at least Token Bucket and Sliding Window
2. Explain trade-offs (memory, precision, bursting)
3. Discuss distributed implementation (Redis + Lua for atomicity)
4. Mention HTTP 429 response code and rate limit headers
5. Explain multi-tier limiting (per-second + per-minute + per-hour)

