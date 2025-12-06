# Quorum Algorithms

## What They Are

**Quorum** is a mechanism ensuring consistency in distributed systems by requiring a minimum number of nodes to agree on operations. It's the foundation for balancing consistency and availability.

---

## The Analogy 🗳️

Think of a **board of directors voting**:
- 7 board members total
- Major decisions need majority (4+) approval
- Even if 3 members are absent, decisions can proceed
- Ensures any two voting groups overlap by at least 1 member

---

## Why They Exist

### The Problem: Reading Stale Data
```
3 replicas: A, B, C
Write goes to A (success), B (success), C (fails)

Later:
Read from C → Gets old data!
Read from A or B → Gets new data

How do we guarantee reading latest data?
```

### What Quorum Solves:
- **Read-your-writes** - Guaranteed to see latest
- **Fault tolerance** - Works with some failures
- **Tunable consistency** - Trade-off flexibility
- **No single point of failure** - Any quorum works

---

## How It Works

### Basic Formula:
```
N = Total replicas
W = Write quorum (nodes that must acknowledge write)
R = Read quorum (nodes that must respond to read)

For consistency: W + R > N

This guarantees overlap between write and read sets
```

### Visualization:
```
┌─────────────────────────────────────────────────────────────────┐
│                    Quorum Overlap                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   N = 5 replicas                                                 │
│   W = 3, R = 3  (W + R = 6 > 5 ✓)                               │
│                                                                  │
│   Write to nodes:     ┌───┬───┬───┬───┬───┐                     │
│                       │ A │ B │ C │ D │ E │                     │
│                       │ ✓ │ ✓ │ ✓ │   │   │                     │
│                       └───┴───┴───┴───┴───┘                     │
│                                                                  │
│   Read from nodes:    ┌───┬───┬───┬───┬───┐                     │
│                       │ A │ B │ C │ D │ E │                     │
│                       │   │   │ ✓ │ ✓ │ ✓ │                     │
│                       └───┴───┴───┴───┴───┘                     │
│                                                                  │
│   Overlap at C → Guaranteed to get latest value                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Configurations

| Configuration | W | R | Behavior |
|---------------|---|---|----------|
| **Majority** | ⌈(N+1)/2⌉ | ⌈(N+1)/2⌉ | Balanced consistency |
| **Write-heavy** | N | 1 | Fast reads, slow writes |
| **Read-heavy** | 1 | N | Fast writes, slow reads |
| **Strong consistency** | N | N | All nodes agree |

### Examples with N=3:
```
W=2, R=2: Standard quorum
- Write to 2, read from 2
- Tolerates 1 failure

W=3, R=1: Write-all, read-one
- Every write to all nodes
- Any node has latest
- Write fails if any node down

W=1, R=3: Write-one, read-all
- Writes always succeed
- Must read all to find latest
- Good for write-heavy
```

---

## Sloppy Quorum

### Problem with Strict Quorum:
```
3 replicas: A, B, C
A and B are down
Strict quorum (W=2): Cannot write!

Solution: Sloppy quorum
Write to C and D (D is not usual replica)
When A/B recover, D hands off data (hinted handoff)
```

### Trade-off:
- **Higher availability** - More writes succeed
- **Lower consistency** - D might not be read

---

## Read Repair

```
┌─────────────────────────────────────────────────────────────────┐
│                      Read Repair                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Read with R=2 from A, B, C:                                   │
│   A: value="new", version=2                                      │
│   B: value="new", version=2                                      │
│   C: value="old", version=1  ← Stale!                           │
│                                                                  │
│   Coordinator:                                                   │
│   1. Returns "new" to client (latest version wins)              │
│   2. Sends "new" to C to repair                                  │
│                                                                  │
│   C now has latest value                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## When to Use

### ✅ Good Fit:
- **Distributed databases** - DynamoDB, Cassandra, Riak
- **Replicated storage** - Object stores
- **Consensus systems** - Raft, Paxos
- **Tunable consistency needs**

### ❌ Avoid When:
- Single-node sufficient
- Need ACID transactions
- Cannot tolerate any staleness

---

## CAP Theorem Connection

```
Quorum allows tuning between:

W + R > N: Strong consistency (CP)
- Always get latest value
- Some operations may fail during partition

W + R ≤ N: Eventual consistency (AP)
- May read stale data
- Higher availability
```

---

## Real-World Examples

| System | Default Quorum |
|--------|----------------|
| **Cassandra** | Configurable (ONE, QUORUM, ALL) |
| **DynamoDB** | Configurable per operation |
| **Riak** | N=3, R=2, W=2 |
| **etcd/Raft** | Majority (N/2 + 1) |
| **ZooKeeper** | Majority |

---

## Interview Tips

When discussing Quorum:
1. Explain the W + R > N formula
2. Draw the overlap diagram
3. Discuss read-heavy vs write-heavy configs
4. Mention sloppy quorum for availability
5. Connect to CAP theorem trade-offs

