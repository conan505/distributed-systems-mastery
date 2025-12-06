# CQRS & Event Sourcing

## What It Is

**CQRS (Command Query Responsibility Segregation)** separates read and write operations into different models. Commands (writes) go to one model, queries (reads) go to another.

**Event Sourcing** stores state changes as a sequence of events rather than just the current state. Instead of storing "balance = $100", you store "deposited $50", "deposited $60", "withdrew $10".

---

## The Analogy 🏦

Think of a **bank ledger**:
- **Traditional approach**: You only see the current balance ($100)
- **Event Sourcing**: You see every transaction that led to that balance

Now imagine the bank has:
- **Tellers** (write operations) - they process deposits/withdrawals
- **ATMs** (read operations) - they just show your balance

**CQRS** says: Why use the same system for both? Tellers need transactional guarantees, ATMs just need fast reads.

---

## Why It Exists

### Problems with Traditional CRUD:
1. **Read/Write Contention**: Same database handles both, causing locks
2. **Scaling Mismatch**: Reads are typically 10-100x more frequent than writes
3. **Lost History**: You only know current state, not how you got there
4. **Complex Queries**: Write-optimized schemas are bad for complex reads

### What CQRS + Event Sourcing Solve:
- Scale reads and writes independently
- Complete audit trail of all changes
- Time-travel debugging (replay events to any point)
- Rebuild read models without touching write logic

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                         CQRS Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐         ┌─────────────┐        ┌──────────────┐  │
│   │  Client  │────────▶│  Command    │───────▶│  Write DB    │  │
│   │          │         │  Handler    │        │  (Events)    │  │
│   └──────────┘         └─────────────┘        └──────┬───────┘  │
│        │                                              │         │
│        │                                              ▼         │
│        │                                      ┌──────────────┐  │
│        │                                      │  Event Bus   │  │
│        │                                      └──────┬───────┘  │
│        │                                              │         │
│        ▼                                              ▼         │
│   ┌──────────┐         ┌─────────────┐        ┌──────────────┐  │
│   │  Query   │◀────────│  Query      │◀───────│  Read DB     │  │
│   │  Result  │         │  Handler    │        │  (Projections)│ │
│   └──────────┘         └─────────────┘        └──────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Event Sourcing Flow:
1. **Command received**: "Transfer $50 from A to B"
2. **Event created**: `MoneyTransferred { from: A, to: B, amount: 50 }`
3. **Event stored**: Append-only to event store
4. **Projections updated**: Read models rebuilt from events

---

## When to Use It

### ✅ Good Fit:
- **Financial systems** - Need complete audit trail
- **E-commerce orders** - Track order lifecycle
- **Collaborative apps** - Google Docs-style real-time sync
- **Gaming** - Replay and undo functionality
- **High read-to-write ratio** - News feeds, dashboards

### ❌ Avoid When:
- Simple CRUD with no history needs
- Small-scale applications
- When eventual consistency is unacceptable
- Team lacks distributed systems experience

---

## How to Use It Effectively

### Best Practices:

1. **Design Events as Facts**
   ```
   ✅ OrderPlaced, ItemAddedToCart, PaymentReceived
   ❌ OrderUpdated, CartModified (too vague)
   ```

2. **Events are Immutable** - Never modify past events

3. **Handle Eventual Consistency**
   - Read models may lag behind writes
   - Use optimistic UI updates

4. **Event Versioning** - Plan for schema evolution
   ```json
   { "type": "OrderPlaced", "version": 2, "data": {...} }
   ```

5. **Snapshots for Performance** - Don't replay 1M events every time
   - Take snapshots every N events
   - Rebuild from snapshot + recent events

---

## Real-World Examples

| Company | Use Case |
|---------|----------|
| **Netflix** | Viewing history, recommendations |
| **Uber** | Trip events, driver location updates |
| **Banking** | Transaction ledgers |
| **Kafka** | Event log is the source of truth |

---

## Interview Tips

When asked about CQRS in interviews:
1. Explain the separation of concerns clearly
2. Discuss trade-offs (complexity vs scalability)
3. Mention eventual consistency challenges
4. Give concrete examples (order tracking, banking)

