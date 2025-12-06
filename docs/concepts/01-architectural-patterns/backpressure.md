# Backpressure

## What It Is

**Backpressure** is a mechanism where downstream systems signal to upstream systems to slow down when they're overwhelmed. It's a feedback loop that prevents system overload by controlling the rate of data flow.

---

## The Analogy 🚰

Imagine a **water pipe system**:
- Water flows from tank → pipe → faucet
- If the faucet is partially closed, pressure builds up
- The tank needs to reduce flow or pipes burst

Without backpressure: Water keeps coming → pipes burst → system fails
With backpressure: Faucet signals "slow down" → tank reduces flow → everyone's happy

In software:
- Producer (tank) → Queue/Buffer (pipe) → Consumer (faucet)
- If consumer is slow, tell producer to slow down or stop

---

## Why It Exists

### The Problem: Unbounded Queues and OOM
```
Fast Producer (10,000 msg/sec)
        │
        ▼
   ┌─────────────┐
   │   Queue     │ ← Grows forever!
   │  (unbounded)│
   └─────────────┘
        │
        ▼
Slow Consumer (1,000 msg/sec)

Result: Queue grows → Memory exhausted → OOM crash
```

### Without Backpressure:
- Memory exhaustion (OutOfMemory)
- Increased latency (queue buildup)
- System crashes
- Data loss

### What Backpressure Solves:
- **System stability** - Prevents resource exhaustion
- **Predictable latency** - Bounded queue sizes
- **Graceful degradation** - Slow down instead of crash
- **Resource efficiency** - Producer doesn't waste work

---

## How It Works

### Backpressure Strategies:

#### 1. Drop Messages
```
Producer → [Buffer full] → DROP new messages
Use when: Sensor data, metrics (latest is most important)
```

#### 2. Buffer and Block
```
Producer → [Buffer full] → BLOCK producer until space available
Use when: Important data that cannot be lost
```

#### 3. Sample/Throttle
```
Producer → [High rate] → Only accept every Nth message
Use when: High-frequency events where sampling is acceptable
```

#### 4. Error/Reject
```
Producer → [Buffer full] → Return error to producer
Use when: Producer should retry later (HTTP 429, gRPC RESOURCE_EXHAUSTED)
```

### Visual Flow:
```
┌─────────────────────────────────────────────────────────────────┐
│                    Backpressure Flow                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐      ┌───────────────┐      ┌──────────┐         │
│   │ Producer │ ───▶ │  Bounded      │ ───▶ │ Consumer │         │
│   │          │      │  Buffer (100) │      │          │         │
│   └──────────┘      └───────────────┘      └──────────┘         │
│        ▲                   │                     │              │
│        │                   │                     │              │
│        └───────────────────┴─────────────────────┘              │
│                    "Slow down!" signal                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Examples

### Reactive Streams (Java/Kotlin)
```java
// Publisher respects subscriber demand
Flux.range(1, 1000000)
    .onBackpressureBuffer(100)      // Buffer 100, then...
    .onBackpressureDrop()           // Drop if buffer full
    .subscribe(item -> process(item));
```

### Kafka Consumer
```java
// Control how many records fetched per poll
props.put("max.poll.records", 100);  // Bounded fetching
props.put("fetch.max.bytes", 1048576); // Bounded bytes
```

### gRPC Flow Control
```java
// Server signals when ready for more
responseObserver.setOnReadyHandler(() -> {
    while (responseObserver.isReady()) {
        responseObserver.onNext(generateData());
    }
});
```

---

## When to Use It

### ✅ Essential For:
- **Stream processing** - Kafka, Flink, Spark Streaming
- **Microservices** - Inter-service communication
- **Real-time systems** - Websockets, event streams
- **Message queues** - RabbitMQ, SQS, Kafka
- **API rate limiting** - Protect backends from overload

### ❌ Less Relevant When:
- Request-response with immediate processing
- Small, predictable workloads
- Already using rate limiting at ingress

---

## How to Use It Effectively

### Best Practices:

1. **Bound everything** - Queues, buffers, thread pools
   ```java
   new ArrayBlockingQueue<>(1000);  // Not LinkedBlockingQueue()
   ```

2. **Monitor queue depths** - Alert before full
   ```
   Alert: queue_depth > 80% capacity
   ```

3. **Choose strategy per use case**
   - Financial data: Block, never drop
   - Metrics: Sample, latest is fine
   - Logs: Buffer then drop oldest

4. **Propagate backpressure** - Don't absorb in one layer
   ```
   API → Queue → Worker → DB
          ↑        ↑      ↑
   All layers should propagate "slow down"
   ```

5. **Test under load** - Simulate slow consumers

---

## Real-World Examples

| System | Backpressure Mechanism |
|--------|------------------------|
| **Kafka** | Consumer controls fetch rate |
| **RabbitMQ** | Prefetch limits, memory alarms |
| **gRPC** | Flow control per stream |
| **Reactive Streams** | Subscriber requests items |
| **TCP** | Window-based flow control |

---

## Interview Tips

When discussing Backpressure:
1. Start with the unbounded queue problem
2. Use the water pipe analogy
3. Explain different strategies (drop, block, sample)
4. Mention real implementations (Reactive Streams, Kafka)

