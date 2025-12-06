# Consensus Algorithms (Raft & Paxos)

## What They Are

**Consensus algorithms** enable distributed systems to agree on a single value or sequence of values, even when some nodes fail. They're fundamental to building reliable distributed systems like databases, configuration stores, and coordination services.

---

## The Analogy 🗳️

Imagine a **group decision** across different time zones:
- Members can't all meet simultaneously
- Some members might be unreachable
- Need to agree on one decision (not conflicting ones)
- Once decided, everyone must know the outcome

Consensus algorithms are like **voting protocols** that guarantee agreement even when some participants are unavailable.

---

## Why They Exist

### The Problem: Agreement in Distributed Systems
```
Without consensus:
Node A: "Value is 5"
Node B: "Value is 7"
Node C: "Value is 5"

Which is correct? Who decides?
What if Node A crashes mid-update?
```

### What Consensus Solves:
- **Agreement** - All working nodes agree on same value
- **Validity** - Agreed value was proposed by some node
- **Termination** - Eventually a decision is made
- **Fault tolerance** - Works despite f failures (need 2f+1 nodes)

---

## Raft: The Understandable Consensus

### Why Raft?
Paxos is notoriously hard to understand. Raft was designed to be **understandable first**.

### Three Roles:
```
┌─────────────────────────────────────────────────────────────────┐
│                        Raft Roles                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│   │  LEADER  │    │ FOLLOWER │    │ FOLLOWER │                  │
│   │          │───▶│          │    │          │                  │
│   │ (handles │    │ (passive │    │ (passive │                  │
│   │  writes) │───▶│  replica)│    │  replica)│                  │
│   └──────────┘    └──────────┘    └──────────┘                  │
│        │                                                         │
│        └── Sends heartbeats and log entries to followers        │
│                                                                  │
│   CANDIDATE: Temporary state during leader election             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Leader Election:
```
1. Leader sends heartbeats every 150ms
2. Follower doesn't hear heartbeat for 300-500ms
3. Follower becomes CANDIDATE, starts election
4. Candidate votes for itself, requests votes from others
5. Node with majority votes becomes LEADER
6. New leader starts sending heartbeats

Term numbers prevent split-brain (higher term wins)
```

### Log Replication:
```
Client Request → Leader
     │
     ▼
Leader appends to log
     │
     ▼
Leader sends to followers
     │
     ▼
Majority acknowledge
     │
     ▼
Leader commits, responds to client
     │
     ▼
Followers learn of commit via next heartbeat
```

---

## Paxos: The Classic

### Basic Paxos (Single Value):

**Three Roles:**
- **Proposer** - Proposes values
- **Acceptor** - Votes on proposals
- **Learner** - Learns the decided value

**Two Phases:**
```
Phase 1: PREPARE
  Proposer → Acceptors: "Prepare(n)" (proposal number n)
  Acceptors → Proposer: "Promise(n)" or "Already accepted higher"

Phase 2: ACCEPT
  Proposer → Acceptors: "Accept(n, value)"
  Acceptors → Proposer: "Accepted" if still valid
  
When majority accepts → Value is chosen
```

### Multi-Paxos:
Optimizes for multiple values by having a stable leader, reducing Phase 1 overhead.

---

## Raft vs Paxos Comparison

| Aspect | Raft | Paxos |
|--------|------|-------|
| **Understandability** | High (designed for it) | Low (notoriously complex) |
| **Leader** | Always has one leader | Leader-less possible |
| **Log** | Strong leader, simple replication | Flexible, complex |
| **Implementation** | Easier | Harder |
| **Performance** | Similar | Similar |
| **Real-world use** | etcd, CockroachDB, Consul | Chubby, Spanner |

---

## When to Use Consensus

### ✅ Good Fit:
- **Distributed databases** - Replication (CockroachDB, Spanner)
- **Configuration stores** - etcd, Consul, ZooKeeper
- **Leader election** - Who's the primary?
- **Distributed locks** - Coordination between services
- **Atomic broadcast** - Ordered message delivery

### ❌ Avoid When:
- Single node is sufficient
- Eventual consistency is acceptable
- Latency is critical (consensus adds round trips)
- Network is highly unreliable

---

## Key Properties

| Property | Meaning |
|----------|---------|
| **Safety** | Never return wrong value |
| **Liveness** | Eventually returns a value |
| **Fault Tolerance** | Tolerates f failures with 2f+1 nodes |
| **Consistency** | Strong consistency model |

---

## Real-World Implementations

| System | Algorithm | Usage |
|--------|-----------|-------|
| **etcd** | Raft | Kubernetes config store |
| **Consul** | Raft | Service discovery |
| **CockroachDB** | Raft | Distributed SQL |
| **ZooKeeper** | ZAB (Paxos-like) | Coordination |
| **Google Spanner** | Paxos | Global database |
| **Chubby** | Paxos | Lock service |

---

## Interview Tips

When discussing Consensus:
1. Know Raft's leader election and log replication
2. Understand the 2f+1 requirement
3. Explain split-brain prevention (term numbers)
4. Compare Raft's understandability vs Paxos
5. Mention real systems (etcd, Consul, ZooKeeper)

