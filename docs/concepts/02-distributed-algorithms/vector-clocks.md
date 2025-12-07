# Vector Clocks & Lamport Timestamps

## What They Are

**Lamport Timestamps** and **Vector Clocks** are mechanisms to establish ordering of events in distributed systems where there's no global clock.

- **Lamport Timestamp**: Single counter, establishes partial order
- **Vector Clock**: One counter per node, detects concurrent events

---

## The Analogy 📅

Imagine **friends texting without timestamps**:
- Alice sends "Want to meet?" 
- Bob sends "I'm free tomorrow"
- Charlie receives both - which came first?

**Lamport**: Like numbering your messages (Message #5, #6, #7)
**Vector Clock**: Like each person tracking everyone's message count

---

## Why They Exist

### The Problem: No Global Clock
```
Node A clock: 10:00:00
Node B clock: 10:00:03  (3 seconds ahead)
Node C clock: 09:59:58  (2 seconds behind)

A sends at "10:00:00" → B receives at "10:00:01"
C sends at "10:00:01" → A receives at "10:00:02"

Which happened first? Wall clocks can't tell us!
```

### What They Solve:
- **Causal ordering** - If A caused B, we know A < B
- **Conflict detection** - Identify concurrent updates
- **Consistency** - Reason about event order without synchronized clocks

---

## Lamport Timestamps

### How It Works:
```
Rules:
1. Before any event: increment local counter
2. Send message: attach current counter
3. Receive message: local = max(local, received) + 1

Example:
┌─────────────────────────────────────────────────────────────────┐
│   Node A          Node B          Node C                         │
│     │               │               │                            │
│   (1) event         │               │                            │
│     │───────────────┼──────────────▶│ receives (2)               │
│     │               │               │                            │
│     │             (1) event         │                            │
│     │◀──────────────│               │                            │
│   (3) receives      │               │                            │
│     │               │               │                            │
└─────────────────────────────────────────────────────────────────┘
```

### Limitations:
- L(a) < L(b) does NOT mean a happened before b
- Cannot detect concurrent events
- Only establishes: if a → b, then L(a) < L(b) (not vice versa)

---

## Vector Clocks

### How It Works:
```
Each node maintains a vector [A_count, B_count, C_count]

Rules:
1. Before event: increment own position
2. Send message: attach full vector
3. Receive: take element-wise max, then increment own

Example:
┌─────────────────────────────────────────────────────────────────┐
│   Node A             Node B             Node C                   │
│   [0,0,0]            [0,0,0]            [0,0,0]                  │
│     │                  │                  │                      │
│   [1,0,0] event        │                  │                      │
│     │─────────────────▶│                  │                      │
│     │                [1,1,0]              │                      │
│     │                  │                  │                      │
│     │                  │──────────────────▶                      │
│     │                  │                [1,1,1]                  │
│     │                  │                  │                      │
│   [2,0,0] event        │                  │                      │
│     │                  │                  │                      │
└─────────────────────────────────────────────────────────────────┘

Comparing vectors:
[2,0,0] vs [1,1,1] → Neither dominates → CONCURRENT
[1,0,0] vs [1,1,1] → [1,0,0] < [1,1,1] → Happened before
```

### Ordering Rules:
```
V(a) < V(b) if:
  - All components of V(a) ≤ V(b), AND
  - At least one component strictly less

V(a) || V(b) (concurrent) if:
  - Neither V(a) < V(b) nor V(b) < V(a)
```

---

## Comparison

| Aspect | Lamport Timestamp | Vector Clock |
|--------|-------------------|--------------|
| **Size** | O(1) single integer | O(n) per node |
| **Detects causality** | Partial (one direction) | Full |
| **Detects concurrency** | No | Yes |
| **Scalability** | Excellent | Limited by node count |
| **Use case** | Ordering events | Conflict detection |

---

## When to Use Each

### Lamport Timestamps ✅:
- **Event ordering** in logs
- **Distributed snapshots** (Chandy-Lamport)
- **Mutual exclusion** algorithms
- When you only need partial ordering

### Vector Clocks ✅:
- **Conflict detection** in replicated data
- **Eventual consistency** systems (Dynamo)
- **Optimistic replication**
- When you need to detect concurrent updates

---

## Real-World Examples

| System | Mechanism |
|--------|-----------|
| **Amazon Dynamo** | Vector clocks for conflict detection |
| **Riak** | Dotted version vectors |
| **CockroachDB** | Hybrid logical clocks |
| **Cassandra** | Timestamps (not vector clocks) |

---

## Practical Considerations

### Vector Clock Explosion:
```
Problem: Vector grows with every client
100,000 clients = 100,000 element vector!

Solutions:
1. Prune old entries
2. Use server-side vector clocks only
3. Dotted Version Vectors (improved algorithm)
```

### Hybrid Logical Clocks (HLC):
Combines wall clock with logical clock:
- Uses physical time when possible
- Falls back to logical increment on conflicts
- Bounded drift from real time

---

## Implementation

### Lamport Timestamp

```python
class LamportClock:
    """Simple logical clock for event ordering."""

    def __init__(self):
        self.time = 0

    def tick(self) -> int:
        """Local event - increment and return."""
        self.time += 1
        return self.time

    def send(self) -> int:
        """Prepare message - return timestamp to attach."""
        return self.tick()

    def receive(self, msg_time: int) -> int:
        """Receive message - sync clocks."""
        self.time = max(self.time, msg_time) + 1
        return self.time
```

### Vector Clock

```python
from typing import Dict

class VectorClock:
    """Tracks causality and detects concurrent events."""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.clock: Dict[str, int] = {node_id: 0}

    def tick(self):
        """Local event."""
        self.clock[self.node_id] = self.clock.get(self.node_id, 0) + 1

    def send(self) -> Dict[str, int]:
        """Return clock to attach to message."""
        self.tick()
        return self.clock.copy()

    def receive(self, other: Dict[str, int]):
        """Merge with received clock."""
        for node, time in other.items():
            self.clock[node] = max(self.clock.get(node, 0), time)
        self.tick()

    def compare(self, other: Dict[str, int]) -> str:
        """
        Compare two vector clocks.
        Returns: 'before', 'after', 'concurrent'
        """
        dominated_by_other = all(
            self.clock.get(k, 0) <= other.get(k, 0)
            for k in set(self.clock) | set(other)
        )
        dominates_other = all(
            other.get(k, 0) <= self.clock.get(k, 0)
            for k in set(self.clock) | set(other)
        )

        if dominated_by_other and not dominates_other:
            return 'before'
        elif dominates_other and not dominated_by_other:
            return 'after'
        elif dominated_by_other and dominates_other:
            return 'equal'
        else:
            return 'concurrent'  # Conflict!

# Usage
a = VectorClock('A')
b = VectorClock('B')

msg = a.send()          # A: {'A': 1}
b.receive(msg)          # B: {'A': 1, 'B': 1}

# Concurrent events
a.tick()                # A: {'A': 2}
b.tick()                # B: {'A': 1, 'B': 2}

print(a.compare(b.clock))  # 'concurrent' - conflict detected!
```

---

## Interview Tips

When discussing Logical Clocks:
1. Start with "no global clock" problem
2. Explain Lamport's 3 rules (tick, send, receive)
3. Show why vector clocks detect concurrency (neither dominates)
4. Discuss scalability trade-offs (O(n) vector size)
5. Mention real systems (Dynamo, Riak)

