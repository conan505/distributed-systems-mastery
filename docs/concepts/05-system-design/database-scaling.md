# Database Scaling

## What It Is

**Database Scaling** refers to techniques for handling increased load on databases, including vertical scaling (bigger machines), horizontal scaling (more machines), and various optimization strategies.

---

## The Analogy 🏢

Think of a **growing company**:
- **Vertical scaling**: Move to bigger office
- **Horizontal scaling**: Open branch offices
- **Read replicas**: Hire assistants to answer questions
- **Sharding**: Divide work by region

---

## Why It Matters

### The Problem: Database Bottleneck
```
Single database limits:
- CPU: ~100K queries/sec
- Storage: ~100TB practical limit
- Connections: ~10K concurrent
- Write throughput: Single master bottleneck

When you hit limits:
- Queries slow down
- Timeouts increase
- Application fails
```

---

## Scaling Strategies

### 1. Vertical Scaling (Scale Up)
```
┌─────────────────────────────────────────────────────────────────┐
│                    Vertical Scaling                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Before:           After:                                       │
│   ┌─────────┐       ┌─────────────┐                             │
│   │ 4 CPU   │  ──▶  │ 64 CPU      │                             │
│   │ 16GB RAM│       │ 512GB RAM   │                             │
│   │ 1TB SSD │       │ 10TB NVMe   │                             │
│   └─────────┘       └─────────────┘                             │
│                                                                  │
│   ✅ Simple - no code changes                                    │
│   ✅ No distributed complexity                                   │
│   ❌ Hardware limits (can't scale forever)                       │
│   ❌ Expensive (exponential cost)                                │
│   ❌ Single point of failure                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Read Replicas
```
┌─────────────────────────────────────────────────────────────────┐
│                     Read Replicas                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Writes ──▶ [Primary] ──replication──▶ [Replica 1]             │
│                  │                      [Replica 2]             │
│                  │                      [Replica 3]             │
│                  │                           │                   │
│                  └───────────────────────────┘                   │
│                              │                                   │
│                         Reads ◀──                               │
│                                                                  │
│   ✅ Scale reads horizontally                                    │
│   ✅ Geographic distribution                                     │
│   ❌ Replication lag (eventual consistency)                      │
│   ❌ Writes still limited to primary                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Sharding (Horizontal Partitioning)
```
┌─────────────────────────────────────────────────────────────────┐
│                       Sharding                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Users table split by user_id:                                 │
│                                                                  │
│   [Shard 1]        [Shard 2]        [Shard 3]                   │
│   user_id 1-1M     user_id 1M-2M    user_id 2M-3M              │
│                                                                  │
│   Sharding strategies:                                           │
│   - Range: user_id 1-1M, 1M-2M, ...                             │
│   - Hash: hash(user_id) % num_shards                            │
│   - Directory: lookup table for shard                           │
│                                                                  │
│   ✅ Scale writes horizontally                                   │
│   ✅ No single point of failure                                  │
│   ❌ Complex queries (cross-shard joins)                         │
│   ❌ Rebalancing is hard                                         │
│   ❌ Application complexity                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Optimization Before Scaling

### 1. Indexing
```sql
-- Before: Full table scan
SELECT * FROM orders WHERE customer_id = 123;
-- 10 seconds on 100M rows

-- After: Index lookup
CREATE INDEX idx_customer ON orders(customer_id);
-- 10 milliseconds
```

### 2. Query Optimization
```sql
-- Bad: SELECT *
SELECT * FROM users WHERE id = 1;

-- Good: Select only needed columns
SELECT name, email FROM users WHERE id = 1;

-- Bad: N+1 queries
for order in orders:
    customer = query("SELECT * FROM customers WHERE id = ?", order.customer_id)

-- Good: JOIN or batch
SELECT o.*, c.name FROM orders o JOIN customers c ON o.customer_id = c.id
```

### 3. Connection Pooling
```python
# Without pooling: New connection per request
conn = db.connect()  # 50ms overhead each time

# With pooling: Reuse connections
pool = ConnectionPool(min=10, max=100)
conn = pool.get()  # <1ms
```

### 4. Caching
```python
def get_user(user_id):
    # Check cache first
    user = cache.get(f"user:{user_id}")
    if user:
        return user
    
    # Cache miss - query DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    cache.set(f"user:{user_id}", user, ttl=300)
    return user
```

---

## Sharding Strategies

| Strategy | Pros | Cons |
|----------|------|------|
| **Range** | Simple, range queries work | Hot spots possible |
| **Hash** | Even distribution | Range queries broken |
| **Directory** | Flexible | Lookup overhead |
| **Geographic** | Low latency | Uneven distribution |

---

## CAP Theorem Trade-offs

```
┌─────────────────────────────────────────────────────────────────┐
│                     CAP Theorem                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Pick 2 of 3:                                                  │
│                                                                  │
│   Consistency ─────────── Availability                          │
│        \                    /                                    │
│         \                  /                                     │
│          \                /                                      │
│           \              /                                       │
│            \            /                                        │
│         Partition Tolerance                                      │
│                                                                  │
│   CP: Strong consistency, may be unavailable                    │
│       (Traditional RDBMS, MongoDB)                              │
│                                                                  │
│   AP: Always available, eventually consistent                   │
│       (Cassandra, DynamoDB)                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## When to Use Each Strategy

| Scenario | Strategy |
|----------|----------|
| Read-heavy, can tolerate lag | Read replicas |
| Write-heavy, need scale | Sharding |
| Simple, moderate growth | Vertical scaling |
| Global users | Geographic sharding |
| Mixed workload | Combination |

---

## Real-World Examples

| Company | Strategy |
|---------|----------|
| **Instagram** | Sharding by user_id |
| **Uber** | Geographic sharding |
| **Netflix** | Read replicas + caching |
| **Slack** | Sharding by workspace |

---

## Interview Tips

When discussing Database Scaling:
1. Start with optimization (indexes, queries)
2. Explain read replicas for read scaling
3. Discuss sharding for write scaling
4. Know sharding strategies and trade-offs
5. Mention CAP theorem implications

