# LMAX Disruptor

## What It Is

The **LMAX Disruptor** is a high-performance, lock-free data structure for inter-thread messaging. It's a ring buffer that enables millions of operations per second with predictable latency.

---

## The Analogy 🎠

Think of a **carousel/merry-go-round**:
- Fixed number of seats (ring buffer slots)
- Producer puts items on seats as they pass
- Consumers take items from seats
- No stopping, just continuous rotation
- Everyone knows their position

---

## Why It Exists

### The Problem: Queue Contention
```
Traditional queues (ArrayBlockingQueue):
- Lock contention on head/tail
- False sharing between threads
- Memory allocation/GC for nodes
- Unpredictable latency spikes

Result: ~5 million ops/sec max
```

### What Disruptor Solves:
- **No locks** - CAS-based coordination
- **No allocation** - Pre-allocated ring buffer
- **No false sharing** - Cache-line padding
- **Predictable latency** - No GC pauses

Result: **100+ million ops/sec**

---

## How It Works

### Ring Buffer:
```
┌─────────────────────────────────────────────────────────────────┐
│                     Disruptor Ring Buffer                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    Sequence: 15                                  │
│                         │                                        │
│              ┌──────────┼──────────┐                            │
│          ┌───▼───┐  ┌───────┐  ┌───────┐                        │
│          │ Slot  │  │ Slot  │  │ Slot  │                        │
│          │  15   │  │  16   │  │  17   │                        │
│          │(next) │  │(empty)│  │(empty)│                        │
│          └───────┘  └───────┘  └───────┘                        │
│              ▲                      │                            │
│              │    Ring Buffer       │                            │
│          ┌───────┐  ┌───────┐  ┌───────┐                        │
│          │ Slot  │  │ Slot  │  │ Slot  │                        │
│          │  14   │  │  13   │  │  12   │                        │
│          │(proc) │  │(done) │  │(done) │                        │
│          └───────┘  └───────┘  └───────┘                        │
│              ▲                                                   │
│              │                                                   │
│         Consumer at: 14                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Position = sequence % buffer_size
Buffer size = power of 2 (fast modulo via bitwise AND)
```

### Key Components:

**Sequence:**
- Atomic counter (64-bit)
- Tracks producer/consumer positions
- Padded to avoid false sharing

**RingBuffer:**
- Pre-allocated array of events
- Events reused, not reallocated
- Size is power of 2

**SequenceBarrier:**
- Coordinates between producers/consumers
- Wait strategies (busy spin, yield, block)

---

## Producer-Consumer Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Single Producer Flow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Producer:                                                      │
│   1. Claim next sequence (CAS)                                  │
│   2. Write data to slot                                          │
│   3. Publish (update sequence)                                   │
│                                                                  │
│   Consumer:                                                      │
│   1. Wait for sequence to be available                          │
│   2. Read data from slot                                         │
│   3. Update consumer sequence                                    │
│                                                                  │
│   Producer       ┌─────────────────┐     Consumer               │
│   Seq: 100  ───▶ │   Ring Buffer   │ ───▶ Seq: 98              │
│                  │   Size: 64      │                            │
│                  └─────────────────┘                            │
│                                                                  │
│   Slots 99, 100 are published, consumer can read                │
│   Slots 101+ are being written or empty                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Wait Strategies

| Strategy | CPU | Latency | Use Case |
|----------|-----|---------|----------|
| **BusySpinWait** | High | Lowest | Ultra-low latency |
| **YieldingWait** | Medium | Low | Low latency |
| **SleepingWait** | Low | Higher | Normal throughput |
| **BlockingWait** | Lowest | Highest | Infrequent events |

---

## Why It's Fast

### 1. No Locks
```java
// Traditional queue
synchronized (lock) {
    queue.add(item);  // Contention!
}

// Disruptor
long seq = ringBuffer.next();  // CAS, no lock
ringBuffer.get(seq).set(data);
ringBuffer.publish(seq);
```

### 2. Cache-Friendly
```java
// Padding to prevent false sharing
class Sequence {
    long p1, p2, p3, p4, p5, p6, p7;  // 56 bytes padding
    volatile long value;               // 8 bytes
    long p8, p9, p10, p11, p12, p13, p14;  // 56 bytes padding
}
// Total: 128 bytes (2 cache lines)
```

### 3. Pre-allocation
```java
// Events pre-created, reused
Event[] buffer = new Event[bufferSize];
for (int i = 0; i < bufferSize; i++) {
    buffer[i] = new Event();  // Only at startup
}
```

---

## When to Use

### ✅ Good Fit:
- **Low-latency systems** - Trading, gaming
- **High-throughput messaging** - Millions ops/sec
- **Event processing pipelines** - SEDA-like stages
- **Avoiding GC pauses** - Pre-allocated buffers

### ❌ Avoid When:
- Simple use cases (overkill)
- Distributed systems (single JVM only)
- Varying message sizes
- Need unbounded queue

---

## Real-World Usage

| Company | Usage |
|---------|-------|
| **LMAX** | Financial exchange (origin) |
| **Log4j2** | Async logging |
| **Apache Storm** | Internal messaging |
| **Hazelcast Jet** | Stream processing |

---

## Interview Tips

When discussing LMAX Disruptor:
1. Explain the ring buffer concept
2. Discuss why no locks (CAS operations)
3. Mention cache line padding
4. Compare throughput to BlockingQueue
5. Know when it's appropriate (low-latency)

