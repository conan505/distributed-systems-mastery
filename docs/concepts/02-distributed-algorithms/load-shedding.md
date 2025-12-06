# Load Shedding

## What It Is

**Load Shedding** is the practice of intentionally dropping requests when a system is overloaded to maintain service quality for the requests that are processed.

---

## The Analogy 🚢

Think of a **sinking ship**:
- Ship is overloaded and taking on water
- Captain throws cargo overboard (sheds load)
- Ship stays afloat, crew survives
- Better to lose some cargo than everything

---

## Why It Exists

### The Problem: Cascading Failure
```
Without load shedding:

Load increases → Response time increases
→ Timeouts → Retries → More load
→ Memory exhaustion → OOM crash
→ All requests fail

With load shedding:

Load increases → Shed excess requests
→ Remaining requests succeed
→ System stays healthy
→ Partial service > no service
```

### The Math:
```
Server capacity: 1000 req/s
Incoming load: 2000 req/s

Without shedding:
  All 2000 requests queue up
  Latency spikes, timeouts
  Eventually: 0 successful

With shedding:
  Accept 1000, reject 1000
  1000 succeed with normal latency
  1000 get fast failure (retry elsewhere)
```

---

## Load Shedding Strategies

### 1. Queue-Based Shedding
```python
class QueueBasedShedder:
    def __init__(self, max_queue_size=100):
        self.queue = Queue(maxsize=max_queue_size)
    
    def accept_request(self, request):
        try:
            self.queue.put_nowait(request)
            return True
        except queue.Full:
            # Queue full, shed this request
            return False
```

### 2. Rate-Based Shedding
```python
class RateBasedShedder:
    def __init__(self, max_rps=1000):
        self.max_rps = max_rps
        self.current_count = 0
        self.window_start = time.time()
    
    def accept_request(self, request):
        now = time.time()
        
        # Reset window every second
        if now - self.window_start >= 1.0:
            self.current_count = 0
            self.window_start = now
        
        if self.current_count >= self.max_rps:
            return False  # Shed
        
        self.current_count += 1
        return True
```

### 3. Latency-Based Shedding
```python
class LatencyBasedShedder:
    def __init__(self, target_latency_ms=100):
        self.target_latency = target_latency_ms
        self.current_latency = 0
    
    def accept_request(self, request):
        if self.current_latency > self.target_latency * 2:
            # Latency too high, shed aggressively
            return random.random() > 0.5
        elif self.current_latency > self.target_latency:
            # Latency elevated, shed some
            return random.random() > 0.2
        return True
    
    def record_latency(self, latency_ms):
        # Exponential moving average
        self.current_latency = 0.9 * self.current_latency + 0.1 * latency_ms
```

### 4. Priority-Based Shedding
```python
class PriorityBasedShedder:
    def __init__(self, capacity=1000):
        self.capacity = capacity
        self.current_load = 0
    
    def accept_request(self, request):
        if self.current_load < self.capacity:
            return True
        
        # Over capacity - shed based on priority
        if request.priority == "critical":
            return True  # Always accept critical
        elif request.priority == "high":
            return self.current_load < self.capacity * 1.2
        elif request.priority == "normal":
            return self.current_load < self.capacity * 1.1
        else:  # low priority
            return False  # Shed first
```

---

## LIFO vs FIFO Queuing

```
┌─────────────────────────────────────────────────────────────────┐
│                   LIFO vs FIFO Under Load                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   FIFO (First In, First Out):                                   │
│   [Old] [Old] [Old] [New] [New]                                 │
│     ↑                                                            │
│   Process old requests first                                    │
│   Problem: Old requests may have already timed out!             │
│                                                                  │
│   LIFO (Last In, First Out):                                    │
│   [Old] [Old] [Old] [New] [New]                                 │
│                           ↑                                      │
│   Process new requests first                                    │
│   Benefit: New requests more likely to succeed                  │
│   Old requests shed (client already gave up)                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## CoDel (Controlled Delay)

```python
class CoDel:
    """Controlled Delay - smart queue management"""
    
    def __init__(self, target_delay_ms=5, interval_ms=100):
        self.target = target_delay_ms
        self.interval = interval_ms
        self.first_above_time = 0
        self.drop_next = 0
        self.count = 0
    
    def should_drop(self, packet_delay_ms):
        now = time.time() * 1000
        
        if packet_delay_ms < self.target:
            self.first_above_time = 0
            return False
        
        if self.first_above_time == 0:
            self.first_above_time = now + self.interval
            return False
        
        if now >= self.first_above_time:
            # Been above target for interval, start dropping
            return True
        
        return False
```

---

## Graceful Degradation

```
Instead of complete rejection, offer degraded service:

Full service:
  - Real-time recommendations
  - Full search results
  - High-res images

Degraded service:
  - Cached recommendations
  - Limited search results
  - Thumbnail images

Better than nothing!
```

---

## Best Practices

```
1. Shed early, shed fast
   - Reject at edge, not deep in stack
   
2. Return proper status codes
   - 503 Service Unavailable
   - 429 Too Many Requests
   - Include Retry-After header

3. Prioritize requests
   - Critical > High > Normal > Low
   - Paid users > Free users

4. Monitor shed rate
   - Alert if shedding too much
   - Indicates capacity issue

5. Test load shedding
   - Chaos engineering
   - Load testing
```

---

## Load Shedding vs Rate Limiting

| Aspect | Load Shedding | Rate Limiting |
|--------|---------------|---------------|
| **Trigger** | System overload | Per-client limits |
| **Goal** | Protect system | Fair usage |
| **Scope** | Global | Per client/API |
| **Response** | 503 | 429 |

---

## Interview Tips

When discussing Load Shedding:
1. Explain why partial service beats no service
2. Know different shedding strategies
3. Discuss priority-based shedding
4. Mention LIFO queuing benefit
5. Distinguish from rate limiting

