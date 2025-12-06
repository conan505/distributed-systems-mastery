# SEDA (Staged Event-Driven Architecture)

## What It Is

**SEDA** is an architecture that decomposes complex applications into stages connected by queues. Each stage has its own thread pool and can be independently tuned for load management.

---

## The Analogy 🏭

Think of a **car assembly line**:
- **Stage 1**: Chassis assembly (10 workers)
- **Stage 2**: Engine installation (5 workers)
- **Stage 3**: Painting (8 workers)
- **Stage 4**: Quality check (3 workers)

Each stage:
- Has its own workers (thread pool)
- Has a buffer of work (queue)
- Can be scaled independently
- Doesn't block other stages

---

## Why It Exists

### The Problem: Thread-per-Request
```
Traditional model:
1000 concurrent requests = 1000 threads
Each thread: 1MB stack = 1GB memory!

Problems:
- Memory explosion
- Context switching overhead
- No load isolation
- Cascade failures
```

### What SEDA Solves:
- **Bounded resources** - Fixed thread pools per stage
- **Load shedding** - Queues can drop/reject
- **Graceful degradation** - Stages fail independently
- **Self-tuning** - Adjust workers based on load

---

## How It Works

### Architecture:
```
┌─────────────────────────────────────────────────────────────────┐
│                      SEDA Architecture                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Request                                                        │
│      │                                                           │
│      ▼                                                           │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│   │  Stage 1 │───▶│  Stage 2 │───▶│  Stage 3 │───▶│  Stage 4 │ │
│   │ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │ │
│   │ │Queue │ │    │ │Queue │ │    │ │Queue │ │    │ │Queue │ │ │
│   │ └──────┘ │    │ └──────┘ │    │ └──────┘ │    │ └──────┘ │ │
│   │ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │ │
│   │ │Thread│ │    │ │Thread│ │    │ │Thread│ │    │ │Thread│ │ │
│   │ │ Pool │ │    │ │ Pool │ │    │ │ Pool │ │    │ │ Pool │ │ │
│   │ └──────┘ │    │ └──────┘ │    │ └──────┘ │    │ └──────┘ │ │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘ │
│                                                                  │
│   Controller monitors queues, adjusts thread pool sizes         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Components:

**Stage:**
- Event queue (incoming work)
- Event handler (processing logic)
- Thread pool (workers)
- Controller (admission control)

**Queue:**
- Buffers between stages
- Bounded (finite capacity)
- Enables backpressure

---

## Load Management

### 1. Admission Control
```
If queue is full:
- Reject new requests (503 Service Unavailable)
- Apply backpressure to upstream
- Shed load gracefully
```

### 2. Dynamic Resource Allocation
```
Controller monitors:
- Queue length
- Processing time
- Throughput

Adjusts:
- Thread pool size
- Batching size
- Timeout values
```

### 3. Batching
```
Instead of processing one event:
- Wait for N events or timeout
- Process batch together
- Amortize overhead
```

---

## SEDA vs Other Architectures

| Aspect | Thread-per-Request | Event Loop | SEDA |
|--------|-------------------|------------|------|
| **Threads** | Many (1 per request) | Single | Few (per stage) |
| **Blocking** | Allowed | Not allowed | Allowed in stage |
| **Isolation** | None | None | Per-stage |
| **Load shedding** | Hard | Hard | Built-in |
| **Tuning** | Global | Global | Per-stage |

---

## When to Use

### ✅ Good Fit:
- **High-concurrency servers** - Web servers, proxies
- **Pipeline processing** - ETL, media processing
- **Mixed workloads** - CPU + I/O bound stages
- **Graceful degradation needed** - Load shedding

### ❌ Avoid When:
- Simple request-response
- Low concurrency
- Latency-critical (queuing adds delay)
- Simple stateless services

---

## Real-World Examples

| System | Usage |
|--------|-------|
| **Apache Cassandra** | Request processing pipeline |
| **NGINX** | Event-driven stages |
| **Netty** | Boss/Worker thread groups |
| **Disruptor** | LMAX's high-performance variant |

---

## Implementation Example

```java
class Stage {
    private BlockingQueue<Event> queue;
    private ExecutorService threadPool;
    private EventHandler handler;
    private Stage nextStage;
    
    void submit(Event event) {
        if (!queue.offer(event)) {
            // Queue full - load shedding
            handleOverload(event);
        }
    }
    
    void process() {
        while (true) {
            Event event = queue.take();
            Result result = handler.handle(event);
            if (nextStage != null) {
                nextStage.submit(result);
            }
        }
    }
}
```

---

## Key Metrics to Monitor

| Metric | Purpose |
|--------|---------|
| **Queue length** | Backlog indicator |
| **Queue wait time** | Latency component |
| **Thread utilization** | Sizing guidance |
| **Drop rate** | Load shedding frequency |
| **Throughput per stage** | Bottleneck identification |

---

## Interview Tips

When discussing SEDA:
1. Use the assembly line analogy
2. Explain stage isolation benefits
3. Discuss load shedding mechanism
4. Compare with thread-per-request
5. Mention real examples (Cassandra)

