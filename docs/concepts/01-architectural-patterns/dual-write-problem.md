# Dual Write Problem

## What It Is

The **Dual Write Problem** occurs when a service needs to update two different systems (e.g., database and message queue) atomically, but can't guarantee both succeed or both fail.

---

## The Analogy 📝

Think of **sending a letter and updating your address book**:
- You write a letter and update your contacts
- Letter gets lost in mail
- Your address book says "sent" but recipient never got it
- Systems are now inconsistent

---

## Why It's a Problem

### The Scenario:
```
┌─────────────────────────────────────────────────────────────────┐
│                    Dual Write Problem                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Order Service needs to:                                        │
│   1. Save order to database                                     │
│   2. Publish event to message queue                             │
│                                                                  │
│   def create_order(order):                                      │
│       db.save(order)           # Step 1: DB write               │
│       queue.publish(order)     # Step 2: Queue publish          │
│                                                                  │
│   What can go wrong?                                            │
│                                                                  │
│   Failure after Step 1:                                         │
│   - Order in DB ✓                                               │
│   - Event NOT published ✗                                       │
│   - Downstream services never notified!                         │
│                                                                  │
│   Failure after Step 2 (if reversed):                           │
│   - Event published ✓                                           │
│   - Order NOT in DB ✗                                           │
│   - Downstream processes non-existent order!                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why Transactions Don't Help:
```
Database transaction:
BEGIN
  INSERT INTO orders ...
  -- Can't include Kafka publish in DB transaction!
COMMIT

Two different systems = no distributed transaction
(2PC is slow, complex, and often unavailable)
```

---

## Solutions

### 1. Transactional Outbox Pattern
```
┌─────────────────────────────────────────────────────────────────┐
│                   Transactional Outbox                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Single DB transaction:                                         │
│   BEGIN                                                          │
│     INSERT INTO orders (...)                                    │
│     INSERT INTO outbox (event_data, ...)                        │
│   COMMIT                                                         │
│                                                                  │
│   Separate process (CDC or poller):                             │
│   - Reads outbox table                                          │
│   - Publishes to message queue                                  │
│   - Marks as published                                          │
│                                                                  │
│   ✅ Atomic: Both writes in same transaction                    │
│   ✅ Reliable: Events eventually published                      │
│   See: transactional-outbox.md for details                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Event Sourcing
```
┌─────────────────────────────────────────────────────────────────┐
│                     Event Sourcing                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Events ARE the database:                                       │
│                                                                  │
│   Event Store:                                                   │
│   [OrderCreated] [ItemAdded] [OrderShipped] ...                 │
│                                                                  │
│   - Single write to event store                                 │
│   - Consumers read from event store                             │
│   - No dual write needed!                                       │
│                                                                  │
│   ✅ Eliminates the problem entirely                            │
│   ❌ Requires architectural change                               │
│   See: cqrs-event-sourcing.md for details                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Change Data Capture (CDC)
```
┌─────────────────────────────────────────────────────────────────┐
│                  Change Data Capture                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Write only to database:                                        │
│   db.save(order)                                                │
│                                                                  │
│   CDC tool (Debezium) watches DB transaction log:               │
│   - Captures INSERT/UPDATE/DELETE                               │
│   - Publishes to Kafka automatically                            │
│                                                                  │
│   ✅ Application only writes to DB                              │
│   ✅ Events derived from DB changes                             │
│   ❌ Events are DB-centric, not domain events                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Saga Pattern
```
┌─────────────────────────────────────────────────────────────────┐
│                      Saga Pattern                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Accept eventual consistency:                                   │
│                                                                  │
│   1. Publish event first (with unique ID)                       │
│   2. Consumer processes event                                   │
│   3. If processing fails, compensate                            │
│                                                                  │
│   Requires:                                                      │
│   - Idempotent consumers                                        │
│   - Compensation logic                                          │
│   - Correlation IDs                                             │
│                                                                  │
│   See: saga-pattern.md for details                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Solution Comparison

| Solution | Complexity | Consistency | Latency |
|----------|------------|-------------|---------|
| **Outbox** | Medium | Strong | Low |
| **Event Sourcing** | High | Strong | Low |
| **CDC** | Medium | Eventual | Medium |
| **Saga** | High | Eventual | Variable |

---

## Anti-Patterns

### ❌ Hope It Works
```python
def create_order(order):
    db.save(order)
    try:
        queue.publish(order)
    except:
        pass  # Hope it's fine 🤞
```

### ❌ Manual Retry Without Idempotency
```python
def create_order(order):
    db.save(order)
    retry(queue.publish, order)  # May publish duplicates!
```

### ❌ Distributed Transactions (2PC)
```
Technically correct but:
- Slow (blocking)
- Reduces availability
- Many systems don't support it
```

---

## When Each Solution Fits

| Scenario | Recommended Solution |
|----------|---------------------|
| Existing DB-centric app | Transactional Outbox |
| Greenfield project | Event Sourcing |
| Need DB change events | CDC (Debezium) |
| Complex workflows | Saga |

---

## Interview Tips

When discussing Dual Write:
1. Explain why it's a problem (no atomic cross-system writes)
2. Know the Transactional Outbox pattern well
3. Mention CDC as an alternative
4. Discuss trade-offs of each solution
5. Emphasize idempotency requirement

