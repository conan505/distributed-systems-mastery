# Transactional Outbox Pattern

## What It Is

The **Transactional Outbox Pattern** ensures reliable message publishing in distributed systems by storing events in an "outbox" table within the same database transaction as the business data. A separate process then reads and publishes these events to a message broker.

---

## The Analogy 📬

Think of a **corporate mailroom**:
- You write a letter (business operation)
- You put it in your department's outbox tray (same transaction)
- The mailroom picks it up and sends it (separate process)
- If the mailroom is down, letters wait safely in the tray
- Letters are never lost or sent without being written

---

## Why It Exists

### The Dual Write Problem:
```
Traditional Approach (DANGEROUS):
1. Save order to database     ✅ Success
2. Publish "OrderCreated" to Kafka  ❌ Fails (network issue)

Result: Order exists but event never published!
         Downstream services never know about it.
```

You can't atomically:
- Write to database AND
- Publish to message broker

These are two different systems with no shared transaction.

### What Transactional Outbox Solves:
- **Atomicity** - Data and event saved in single transaction
- **Reliability** - Events are never lost
- **Eventual delivery** - Events will be published (eventually)
- **Ordering** - Events published in sequence

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                   Transactional Outbox Flow                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐                                               │
│   │   Service   │                                               │
│   └──────┬──────┘                                               │
│          │                                                       │
│          ▼  BEGIN TRANSACTION                                    │
│   ┌─────────────────────────────────────┐                       │
│   │           Database                   │                       │
│   │  ┌───────────────┬───────────────┐  │                       │
│   │  │  Orders Table │  Outbox Table │  │                       │
│   │  │               │               │  │                       │
│   │  │ INSERT order  │ INSERT event  │  │                       │
│   │  └───────────────┴───────────────┘  │                       │
│   └─────────────────────────────────────┘                       │
│          │  COMMIT                                               │
│          ▼                                                       │
│   ┌─────────────┐        ┌─────────────┐        ┌────────────┐  │
│   │  Message    │ poll   │   Outbox    │  read  │   Outbox   │  │
│   │   Relay     │◀───────│   Table     │───────▶│   Table    │  │
│   └──────┬──────┘        └─────────────┘        └────────────┘  │
│          │                                                       │
│          ▼                                                       │
│   ┌─────────────┐                                               │
│   │   Kafka /   │                                               │
│   │   RabbitMQ  │                                               │
│   └─────────────┘                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Outbox Table:
```sql
CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(255),    -- "Order", "Payment"
    aggregate_id VARCHAR(255),      -- order_123
    event_type VARCHAR(255),        -- "OrderCreated"
    payload JSONB,                  -- event data
    created_at TIMESTAMP,
    published_at TIMESTAMP NULL     -- NULL = not yet published
);
```

### Two Publishing Approaches:

#### 1. Polling Publisher
```java
// Every 100ms, check for unpublished events
@Scheduled(fixedDelay = 100)
public void publishEvents() {
    List<OutboxEvent> events = outboxRepo.findUnpublished();
    for (OutboxEvent event : events) {
        kafka.publish(event);
        outboxRepo.markPublished(event.getId());
    }
}
```

#### 2. CDC (Change Data Capture) - Debezium
- Debezium watches database transaction log
- Automatically streams outbox changes to Kafka
- No polling, lower latency

---

## When to Use It

### ✅ Good Fit:
- **Event-driven microservices** - Reliable event publishing
- **Financial systems** - Cannot lose transaction events
- **Order processing** - Downstream services need order events
- **Any dual-write scenario** - DB + message broker

### ❌ Avoid When:
- Single service, single database (no need)
- Eventual consistency is unacceptable
- Very high throughput (CDC approach recommended)
- Simple request-response patterns

---

## How to Use It Effectively

### Best Practices:

1. **Keep outbox rows small** - Only essential data
2. **Clean up old entries** - Archive or delete after publishing
3. **Idempotent consumers** - Events may be published multiple times
4. **Order by created_at** - Maintain event sequence
5. **Use CDC for scale** - Debezium for high-throughput systems

### Common Pitfalls:
- Forgetting to handle publisher failures
- Not cleaning up the outbox table
- Publishing before marking as published (duplicates)

---

## Real-World Examples

| Company | Use Case |
|---------|----------|
| **Uber** | Trip events reliably published |
| **Stripe** | Payment webhooks never lost |
| **Netflix** | Content catalog changes |
| **Shopify** | Order events to fulfillment |

---

## Implementation

### Transactional Outbox with Polling Publisher

```python
import uuid
from datetime import datetime
from typing import Optional
from dataclasses import dataclass
import json

@dataclass
class OutboxEvent:
    id: str
    aggregate_type: str
    aggregate_id: str
    event_type: str
    payload: dict
    created_at: datetime
    published_at: Optional[datetime] = None

class OutboxRepository:
    """Repository for outbox table operations."""

    def __init__(self, db_connection):
        self.db = db_connection

    def save(self, event: OutboxEvent):
        self.db.execute("""
            INSERT INTO outbox (id, aggregate_type, aggregate_id,
                              event_type, payload, created_at)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, (event.id, event.aggregate_type, event.aggregate_id,
              event.event_type, json.dumps(event.payload), event.created_at))

    def find_unpublished(self, limit: int = 100) -> list[OutboxEvent]:
        rows = self.db.execute("""
            SELECT * FROM outbox
            WHERE published_at IS NULL
            ORDER BY created_at
            LIMIT %s FOR UPDATE SKIP LOCKED
        """, (limit,))
        return [self._to_event(row) for row in rows]

    def mark_published(self, event_id: str):
        self.db.execute("""
            UPDATE outbox SET published_at = NOW() WHERE id = %s
        """, (event_id,))

class OrderService:
    """Example service using outbox pattern."""

    def __init__(self, db, outbox_repo):
        self.db = db
        self.outbox = outbox_repo

    def create_order(self, order_data: dict) -> str:
        order_id = str(uuid.uuid4())

        # Single transaction for data + event
        with self.db.transaction():
            # 1. Save business data
            self.db.execute("""
                INSERT INTO orders (id, customer_id, total, status)
                VALUES (%s, %s, %s, 'CREATED')
            """, (order_id, order_data['customer_id'], order_data['total']))

            # 2. Save event to outbox (same transaction)
            event = OutboxEvent(
                id=str(uuid.uuid4()),
                aggregate_type='Order',
                aggregate_id=order_id,
                event_type='OrderCreated',
                payload={'order_id': order_id, **order_data},
                created_at=datetime.utcnow()
            )
            self.outbox.save(event)

        return order_id

class OutboxPublisher:
    """Background worker that publishes outbox events."""

    def __init__(self, outbox_repo, kafka_producer):
        self.outbox = outbox_repo
        self.kafka = kafka_producer

    def poll_and_publish(self):
        """Called periodically (e.g., every 100ms)."""
        events = self.outbox.find_unpublished()

        for event in events:
            try:
                # Publish to Kafka
                self.kafka.send(
                    topic=f"{event.aggregate_type.lower()}-events",
                    key=event.aggregate_id,
                    value=json.dumps(event.payload)
                )

                # Mark as published
                self.outbox.mark_published(event.id)

            except Exception as e:
                # Will retry on next poll
                print(f"Failed to publish {event.id}: {e}")
                break  # Maintain ordering
```

---

## Interview Tips

When discussing Transactional Outbox:
1. Start with the dual-write problem (DB + broker not atomic)
2. Explain why distributed transactions don't work here
3. Walk through the outbox table structure
4. Mention CDC (Debezium) for high-throughput systems
5. Emphasize idempotent consumers (events may republish)

