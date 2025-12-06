# CRDTs (Conflict-free Replicated Data Types)

## What They Are

**CRDTs** are data structures that can be replicated across multiple nodes, updated independently and concurrently, and always converge to a consistent state without coordination.

---

## The Analogy 🧮

Think of **distributed tally counters**:
- Multiple people counting attendees at different doors
- Each person has their own counter
- At the end, we ADD all counters together
- Order doesn't matter, result is always correct

CRDTs are designed so merging always works, regardless of order or timing.

---

## Why They Exist

### The Problem: Conflicts in Distributed Systems
```
Traditional replication:
Node A: counter = 5, increment → 6
Node B: counter = 5, increment → 6
Sync: Which 6 wins? Both increments lost one!

With coordination (locks, consensus):
- Higher latency
- Reduced availability
- Network partition = unavailable
```

### What CRDTs Solve:
- **Always available** - No coordination needed
- **Eventual consistency** - Guaranteed convergence
- **Conflict-free** - By mathematical design
- **Partition tolerant** - Work offline, sync later

---

## Types of CRDTs

### 1. State-based (CvRDT)
```
┌─────────────────────────────────────────────────────────────────┐
│              State-based CRDT (Convergent)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Node A              Node B              Node C                 │
│   state={1,2}         state={2,3}         state={1,3}           │
│       │                   │                   │                  │
│       └───────────────────┼───────────────────┘                  │
│                           ▼                                      │
│                    merge(A, B, C)                                │
│                    = {1, 2, 3}                                   │
│                                                                  │
│   Merge function: Usually union, max, or LUB                    │
│   Requirement: Merge must be commutative, associative, idempotent│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Operation-based (CmRDT)
```
┌─────────────────────────────────────────────────────────────────┐
│           Operation-based CRDT (Commutative)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Node A: increment()                                            │
│   Node B: increment()                                            │
│                                                                  │
│   Broadcast operations to all nodes                              │
│   Apply in any order → Same result                              │
│                                                                  │
│   Requirement: Operations must be commutative                   │
│   (order doesn't matter)                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common CRDT Types

### G-Counter (Grow-only Counter)
```python
# Each node has its own counter
{
  "node_A": 5,
  "node_B": 3,
  "node_C": 7
}
# Total = 5 + 3 + 7 = 15

# Merge: take max of each node's count
merge(X, Y) = {node: max(X[node], Y[node]) for all nodes}
```

### PN-Counter (Positive-Negative Counter)
```python
# Two G-Counters: one for increments, one for decrements
P = {"A": 10, "B": 5}   # Increments
N = {"A": 2, "B": 1}    # Decrements
Value = sum(P) - sum(N) = 15 - 3 = 12
```

### G-Set (Grow-only Set)
```python
# Only additions, never remove
merge(A, B) = A ∪ B  # Union
```

### OR-Set (Observed-Remove Set)
```python
# Add: attach unique tag
# Remove: remove all observed tags
{
  "apple": ["tag1", "tag2"],  # Added twice
  "banana": ["tag3"]
}
# Remove "apple" → remove tag1, tag2
# If another node adds "apple" with tag4, it survives
```

### LWW-Register (Last-Writer-Wins)
```python
# Each write has timestamp
# Merge: keep value with highest timestamp
{"value": "hello", "timestamp": 1000}
{"value": "world", "timestamp": 1001}  # This wins
```

---

## When to Use CRDTs

### ✅ Good Fit:
- **Collaborative editing** - Google Docs-like apps
- **Distributed counters** - Likes, views, votes
- **Shopping carts** - Add/remove items offline
- **Presence indicators** - Who's online
- **Offline-first apps** - Sync when connected

### ❌ Avoid When:
- Strong consistency required
- Complex invariants (bank balance ≥ 0)
- Bounded data needed (CRDTs often grow)
- Simple single-leader works

---

## Real-World Examples

| System | CRDT Usage |
|--------|------------|
| **Riak** | Built-in CRDT support |
| **Redis** | CRDT-based active-active replication |
| **Apple Notes** | Sync across devices |
| **Figma** | Real-time collaboration |
| **SoundCloud** | Like counters |

---

## Trade-offs

| Aspect | Consideration |
|--------|---------------|
| **Metadata overhead** | Tags, vectors grow over time |
| **Garbage collection** | Need to prune old metadata |
| **Semantics** | Not all operations are CRDT-friendly |
| **Complexity** | Harder to reason about than locks |

---

## Interview Tips

When discussing CRDTs:
1. Explain the "no coordination" benefit
2. Give counter example (G-Counter, PN-Counter)
3. Discuss merge function requirements
4. Mention real systems (Riak, collaborative editors)
5. Discuss limitations (growing metadata, garbage collection)

