# Distributed Locks

## What They Are

**Distributed Locks** provide mutual exclusion across multiple processes or machines. They ensure only one client can hold a lock at a time, preventing concurrent access to shared resources.

---

## The Analogy 🔐

Think of a **bathroom key at a coffee shop**:
- Only one key exists
- Customer takes key → bathroom locked
- Others must wait
- Customer returns key → next person can go

In distributed systems:
- Lock = the key
- Processes = customers
- Shared resource = bathroom

---

## Why They Exist

### The Problem: Concurrent Access
```
Without locks:
Process A: Read balance (100)
Process B: Read balance (100)
Process A: Deduct 30, Write (70)
Process B: Deduct 50, Write (50)  ← Lost update!

Expected: 100 - 30 - 50 = 20
Actual: 50 (Process A's write lost)
```

### What Distributed Locks Solve:
- **Mutual exclusion** - Only one holder at a time
- **Coordination** - Serialize access to resources
- **Consistency** - Prevent race conditions
- **Leader election** - Who's the primary?

---

## Implementation Approaches

### 1. Database-Based Lock
```sql
-- Acquire lock
INSERT INTO locks (resource_id, owner, expires_at)
VALUES ('order-123', 'process-A', NOW() + INTERVAL '30 seconds')
ON CONFLICT DO NOTHING;

-- Check if acquired
SELECT * FROM locks WHERE resource_id = 'order-123' AND owner = 'process-A';

-- Release lock
DELETE FROM locks WHERE resource_id = 'order-123' AND owner = 'process-A';
```

### 2. Redis Single-Node Lock
```python
# Acquire
SET resource_name my_random_value NX PX 30000
# NX = only if not exists
# PX = expire in 30000ms

# Release (Lua script for atomicity)
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### 3. Redlock (Redis Distributed)
```
┌─────────────────────────────────────────────────────────────────┐
│                        Redlock Algorithm                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   5 Independent Redis Masters                                    │
│   ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐                            │
│   │ 1 │  │ 2 │  │ 3 │  │ 4 │  │ 5 │                            │
│   └───┘  └───┘  └───┘  └───┘  └───┘                            │
│     ✓      ✓      ✗      ✓      ✗                               │
│                                                                  │
│   Steps:                                                         │
│   1. Get current time                                            │
│   2. Try to acquire lock on all N nodes                         │
│   3. Lock acquired if:                                           │
│      - Majority (N/2 + 1) succeeded                             │
│      - Total time < lock validity                                │
│   4. Effective TTL = initial TTL - elapsed time                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Challenges

### 1. Lock Expiration (Fencing Problem)
```
┌─────────────────────────────────────────────────────────────────┐
│                    The Fencing Problem                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Client A acquires lock                                         │
│   Client A pauses (GC, network)                                  │
│   Lock expires                                                   │
│   Client B acquires lock                                         │
│   Client A resumes, thinks it has lock!                         │
│   Both clients write → DATA CORRUPTION                          │
│                                                                  │
│   Solution: Fencing Tokens                                       │
│   - Lock includes monotonic token (1, 2, 3...)                  │
│   - Resource rejects writes with old tokens                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Clock Drift
- Redlock assumes synchronized clocks
- Clock drift can cause safety violations
- Use NTP, but be aware of limitations

### 3. Network Partitions
- Client may not know lock expired
- Split-brain scenarios possible

---

## When to Use Distributed Locks

### ✅ Good Fit:
- **Efficiency** - Prevent duplicate work
- **Correctness** - When safety is critical
- **Leader election** - Single active worker
- **Rate limiting** - Per-resource limits

### ❌ Avoid When:
- Can use optimistic concurrency instead
- Idempotent operations possible
- Single-node system
- Performance critical (locks add latency)

---

## Comparison of Approaches

| Approach | Consistency | Availability | Complexity |
|----------|-------------|--------------|------------|
| **Single Redis** | Weak | High | Low |
| **Redlock** | Better | Medium | Medium |
| **ZooKeeper** | Strong | Medium | High |
| **etcd** | Strong | Medium | Medium |
| **Database** | Strong | Depends | Low |

---

## Best Practices

### 1. Always Set TTL
```python
# Bad: Lock forever if process dies
lock.acquire()

# Good: Auto-expire
lock.acquire(ttl=30)
```

### 2. Use Unique Owner ID
```python
# Include random value to prevent accidental release
owner_id = f"{hostname}-{pid}-{uuid4()}"
```

### 3. Extend Lock If Needed
```python
while doing_work:
    lock.extend(ttl=30)  # Heartbeat
    do_chunk_of_work()
```

### 4. Use Fencing Tokens
```python
token = lock.acquire()
storage.write(data, fencing_token=token)
```

---

## Real-World Tools

| Tool | Type | Use Case |
|------|------|----------|
| **Redis** | Single/Redlock | Fast, simple locks |
| **ZooKeeper** | Consensus-based | Strong consistency |
| **etcd** | Consensus-based | Kubernetes, strong consistency |
| **Consul** | Consensus-based | Service mesh |
| **DynamoDB** | Conditional writes | AWS native |

---

## Interview Tips

When discussing Distributed Locks:
1. Explain the fencing problem
2. Compare single-node vs consensus-based
3. Discuss TTL and auto-expiration
4. Mention Redlock controversy (Martin Kleppmann's critique)
5. Know when NOT to use locks (idempotency, optimistic locking)

