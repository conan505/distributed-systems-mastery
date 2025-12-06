# Leader Election

## What It Is

**Leader Election** is a distributed algorithm that selects one node from a cluster to act as the coordinator or leader, responsible for critical tasks like writes, scheduling, or coordination.

---

## The Analogy 👑

Think of **electing a class president**:
- Any student can run for president
- Students vote
- Highest votes wins
- If president leaves school → new election
- Only one president at a time

---

## Why It Exists

### The Problem: Who's in Charge?
```
Scenario: Database cluster with 5 nodes
- All need to know who handles writes
- If leader fails, need new leader
- Can't have two leaders (split-brain)

Without leader election:
- Split-brain: Two nodes think they're leader
- No leader: System stalls
- Conflicting decisions
```

### What Leader Election Solves:
- **Single point of coordination** - One decision maker
- **Fault tolerance** - Automatic failover
- **Consistency** - Avoid conflicts
- **Partition handling** - Only majority can elect

---

## Common Algorithms

### 1. Bully Algorithm
```
┌─────────────────────────────────────────────────────────────────┐
│                    Bully Algorithm                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Nodes: 1, 2, 3, 4, 5 (higher ID = higher priority)           │
│   Current leader: 5                                              │
│                                                                  │
│   5 fails:                                                       │
│   1. Node 3 detects, sends ELECTION to 4, 5                     │
│   2. Node 4 responds OK (bullies 3)                              │
│   3. Node 5 no response (dead)                                   │
│   4. Node 4 sends ELECTION to 5                                  │
│   5. No response → 4 becomes LEADER                             │
│   6. Node 4 broadcasts COORDINATOR message                       │
│                                                                  │
│   + Simple, deterministic                                        │
│   - High message complexity O(n²)                                │
│   - Assumes reliable failure detection                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Ring Algorithm
```
┌─────────────────────────────────────────────────────────────────┐
│                    Ring Algorithm                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│              1 ──────▶ 2                                        │
│              ▲          │                                        │
│              │          ▼                                        │
│              5          3                                        │
│              ▲          │                                        │
│              │          ▼                                        │
│              └──── 4 ◀──┘                                       │
│                                                                  │
│   1. Node detects failure, sends ELECTION with its ID           │
│   2. Each node adds its ID, forwards to next                    │
│   3. Message returns to initiator with all alive IDs            │
│   4. Highest ID elected, COORDINATOR broadcasted                │
│                                                                  │
│   + Lower message complexity O(n)                                │
│   - Single ring failure problematic                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Raft-based Election
```
┌─────────────────────────────────────────────────────────────────┐
│                    Raft Leader Election                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   States: FOLLOWER → CANDIDATE → LEADER                         │
│                                                                  │
│   1. Followers wait for heartbeat (random timeout)              │
│   2. On timeout, become CANDIDATE, increment term               │
│   3. Vote for self, request votes from others                   │
│   4. Win with majority → become LEADER                          │
│   5. Leader sends heartbeats to prevent new elections           │
│                                                                  │
│   Term numbers prevent split-brain:                              │
│   - Higher term always wins                                      │
│   - Stale leaders step down                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Split-Brain Prevention

### Problem:
```
Network partition:
[Node1, Node2] | [Node3, Node4, Node5]

Without protection:
- Left side elects Node2 as leader
- Right side elects Node5 as leader
- Two leaders = data corruption!
```

### Solutions:
```
1. Quorum requirement: Need majority (n/2 + 1) to elect
   - Left: 2 nodes, need 3 → can't elect
   - Right: 3 nodes, need 3 → can elect

2. Fencing tokens: Each leader gets unique, increasing token
   - Resources only accept highest token

3. Lease-based: Leader holds time-limited lease
   - Must renew before expiration
```

---

## When to Use

### ✅ Good Fit:
- **Database primary selection** - Write coordination
- **Distributed locks** - Lock manager election
- **Job scheduling** - Single scheduler
- **Consensus groups** - Raft/Paxos leader

### ❌ Avoid When:
- Can use external coordinator (ZooKeeper, etcd)
- Leaderless design works (Cassandra, DynamoDB)
- Single node sufficient

---

## Real-World Implementations

| System | Approach |
|--------|----------|
| **ZooKeeper** | ZAB protocol, built-in election |
| **etcd** | Raft leader election |
| **Kafka** | Controller election via ZooKeeper |
| **Redis Sentinel** | Quorum-based election |
| **Consul** | Raft for server leadership |

---

## Using ZooKeeper for Leader Election

```python
# Ephemeral sequential nodes
def elect_leader(zk):
    # Create ephemeral sequential node
    my_node = zk.create("/election/node-", ephemeral=True, sequence=True)
    
    while True:
        children = sorted(zk.get_children("/election"))
        
        if my_node == children[0]:
            # I'm the leader!
            return "LEADER"
        else:
            # Watch the node before me
            prev_node = children[children.index(my_node) - 1]
            zk.exists(f"/election/{prev_node}", watch=on_change)
            wait_for_change()
```

---

## Interview Tips

When discussing Leader Election:
1. Explain why single leader is needed
2. Describe split-brain problem and quorum solution
3. Know Raft's approach (terms, heartbeats)
4. Mention practical tools (ZooKeeper, etcd)
5. Discuss trade-offs (availability during election)

