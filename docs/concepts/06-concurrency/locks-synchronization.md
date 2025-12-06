# Locks and Synchronization

## What They Are

**Locks** are synchronization primitives that ensure only one thread can access a shared resource at a time. **Synchronization** is the coordination of concurrent threads to ensure correct execution.

---

## The Analogy 🚪

Think of a **bathroom with a lock**:
- **Mutex**: Single-occupancy bathroom lock
- **Semaphore**: Bathroom with N stalls
- **Read-Write Lock**: Library (many readers, one writer)

---

## Why They Exist

### The Problem: Concurrent Access
```python
# Without synchronization
balance = 100

# Thread A                    # Thread B
temp = balance  # 100         temp = balance  # 100
temp = temp + 50              temp = temp - 30
balance = temp  # 150         balance = temp  # 70

# Expected: 120, Got: 70 or 150 (race condition!)
```

### What Locks Solve:
- **Mutual exclusion** - Only one thread in critical section
- **Atomicity** - Operations complete without interruption
- **Visibility** - Changes visible to other threads
- **Ordering** - Establish happens-before relationships

---

## Types of Locks

### 1. Mutex (Mutual Exclusion)
```
┌─────────────────────────────────────────────────────────────────┐
│                         Mutex                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Only ONE thread can hold the lock at a time                   │
│                                                                  │
│   Thread A: acquire() ──▶ [critical section] ──▶ release()     │
│   Thread B: acquire() ──▶ BLOCKED...                            │
│                          (waits for A to release)               │
│                                                                  │
│   Use: Protecting shared mutable state                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
import threading

lock = threading.Lock()

def transfer(from_acc, to_acc, amount):
    with lock:  # Acquire lock
        from_acc.balance -= amount
        to_acc.balance += amount
    # Lock automatically released
```

### 2. Read-Write Lock (RWLock)
```
┌─────────────────────────────────────────────────────────────────┐
│                     Read-Write Lock                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Multiple readers OR one writer (not both)                     │
│                                                                  │
│   Readers: [R1] [R2] [R3] ──▶ All can read simultaneously      │
│   Writer:  [W1] ──▶ Exclusive access, blocks all readers       │
│                                                                  │
│   Use: Read-heavy workloads (caches, configs)                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Semaphore
```
┌─────────────────────────────────────────────────────────────────┐
│                       Semaphore                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Allows N concurrent accesses (counting semaphore)             │
│                                                                  │
│   Semaphore(3):                                                 │
│   [T1] [T2] [T3] ──▶ All proceed                               │
│   [T4] ──▶ BLOCKED (waits for one to release)                  │
│                                                                  │
│   Use: Connection pools, rate limiting                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Spinlock
```
┌─────────────────────────────────────────────────────────────────┐
│                       Spinlock                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Thread busy-waits (spins) instead of sleeping                 │
│                                                                  │
│   while (!lock.try_acquire()):                                  │
│       pass  # Keep trying (burns CPU)                           │
│                                                                  │
│   ✅ No context switch overhead                                  │
│   ❌ Wastes CPU if wait is long                                  │
│   Use: Very short critical sections, kernel code                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Lock-Free Alternatives

### Compare-And-Swap (CAS)
```python
# Atomic operation
def compare_and_swap(location, expected, new_value):
    if location.value == expected:
        location.value = new_value
        return True
    return False

# Lock-free increment
while True:
    old = counter.load()
    if counter.cas(old, old + 1):
        break  # Success!
```

### Atomic Variables
```python
from atomics import AtomicInt

counter = AtomicInt(0)
counter.fetch_add(1)  # Atomic increment
counter.fetch_sub(1)  # Atomic decrement
```

---

## Common Problems

### 1. Deadlock
```
Thread A: holds Lock1, waiting for Lock2
Thread B: holds Lock2, waiting for Lock1
→ Both wait forever!

Prevention:
- Lock ordering (always acquire in same order)
- Timeout on lock acquisition
- Deadlock detection
```

### 2. Livelock
```
Thread A: "After you!"
Thread B: "No, after you!"
→ Both keep yielding, no progress

Solution: Add randomness to retry logic
```

### 3. Priority Inversion
```
High-priority thread blocked by low-priority thread holding lock
Medium-priority thread preempts low-priority
→ High-priority waits for medium!

Solution: Priority inheritance
```

### 4. Lock Contention
```
Many threads competing for same lock
→ Serialized execution, poor performance

Solutions:
- Fine-grained locking
- Lock-free algorithms
- Reduce critical section size
```

---

## Best Practices

```
1. Keep critical sections short
2. Avoid nested locks (deadlock risk)
3. Use lock ordering if multiple locks needed
4. Prefer higher-level abstractions (queues, actors)
5. Consider lock-free for hot paths
6. Use read-write locks for read-heavy workloads
7. Profile before optimizing
```

---

## Language-Specific Primitives

| Language | Mutex | RWLock | Semaphore |
|----------|-------|--------|-----------|
| **Python** | `threading.Lock` | `threading.RLock` | `threading.Semaphore` |
| **Java** | `synchronized` | `ReentrantReadWriteLock` | `Semaphore` |
| **Go** | `sync.Mutex` | `sync.RWMutex` | `chan` (buffered) |
| **Rust** | `std::sync::Mutex` | `RwLock` | `Semaphore` (tokio) |
| **C++** | `std::mutex` | `std::shared_mutex` | `std::counting_semaphore` |

---

## Interview Tips

When discussing Locks:
1. Explain mutex vs semaphore vs RWLock
2. Know deadlock conditions and prevention
3. Discuss lock-free alternatives (CAS)
4. Mention performance trade-offs
5. Give examples of when to use each type

