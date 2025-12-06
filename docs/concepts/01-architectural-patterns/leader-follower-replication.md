# Leader-Follower Replication

## What It Is

**Leader-Follower** (Master-Slave) replication is a pattern where one node (leader) handles all writes and propagates changes to read-only replicas (followers). It's the most common database replication strategy.

---

## The Analogy 📚

Think of a **professor and teaching assistants**:
- **Professor (Leader)**: Creates the original lecture notes
- **TAs (Followers)**: Get copies and help students with questions
- Students can ask any TA for information
- Only the professor can update the official material

---

## Why It Exists

### The Problem: Scaling Reads
```
Single database:
- 1000 reads/sec
- 100 writes/sec
- Read bottleneck!

Scaling writes is hard (need consistency)
Scaling reads is easier (add replicas)
```

### What Leader-Follower Solves:
- **Read scalability** - Distribute reads across followers
- **Fault tolerance** - Followers can become leaders
- **Geographic distribution** - Followers near users
- **Separation of concerns** - Analytics on followers

---

## How It Works

### Architecture:
```
┌─────────────────────────────────────────────────────────────────┐
│               Leader-Follower Replication                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   WRITES                         READS                           │
│     │                              │                             │
│     ▼                              ▼                             │
│   ┌────────────┐              Load Balancer                      │
│   │   LEADER   │               ┌───┴───┐                         │
│   │ (Primary)  │               │       │                         │
│   └─────┬──────┘               ▼       ▼                         │
│         │              ┌──────────┐ ┌──────────┐                 │
│         │              │ FOLLOWER │ │ FOLLOWER │                 │
│         │              │  (Read)  │ │  (Read)  │                 │
│         │              └────▲─────┘ └────▲─────┘                 │
│         │                   │            │                       │
│         └───────────────────┴────────────┘                       │
│              Replication stream (async or sync)                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Replication Methods:

**Statement-based:**
```sql
-- Leader sends SQL statements
INSERT INTO users (name) VALUES ('Alice');
-- Problem: Non-deterministic functions (NOW(), RAND())
```

**Row-based (WAL):**
```
-- Leader sends actual row changes
INSERT: {id: 5, name: 'Alice', created_at: '2024-01-15'}
-- More data, but deterministic
```

**Logical (Change Data Capture):**
```json
{"op": "insert", "table": "users", "data": {...}}
```

---

## Sync vs Async Replication

### Synchronous:
```
┌─────────────────────────────────────────────────────────────────┐
│   Client    Leader      Follower1    Follower2                   │
│     │         │             │            │                       │
│     │──write──▶             │            │                       │
│     │         │──replicate──▶            │                       │
│     │         │◀────ack─────│            │                       │
│     │         │──replicate───────────────▶                       │
│     │         │◀────ack──────────────────│                       │
│     │◀──ok────│             │            │                       │
│                                                                  │
│   + Strong consistency                                           │
│   - High latency, reduced availability                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Asynchronous:
```
┌─────────────────────────────────────────────────────────────────┐
│   Client    Leader      Follower1    Follower2                   │
│     │         │             │            │                       │
│     │──write──▶             │            │                       │
│     │◀──ok────│             │            │                       │
│     │         │──replicate──▶            │  (later)              │
│     │         │──replicate───────────────▶                       │
│                                                                  │
│   + Low latency, high availability                              │
│   - Eventual consistency (replication lag)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Replication Lag Problems

### 1. Read-Your-Writes
```
User writes post → Reads from follower → Post not there yet!

Solutions:
- Read own writes from leader
- Wait for replication before returning
- Use monotonic reads (same follower)
```

### 2. Monotonic Reads
```
User reads from Follower1 (up-to-date)
User reads from Follower2 (lagging) → Data goes backward!

Solution: Sticky sessions to same follower
```

---

## Failover

### Automatic Failover:
```
1. Detect leader failure (heartbeat timeout)
2. Choose new leader (most up-to-date follower)
3. Reconfigure followers to new leader
4. Clients redirect to new leader

Challenges:
- Split-brain (two leaders)
- Lost writes (async replication)
- Choosing right follower
```

---

## When to Use

### ✅ Good Fit:
- **Read-heavy workloads** - Read/write ratio > 10:1
- **Geographic distribution** - Followers near users
- **Reporting/Analytics** - Run on followers
- **High availability** - Failover capability

### ❌ Avoid When:
- Write-heavy workloads
- Need multi-master writes
- Strong consistency everywhere
- Very low replication lag required

---

## Real-World Examples

| Database | Notes |
|----------|-------|
| **PostgreSQL** | Streaming replication |
| **MySQL** | binlog replication |
| **MongoDB** | Replica sets |
| **Redis** | Master-slave replication |
| **Kafka** | Leader partitions with ISR |

---

## Interview Tips

When discussing Leader-Follower:
1. Explain read scaling benefit
2. Discuss sync vs async trade-offs
3. Mention replication lag problems
4. Describe failover challenges
5. Compare with multi-leader

