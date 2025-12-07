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

## Implementation

### Two-Phase Commit Coordinator

```python
from enum import Enum
from typing import List, Dict
import threading

class TxState(Enum):
    INIT = "init"
    PREPARING = "preparing"
    PREPARED = "prepared"
    COMMITTING = "committing"
    COMMITTED = "committed"
    ABORTED = "aborted"

class TwoPhaseCommitCoordinator:
    """
    Simplified 2PC coordinator with timeout handling.
    """

    def __init__(self, participants: List[str], rpc_client):
        self.participants = participants
        self.rpc = rpc_client
        self.state = TxState.INIT
        self.votes: Dict[str, bool] = {}
        self.timeout = 30  # seconds

    def execute(self, transaction) -> bool:
        """Execute distributed transaction."""
        try:
            # Phase 1: Prepare
            if not self._prepare_phase(transaction):
                self._abort_phase()
                return False

            # Phase 2: Commit
            self._commit_phase()
            return True

        except Exception as e:
            self._abort_phase()
            return False

    def _prepare_phase(self, transaction) -> bool:
        """Ask all participants to prepare."""
        self.state = TxState.PREPARING

        for participant in self.participants:
            try:
                vote = self.rpc(participant, "prepare", transaction,
                              timeout=self.timeout)
                self.votes[participant] = vote
                if not vote:
                    return False  # Any NO = abort
            except TimeoutError:
                return False

        self.state = TxState.PREPARED
        return True

    def _commit_phase(self):
        """Tell all participants to commit."""
        self.state = TxState.COMMITTING

        for participant in self.participants:
            # Retry until success (commit must eventually succeed)
            while True:
                try:
                    self.rpc(participant, "commit", timeout=self.timeout)
                    break
                except TimeoutError:
                    continue  # Keep retrying

        self.state = TxState.COMMITTED

    def _abort_phase(self):
        """Tell all participants to abort."""
        for participant in self.participants:
            try:
                self.rpc(participant, "abort", timeout=self.timeout)
            except:
                pass  # Best effort

        self.state = TxState.ABORTED
```

### Participant Implementation

```python
class TwoPhaseCommitParticipant:
    """Participant in 2PC transaction."""

    def __init__(self, storage):
        self.storage = storage
        self.pending_tx = None
        self.state = TxState.INIT

    def prepare(self, transaction) -> bool:
        """
        Prepare to commit - acquire locks, validate.
        Return True if ready to commit.
        """
        try:
            # Acquire locks
            self.storage.lock(transaction.resources)

            # Validate transaction
            if not self.storage.validate(transaction):
                self.storage.unlock(transaction.resources)
                return False

            # Write to WAL (survive crashes)
            self.storage.write_wal("PREPARED", transaction)

            self.pending_tx = transaction
            self.state = TxState.PREPARED
            return True

        except Exception:
            return False

    def commit(self):
        """Commit the prepared transaction."""
        if self.state != TxState.PREPARED:
            raise Exception("Not prepared")

        self.storage.apply(self.pending_tx)
        self.storage.write_wal("COMMITTED", self.pending_tx)
        self.storage.unlock(self.pending_tx.resources)

        self.state = TxState.COMMITTED
        self.pending_tx = None

    def abort(self):
        """Abort and rollback."""
        if self.pending_tx:
            self.storage.unlock(self.pending_tx.resources)
            self.storage.write_wal("ABORTED", self.pending_tx)

        self.state = TxState.ABORTED
        self.pending_tx = None
```

---

## Interview Tips

When discussing 2PC/3PC:
1. Explain the wedding analogy
2. Describe the blocking problem (coordinator crash after prepare)
3. Know why 3PC doesn't fully solve it (network partitions)
4. Mention modern alternatives (Saga, TCC, eventual consistency)
5. Discuss trade-offs (strong consistency vs availability)

