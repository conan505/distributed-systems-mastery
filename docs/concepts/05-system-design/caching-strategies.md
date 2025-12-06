# Caching Strategies

## What It Is

**Caching** stores frequently accessed data in a faster storage layer to reduce latency, database load, and computational costs.

---

## The Analogy 📚

Think of a **library's reading room**:
- Popular books kept on the desk (cache)
- Rare books in the stacks (database)
- Most readers get books instantly from desk
- Occasionally need to fetch from stacks

---

## Why Caching Matters

### The Problem: Slow Data Access
```
Without cache:
- Every request → Database query
- Database: 10ms latency, 1000 QPS limit
- 10,000 requests = bottleneck

With cache:
- 90% cache hits → 1ms latency
- 10% cache misses → 10ms latency
- Average: 1.9ms, database load: 1000 QPS
```

---

## Caching Patterns

### 1. Cache-Aside (Lazy Loading)
```
┌─────────────────────────────────────────────────────────────────┐
│                      Cache-Aside Pattern                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Read:                                                          │
│   1. Check cache                                                 │
│   2. If miss: Read from DB, populate cache                       │
│   3. Return data                                                 │
│                                                                  │
│   Write:                                                         │
│   1. Write to DB                                                 │
│   2. Invalidate cache (or let it expire)                        │
│                                                                  │
│   ✅ Only caches what's needed                                   │
│   ✅ Cache failure = graceful degradation                        │
│   ❌ Cache miss penalty (cold start)                             │
│   ❌ Stale data possible                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
def get_user(user_id):
    # 1. Check cache
    user = cache.get(f"user:{user_id}")
    if user:
        return user
    
    # 2. Cache miss - fetch from DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. Populate cache
    cache.set(f"user:{user_id}", user, ttl=300)
    
    return user
```

### 2. Write-Through
```
┌─────────────────────────────────────────────────────────────────┐
│                    Write-Through Pattern                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Write:                                                         │
│   1. Write to cache AND database (synchronously)                │
│                                                                  │
│   Read:                                                          │
│   1. Always read from cache                                      │
│                                                                  │
│   ✅ Cache always consistent                                     │
│   ✅ Read-heavy workloads benefit                                │
│   ❌ Write latency (both writes)                                 │
│   ❌ Cache filled with rarely-read data                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Write-Behind (Write-Back)
```
┌─────────────────────────────────────────────────────────────────┐
│                    Write-Behind Pattern                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Write:                                                         │
│   1. Write to cache (return immediately)                        │
│   2. Async: Batch write to DB                                   │
│                                                                  │
│   ✅ Fast writes                                                 │
│   ✅ Batching reduces DB load                                    │
│   ❌ Data loss if cache fails before DB write                   │
│   ❌ Complexity in failure handling                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Read-Through
```
Cache handles data loading automatically:
- Application asks cache for data
- Cache fetches from DB on miss
- Cache returns data

Similar to cache-aside but cache-managed
```

---

## Cache Eviction Policies

| Policy | Description | Use Case |
|--------|-------------|----------|
| **LRU** | Least Recently Used | General purpose |
| **LFU** | Least Frequently Used | Popularity-based |
| **FIFO** | First In, First Out | Simple, predictable |
| **TTL** | Time To Live | Time-sensitive data |
| **Random** | Random eviction | Simple, low overhead |

---

## Cache Invalidation

### Strategies:
```
1. TTL-based: Data expires after time period
   cache.set(key, value, ttl=3600)  # 1 hour

2. Event-driven: Invalidate on updates
   def update_user(user_id, data):
       db.update(user_id, data)
       cache.delete(f"user:{user_id}")

3. Version-based: Change key on update
   key = f"user:{user_id}:v{version}"
```

### The Hard Problem:
```
"There are only two hard things in Computer Science: 
cache invalidation and naming things."

Why it's hard:
- Distributed systems: Multiple caches to invalidate
- Timing: Race between read and invalidation
- Consistency: Stale reads during propagation
```

---

## Caching Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                      Caching Layers                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Browser Cache ─────▶ Fastest, per-user                        │
│         │                                                        │
│         ▼                                                        │
│   CDN Cache ─────────▶ Edge, static content                     │
│         │                                                        │
│         ▼                                                        │
│   Load Balancer ─────▶ SSL termination, routing                 │
│         │                                                        │
│         ▼                                                        │
│   Application Cache ──▶ Local memory (per-instance)             │
│         │                                                        │
│         ▼                                                        │
│   Distributed Cache ──▶ Redis/Memcached (shared)                │
│         │                                                        │
│         ▼                                                        │
│   Database Cache ─────▶ Query cache, buffer pool                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Tools

| Tool | Type | Use Case |
|------|------|----------|
| **Redis** | In-memory, distributed | Sessions, caching, pub/sub |
| **Memcached** | In-memory, distributed | Simple key-value cache |
| **Caffeine** | Local (Java) | Application-level cache |
| **Varnish** | HTTP accelerator | Web page caching |
| **CloudFront** | CDN | Static asset caching |

---

## Cache Stampede Problem

```
Problem:
- Key expires
- 1000 requests hit simultaneously
- All miss cache, all query DB
- DB overwhelmed!

Solutions:
1. Lock: Only one request fetches, others wait
2. Stale-while-revalidate: Return stale, async refresh
3. Probabilistic early expiration: Random early refresh
```

---

## Interview Tips

When discussing Caching:
1. Know cache-aside vs write-through patterns
2. Explain cache invalidation challenges
3. Discuss eviction policies (LRU, TTL)
4. Mention cache stampede and solutions
5. Give real examples (Redis, CDN)

