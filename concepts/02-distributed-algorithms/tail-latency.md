# Tail Latency and Hedged Requests

## What It Is

**Tail Latency** refers to the high-percentile response times (p99, p999) that affect a small but significant portion of requests. **Hedged Requests** is a technique to reduce tail latency by sending duplicate requests.

---

## The Analogy 🏃

Think of **ordering food delivery**:
- Average delivery: 30 minutes
- But sometimes: 2 hours (driver got lost, restaurant busy)
- **Hedged request**: Order from two restaurants, cancel the slower one

---

## Why It Matters

### The Problem: Tail Latency Amplification
```
Single service:
  p50: 10ms, p99: 100ms

Request touching 100 services:
  P(all fast) = 0.99^100 = 37%
  P(at least one slow) = 63%

Most requests hit at least one slow service!
```

### Real Impact:
```
1 million requests/day
1% at p99 = 10,000 slow requests
0.1% at p999 = 1,000 very slow requests

These affect real users!
```

---

## Causes of Tail Latency

```
┌─────────────────────────────────────────────────────────────────┐
│                 Tail Latency Causes                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Garbage Collection                                         │
│      - JVM GC pauses: 10ms - 1s                                 │
│                                                                  │
│   2. Network Congestion                                         │
│      - Packet loss, retransmits                                 │
│                                                                  │
│   3. Resource Contention                                        │
│      - CPU scheduling, lock contention                          │
│                                                                  │
│   4. Background Tasks                                           │
│      - Log rotation, backups, compaction                        │
│                                                                  │
│   5. Cold Caches                                                │
│      - Cache miss after restart                                 │
│                                                                  │
│   6. Queueing                                                   │
│      - Request waiting in queue                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Hedged Requests

### Basic Hedging
```python
async def hedged_request(request, timeout_ms=10):
    """Send request, if slow, send duplicate"""
    
    # Start first request
    task1 = asyncio.create_task(send_request(request, server1))
    
    try:
        # Wait for quick response
        result = await asyncio.wait_for(task1, timeout=timeout_ms/1000)
        return result
    except asyncio.TimeoutError:
        # First request slow, hedge with second
        task2 = asyncio.create_task(send_request(request, server2))
        
        # Return whichever finishes first
        done, pending = await asyncio.wait(
            [task1, task2],
            return_when=asyncio.FIRST_COMPLETED
        )
        
        # Cancel the slower one
        for task in pending:
            task.cancel()
        
        return done.pop().result()
```

### Tied Requests (Google's Approach)
```python
async def tied_request(request, servers):
    """Send to multiple servers, they coordinate"""
    
    # Send to two servers with same request_id
    request_id = generate_id()
    
    tasks = [
        send_request(request, server, request_id)
        for server in servers[:2]
    ]
    
    # Servers check if another already processing
    # If so, they skip (reduces wasted work)
    
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )
    
    for task in pending:
        task.cancel()
    
    return done.pop().result()
```

---

## Other Tail Latency Techniques

### 1. Backup Requests with Delay
```python
async def backup_request(request, delay_ms=50):
    """Wait, then send backup if no response"""
    
    primary = asyncio.create_task(send_request(request, primary_server))
    
    await asyncio.sleep(delay_ms / 1000)
    
    if not primary.done():
        backup = asyncio.create_task(send_request(request, backup_server))
        done, _ = await asyncio.wait(
            [primary, backup],
            return_when=asyncio.FIRST_COMPLETED
        )
        return done.pop().result()
    
    return await primary
```

### 2. Speculative Execution
```
Send request to multiple replicas simultaneously
Use first response, ignore others

✅ Lowest latency
❌ Highest resource usage
```

### 3. Request Deadlines
```python
def process_request(request):
    deadline = request.deadline
    
    if time.now() > deadline:
        # Don't bother processing, already too late
        return None
    
    # Process with remaining time budget
    remaining = deadline - time.now()
    return process_with_timeout(request, remaining)
```

---

## Measuring Tail Latency

```
Key Metrics:
- p50 (median): 50% of requests faster
- p90: 90% of requests faster
- p99: 99% of requests faster
- p999: 99.9% of requests faster

Example:
  p50: 10ms
  p90: 25ms
  p99: 100ms
  p999: 500ms

The jump from p99 to p999 often reveals issues!
```

---

## Trade-offs

| Technique | Latency Reduction | Resource Cost |
|-----------|-------------------|---------------|
| **No hedging** | None | 1x |
| **Delayed backup** | Moderate | 1.1-1.5x |
| **Hedged requests** | Good | 1.5-2x |
| **Speculative** | Best | 2-3x |

---

## Best Practices

```
1. Measure tail latency, not just average
2. Set appropriate percentile SLOs (p99, p999)
3. Use hedging for read-only, idempotent requests
4. Tune hedge delay based on latency distribution
5. Monitor hedge rate (should be low, ~1-5%)
6. Implement request deadlines
7. Use tied requests to reduce wasted work
```

---

## Real-World Examples

| Company | Technique |
|---------|-----------|
| **Google** | Tied requests, backup requests |
| **Amazon** | Hedged requests for DynamoDB |
| **Netflix** | Speculative retries |
| **Facebook** | Adaptive hedging |

---

## Interview Tips

When discussing Tail Latency:
1. Explain the amplification problem
2. Know hedged vs backup vs speculative
3. Discuss trade-offs (latency vs resources)
4. Mention idempotency requirement
5. Give percentile examples (p99, p999)

