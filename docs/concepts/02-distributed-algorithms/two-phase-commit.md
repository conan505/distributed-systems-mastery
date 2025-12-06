# Two-Phase Commit (2PC) & Three-Phase Commit (3PC)

## What They Are

**Two-Phase Commit (2PC)** is a distributed algorithm that ensures all participants in a transaction either commit or abort together. It's the classic solution for distributed transactions.

**Three-Phase Commit (3PC)** adds an extra phase to reduce blocking in failure scenarios.

---

## The Analogy 💒

Think of a **wedding ceremony**:

**2PC:**
- Phase 1 (Prepare): "Do you take this person?" → Both say "I do"
- Phase 2 (Commit): "I now pronounce you married"

If either says "no" in Phase 1, wedding is cancelled.

**3PC:**
- Phase 1: "Are you ready to commit?"
- Phase 2: "Prepare to commit" (pre-commit)
- Phase 3: "Now commit"

---

## Why They Exist

### The Problem: Distributed Transactions
```
Transfer $100 from Bank A to Bank B:
1. Bank A: Deduct $100
2. Bank B: Add $100

What if Bank B fails after Bank A deducts?
- Money disappears!
- Need atomic commit across both banks
```

### What 2PC Solves:
- **Atomicity** - All or nothing across nodes
- **Consistency** - No partial commits
- **Coordination** - Agreement on outcome

---

## Two-Phase Commit (2PC)

### The Protocol:
```
┌─────────────────────────────────────────────────────────────────┐
│                    Two-Phase Commit                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   PHASE 1: PREPARE (Voting)                                      │
│   ┌─────────────┐                                               │
│   │ Coordinator │                                               │
│   └──────┬──────┘                                               │
│          │ PREPARE                                               │
│     ┌────┴────┬────────┐                                        │
│     ▼         ▼        ▼                                        │
│   ┌───┐    ┌───┐    ┌───┐                                       │
│   │ A │    │ B │    │ C │  Participants                         │
│   └─┬─┘    └─┬─┘    └─┬─┘                                       │
│     │ YES    │ YES    │ YES                                      │
│     └────────┴────────┘                                         │
│                                                                  │
│   PHASE 2: COMMIT (or ABORT)                                     │
│   ┌─────────────┐                                               │
│   │ Coordinator │ (All voted YES)                               │
│   └──────┬──────┘                                               │
│          │ COMMIT                                                │
│     ┌────┴────┬────────┐                                        │
│     ▼         ▼        ▼                                        │
│   ┌───┐    ┌───┐    ┌───┐                                       │
│   │ A │    │ B │    │ C │  All commit                           │
│   └───┘    └───┘    └───┘                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### State Machine:
```
Coordinator:
INIT → WAITING → COMMITTED/ABORTED

Participant:
INIT → PREPARED → COMMITTED/ABORTED
```

---

## 2PC Problems

### 1. Blocking Problem
```
Scenario:
1. Coordinator sends PREPARE
2. All participants vote YES
3. Coordinator crashes before sending COMMIT
4. Participants are STUCK (holding locks)
   - Can't commit (no instruction)
   - Can't abort (might have committed elsewhere)
```

### 2. Single Point of Failure
- Coordinator failure blocks everyone
- Need coordinator recovery mechanism

### 3. Performance
- Synchronous protocol
- Multiple round trips
- Locks held during entire process

---

## Three-Phase Commit (3PC)

### Added Phase: Pre-Commit
```
┌─────────────────────────────────────────────────────────────────┐
│                    Three-Phase Commit                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Phase 1: CAN-COMMIT?                                           │
│   Coordinator → Participants: "Can you commit?"                  │
│   Participants → Coordinator: "Yes" or "No"                      │
│                                                                  │
│   Phase 2: PRE-COMMIT                                            │
│   Coordinator → Participants: "Prepare to commit"                │
│   Participants: Acquire locks, prepare                           │
│   Participants → Coordinator: "ACK"                              │
│                                                                  │
│   Phase 3: DO-COMMIT                                             │
│   Coordinator → Participants: "Commit now"                       │
│   Participants: Commit and release locks                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why 3PC Helps:
- If coordinator fails after PRE-COMMIT, participants can timeout and commit
- Non-blocking in some failure scenarios
- But still vulnerable to network partitions

---

## Comparison

| Aspect | 2PC | 3PC |
|--------|-----|-----|
| **Phases** | 2 | 3 |
| **Blocking** | Yes (coordinator failure) | Reduced |
| **Messages** | 4n | 6n |
| **Latency** | Lower | Higher |
| **Network partition safe** | No | No |
| **Practical use** | Common | Rare |

---

## When to Use

### ✅ 2PC Good For:
- **Database transactions** - XA transactions
- **Microservice sagas** - Prepare/commit pattern
- **Controlled environments** - Low failure probability
- **Short transactions** - Minimize lock time

### ❌ Avoid When:
- High availability required
- Long-running transactions
- Unreliable networks
- Need for partition tolerance

---

## Modern Alternatives

| Alternative | Approach |
|-------------|----------|
| **Saga Pattern** | Compensating transactions |
| **TCC** | Try-Confirm-Cancel |
| **Eventual Consistency** | Accept temporary inconsistency |
| **Consensus (Paxos/Raft)** | Replicated state machines |

---

## Real-World Usage

| System | Usage |
|--------|-------|
| **MySQL/PostgreSQL** | XA transactions |
| **Oracle** | Distributed transactions |
| **JTA** | Java Transaction API |
| **Spanner** | Modified 2PC with Paxos |

---

## Interview Tips

When discussing 2PC/3PC:
1. Explain the wedding analogy
2. Describe the blocking problem
3. Know why 3PC doesn't fully solve it
4. Mention modern alternatives (Saga, TCC)
5. Discuss trade-offs (consistency vs availability)

