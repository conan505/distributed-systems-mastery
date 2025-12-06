# Split-Brain Resolution

## What It Is

**Split-Brain** occurs when a distributed system partitions into multiple segments that can't communicate, each believing it's the primary and operating independently, leading to data inconsistency.

---

## The Analogy 🧠

Think of **conjoined twins separated by accident**:
- Each twin thinks they're the original
- Both make decisions independently
- When reunited, their memories conflict
- Need to reconcile differences

---

## Why It's a Problem

### The Split-Brain Scenario
```
┌─────────────────────────────────────────────────────────────────┐
│                    Split-Brain Scenario                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Before partition:                                              │
│   [Node A: Primary] ←──── replication ────→ [Node B: Replica]   │
│                                                                  │
│   Network partition occurs:                                      │
│   [Node A: Primary]  ╳╳╳╳╳╳╳╳╳╳╳╳╳╳╳╳╳╳╳╳  [Node B: Replica]   │
│                                                                  │
│   Node B: "A is dead, I'll become primary!"                     │
│                                                                  │
│   [Node A: Primary]                        [Node B: Primary!]   │
│   Accepts writes                           Accepts writes        │
│   balance = 100                            balance = 50          │
│                                                                  │
│   Partition heals:                                               │
│   Which balance is correct? Data is corrupted!                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Consequences:
- **Data corruption** - Conflicting writes
- **Lost updates** - One side's changes discarded
- **Inconsistent state** - Different views of data
- **Application errors** - Unexpected behavior

---

## Prevention Strategies

### 1. Quorum-Based Decisions
```
Require majority to take action

5-node cluster:
- Need 3 nodes to agree (majority)
- Partition: [A, B] vs [C, D, E]
- [A, B] can't reach quorum (only 2)
- [C, D, E] has quorum (3) → stays primary

Only one partition can have majority!
```

### 2. Fencing Tokens
```python
class FencingToken:
    """Ensure only one primary can write."""
    
    def __init__(self, storage):
        self.storage = storage
        self.current_token = 0
    
    def acquire_leadership(self) -> int:
        self.current_token += 1
        return self.current_token
    
    def write(self, token: int, key: str, value: str):
        if token < self.current_token:
            raise StaleTokenError("Fenced out")
        self.storage.write(key, value)
```

```
Sequence:
1. Node A becomes primary, gets token=1
2. Partition occurs
3. Node B becomes primary, gets token=2
4. Partition heals
5. Node A tries to write with token=1
6. Storage rejects: "Token 1 is stale, current is 2"
```

### 3. STONITH (Shoot The Other Node In The Head)
```
When split-brain detected:
1. Use out-of-band mechanism to kill other node
2. Power off via IPMI/BMC
3. Cut storage access

Ensures only one node can operate
Brutal but effective
```

---

## Detection Methods

### 1. Heartbeat Monitoring
```python
class HeartbeatMonitor:
    def __init__(self, nodes: list, timeout_ms: int = 5000):
        self.nodes = nodes
        self.timeout = timeout_ms
        self.last_heartbeat = {}
    
    def check_partition(self) -> bool:
        now = time.time() * 1000
        unreachable = [
            node for node in self.nodes
            if now - self.last_heartbeat.get(node, 0) > self.timeout
        ]
        
        if len(unreachable) > 0 and len(unreachable) < len(self.nodes):
            return True  # Partial failure = possible split
        return False
```

### 2. Witness/Arbiter Node
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│         [Arbiter]  ←── Lightweight witness node                 │
│          /     \                                                 │
│         /       \                                                │
│   [Node A]     [Node B]                                         │
│                                                                  │
│   If A and B can't communicate:                                 │
│   - Node that CAN reach Arbiter stays primary                   │
│   - Node that CAN'T reach Arbiter demotes itself               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Resolution Strategies

### 1. Last-Writer-Wins (LWW)
```
When partitions merge:
- Compare timestamps
- Keep most recent write
- Discard older writes

Simple but may lose data
Used by: Cassandra, DynamoDB
```

### 2. Merge Resolution
```python
def merge_conflicts(value_a, value_b, timestamp_a, timestamp_b):
    # Domain-specific merge logic
    
    # For counters: sum them
    if is_counter(value_a):
        return value_a + value_b
    
    # For sets: union them
    if is_set(value_a):
        return value_a.union(value_b)
    
    # For documents: field-level merge
    return merge_documents(value_a, value_b)
```

### 3. CRDTs (Conflict-Free)
```
Use data structures that merge automatically:
- G-Counter: Grow-only counter
- PN-Counter: Positive-negative counter
- OR-Set: Observed-remove set
- LWW-Register: Last-writer-wins register

See: crdts.md for details
```

---

## Database-Specific Solutions

| Database | Strategy |
|----------|----------|
| **PostgreSQL** | Quorum + witness |
| **MongoDB** | Replica set election |
| **Redis Sentinel** | Quorum-based failover |
| **Cassandra** | Quorum + LWW |
| **CockroachDB** | Raft consensus |

---

## Best Practices

```
1. Use odd number of nodes (3, 5, 7)
   - Ensures clear majority

2. Implement fencing
   - Prevent stale primaries from writing

3. Prefer CP over AP when data critical
   - Unavailable > inconsistent

4. Monitor for partition events
   - Alert on network issues

5. Test partition scenarios
   - Chaos engineering

6. Have manual resolution procedures
   - For when automation fails
```

---

## When to Use Each Strategy

| Scenario | Strategy |
|----------|----------|
| **Financial data** | Quorum + fencing (CP) |
| **Cache** | LWW acceptable |
| **Counters** | CRDTs |
| **User preferences** | Merge or LWW |
| **Critical infrastructure** | STONITH |

---

## Interview Tips

When discussing Split-Brain:
1. Explain the problem clearly
2. Describe quorum-based prevention
3. Discuss fencing tokens
4. Mention resolution strategies (LWW, merge)
5. Know CAP theorem implications

