# Message Queues

## What It Is

A **Message Queue** is middleware that enables asynchronous communication between services by storing messages until they're processed by consumers.

---

## The Analogy 📬

Think of a **post office mailbox**:
- Sender drops letter (message) in mailbox
- Sender doesn't wait for recipient to read it
- Postal service stores and delivers
- Recipient reads when available
- If recipient is busy, letters queue up

---

## Why Message Queues Matter

### The Problem: Synchronous Coupling
```
Without queue:
Order Service ──sync call──▶ Inventory Service
                             (if down, order fails!)

With queue:
Order Service ──publish──▶ [Queue] ──consume──▶ Inventory Service
                          (order succeeds even if Inventory down)
```

### What Queues Solve:
- **Decoupling** - Services don't need to know about each other
- **Reliability** - Messages persist if consumers are down
- **Scalability** - Add consumers to handle load
- **Load leveling** - Smooth out traffic spikes
- **Async processing** - Don't block on slow operations

---

## Messaging Patterns

### 1. Point-to-Point (Queue)
```
┌─────────────────────────────────────────────────────────────────┐
│                    Point-to-Point Queue                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Producer ──▶ [Queue] ──▶ Consumer                             │
│                   │                                              │
│   Each message delivered to ONE consumer                        │
│   Multiple consumers = competing consumers pattern              │
│                                                                  │
│   Producer ─┬▶ [Queue] ──▶ Consumer 1                           │
│   Producer ─┘     │       Consumer 2  (load distributed)        │
│                   └─────▶ Consumer 3                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Publish-Subscribe (Topic)
```
┌─────────────────────────────────────────────────────────────────┐
│                    Publish-Subscribe                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Publisher ──▶ [Topic] ──┬──▶ Subscriber 1                     │
│                           ├──▶ Subscriber 2                     │
│                           └──▶ Subscriber 3                     │
│                                                                  │
│   Each message delivered to ALL subscribers                     │
│   Great for event broadcasting                                   │
│                                                                  │
│   Example: "Order Created" event                                 │
│   - Inventory service: Reserve stock                            │
│   - Email service: Send confirmation                            │
│   - Analytics service: Log event                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Message Delivery Guarantees

| Guarantee | Description | Trade-off |
|-----------|-------------|-----------|
| **At-most-once** | Fire and forget | Fast, may lose messages |
| **At-least-once** | Retry until acked | May have duplicates |
| **Exactly-once** | Delivered once only | Complex, slower |

### At-Least-Once Example:
```
1. Consumer receives message
2. Consumer processes (but crashes before ack)
3. Message redelivered
4. Consumer processes again (duplicate!)

Solution: Make consumers idempotent
```

---

## Message Ordering

### FIFO Guarantee:
```
Producer sends: A, B, C
Consumer receives: A, B, C (guaranteed order)

Kafka: Order within partition
SQS FIFO: Order within message group
```

### No Ordering Guarantee:
```
Producer sends: A, B, C
Consumer receives: B, A, C (any order)

Higher throughput, parallel processing
```

---

## Key Concepts

### Dead Letter Queue (DLQ):
```
Messages that fail processing repeatedly:
1. Original queue → Consumer fails
2. Retry 3 times
3. Still failing → Move to DLQ
4. Human/automated investigation

Prevents poison messages blocking queue
```

### Message TTL:
```
Messages expire after time period
Useful for time-sensitive operations
(e.g., flash sale offers)
```

### Backpressure:
```
When consumer can't keep up:
1. Queue fills up
2. Producer slows down or drops messages
3. Prevents system overload

See: backpressure.md
```

---

## When to Use

### ✅ Good Fit:
- **Async processing** - Email, notifications
- **Work distribution** - Task queues
- **Event-driven architecture** - Microservices
- **Traffic spikes** - Load leveling
- **Reliability** - At-least-once needed

### ❌ Avoid When:
- Need synchronous response
- Simple, low-volume communication
- Adds unnecessary complexity

---

## Real-World Systems

| System | Type | Use Case |
|--------|------|----------|
| **Apache Kafka** | Log-based | High-throughput streaming |
| **RabbitMQ** | Traditional queue | Complex routing |
| **Amazon SQS** | Cloud queue | Serverless, simple |
| **Redis Streams** | In-memory | Low-latency |
| **Apache Pulsar** | Unified | Pub/sub + queuing |

---

## Kafka vs RabbitMQ

| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| **Model** | Append-only log | Message broker |
| **Ordering** | Per-partition | Per-queue |
| **Replay** | Yes (retained logs) | No (consumed = gone) |
| **Throughput** | Very high (100K+ msg/s) | Moderate |
| **Routing** | Simple (topic/partition) | Complex (exchanges) |
| **Use case** | Event streaming, logs | Task queues, RPC |

---

## Interview Tips

When discussing Message Queues:
1. Explain decoupling and reliability benefits
2. Know point-to-point vs pub/sub
3. Discuss delivery guarantees and idempotency
4. Compare Kafka vs RabbitMQ use cases
5. Mention DLQ for error handling

