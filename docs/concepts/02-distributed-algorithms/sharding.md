# Sharding (Partitioning) Strategies

## What It Is

**Sharding** (horizontal partitioning) divides data across multiple database instances, where each shard holds a subset of the data. It's the primary technique for scaling beyond single-machine limits.

---

## The Analogy 📚

Think of a **library system**:
- One library can't hold all books
- Split by category: Fiction→Library A, Science→Library B
- Or by author: A-M→Library A, N-Z→Library B
- Each library manages its portion independently

---

## Why It Exists

### The Problem: Single-Node Limits
```
Single database:
- 1TB storage limit
- 10,000 QPS max
- Data doesn't fit in memory

You need 50TB storage, 500,000 QPS
Solution: 50 shards
```

### What Sharding Solves:
- **Storage scaling** - Distribute data across machines
- **Throughput scaling** - Parallel query processing
- **Memory scaling** - Each shard's data fits in RAM
- **Geographic distribution** - Data near users

---

## Sharding Strategies

### 1. Range-Based Sharding
```
┌─────────────────────────────────────────────────────────────────┐
│                    Range-Based Sharding                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Key: user_id (1 to 1,000,000)                                 │
│                                                                  │
│   Shard 1: user_id 1 - 333,333                                  │
│   Shard 2: user_id 333,334 - 666,666                            │
│   Shard 3: user_id 666,667 - 1,000,000                          │
│                                                                  │
│   ✅ Range queries efficient                                     │
│   ✅ Easy to understand                                          │
│   ❌ Hotspots (new users all go to last shard)                  │
│   ❌ Uneven distribution                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Hash-Based Sharding
```
┌─────────────────────────────────────────────────────────────────┐
│                    Hash-Based Sharding                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   shard = hash(user_id) % num_shards                            │
│                                                                  │
│   hash("user_123") = 7421 → 7421 % 4 = 1 → Shard 1             │
│   hash("user_456") = 2938 → 2938 % 4 = 2 → Shard 2             │
│                                                                  │
│   ✅ Even distribution                                           │
│   ✅ No hotspots                                                 │
│   ❌ Range queries need all shards                              │
│   ❌ Resharding is painful                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Consistent Hashing
```
┌─────────────────────────────────────────────────────────────────┐
│                  Consistent Hashing                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Hash ring with virtual nodes:                                  │
│                    Shard A                                       │
│                      │                                           │
│             ┌────────┼────────┐                                 │
│             │        │        │                                 │
│    Shard D ─┤   ●────●        ├─ Shard B                        │
│             │        │        │                                 │
│             └────────┼────────┘                                 │
│                      │                                           │
│                    Shard C                                       │
│                                                                  │
│   ✅ Minimal data movement on resize                            │
│   ✅ Virtual nodes for balance                                   │
│   See: consistent-hashing.md for details                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Directory-Based Sharding
```
┌─────────────────────────────────────────────────────────────────┐
│                 Directory-Based Sharding                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Lookup service: user_id → shard                               │
│   ┌─────────────────────────────┐                               │
│   │ user_123 → Shard 2          │                               │
│   │ user_456 → Shard 1          │                               │
│   │ user_789 → Shard 3          │                               │
│   └─────────────────────────────┘                               │
│                                                                  │
│   ✅ Flexible mapping                                            │
│   ✅ Easy rebalancing                                            │
│   ❌ Lookup service is SPOF                                      │
│   ❌ Extra network hop                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Choosing a Shard Key

### Good Shard Keys:
```
✅ High cardinality (many unique values)
✅ Even distribution
✅ Matches query patterns
✅ Immutable (or rarely changes)

Examples:
- user_id (for user-centric apps)
- tenant_id (for multi-tenant SaaS)
- order_id (for order processing)
```

### Bad Shard Keys:
```
❌ Low cardinality (status: active/inactive)
❌ Monotonically increasing (timestamp, auto-increment)
❌ Frequently updated
❌ Mismatched with queries
```

---

## Cross-Shard Operations

### Problem:
```sql
-- This query needs ALL shards!
SELECT * FROM orders WHERE status = 'pending'
```

### Solutions:
```
1. Scatter-gather: Query all shards, merge results
   - Simple but slow

2. Global secondary index: 
   - Separate index pointing to shards
   - Adds write overhead

3. Denormalization:
   - Duplicate data on multiple shards
   - Keep related data together
```

---

## When to Shard

### ✅ Good Time to Shard:
- Data exceeds single-node capacity
- Queries exceed single-node throughput
- Need geographic distribution

### ❌ Avoid/Delay When:
- Read replicas can handle load
- Vertical scaling still possible
- Application complexity can't handle it

---

## Real-World Examples

| System | Strategy |
|--------|----------|
| **MongoDB** | Range or hash sharding |
| **Cassandra** | Consistent hashing |
| **Vitess (YouTube)** | Range-based |
| **CockroachDB** | Range-based with auto-splitting |
| **DynamoDB** | Hash-based with partition key |

---

## Interview Tips

When discussing Sharding:
1. Explain the need (single-node limits)
2. Compare strategies (range vs hash)
3. Discuss shard key selection
4. Mention cross-shard query challenges
5. Give real examples (Cassandra, MongoDB)

