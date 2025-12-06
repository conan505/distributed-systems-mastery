# Thread Pools

## What It Is

A **Thread Pool** is a collection of pre-created worker threads that execute tasks from a queue, avoiding the overhead of creating and destroying threads for each task.

---

## The Analogy 🏊

Think of a **restaurant kitchen**:
- **Without pool**: Hire a new chef for each order, fire after cooking
- **With pool**: Keep 5 chefs on staff, they handle orders as they come

Thread pools maintain ready workers instead of creating/destroying per task.

---

## Why They Exist

### The Problem: Thread Creation Overhead
```
Creating a thread:
- Allocate stack memory (~1MB)
- OS kernel call
- Initialize thread structures
- Time: ~1ms

For 10,000 requests:
- Create/destroy: 10,000 × 1ms = 10 seconds overhead!
- Thread pool: Near zero overhead (threads reused)
```

### What Thread Pools Solve:
- **Reduced overhead** - Reuse threads
- **Resource control** - Limit concurrent threads
- **Improved throughput** - Tasks queued efficiently
- **Stability** - Prevent thread explosion

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    Thread Pool Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Task Queue                            │   │
│   │  [Task1] [Task2] [Task3] [Task4] [Task5] ...            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Worker Threads                         │   │
│   │                                                          │   │
│   │   [Worker 1]  [Worker 2]  [Worker 3]  [Worker 4]        │   │
│   │       │           │           │           │              │   │
│   │       ▼           ▼           ▼           ▼              │   │
│   │   Executing   Executing    Idle       Executing         │   │
│   │    Task1       Task2      (waiting)    Task3            │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Workers continuously:                                          │
│   1. Take task from queue                                       │
│   2. Execute task                                               │
│   3. Return to queue for next task                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pool Sizing

### CPU-Bound Tasks:
```
Optimal threads = Number of CPU cores

Why: More threads = context switching overhead
     Fewer threads = CPU underutilized

Example: 8-core machine → 8 threads
```

### I/O-Bound Tasks:
```
Optimal threads = CPU cores × (1 + Wait time / Compute time)

Example: 8 cores, 90% waiting for I/O
Threads = 8 × (1 + 9) = 80 threads

Why: While one thread waits for I/O, others can use CPU
```

### Mixed Workloads:
```
Use separate pools:
- CPU pool: cores threads
- I/O pool: larger pool

Prevents I/O tasks from starving CPU tasks
```

---

## Queue Strategies

### 1. Unbounded Queue
```
✅ Never rejects tasks
❌ Can cause OOM if tasks pile up
Use: When task rate is predictable
```

### 2. Bounded Queue
```
✅ Memory bounded
❌ Need rejection policy
Use: Production systems
```

### 3. Synchronous Queue
```
No queue - direct handoff to worker
✅ Low latency
❌ Rejects if no worker available
Use: When you want to scale workers dynamically
```

---

## Rejection Policies

When queue is full and all workers busy:

| Policy | Behavior |
|--------|----------|
| **Abort** | Throw exception |
| **Discard** | Silently drop task |
| **Discard Oldest** | Drop oldest queued task |
| **Caller Runs** | Execute in submitting thread |

---

## Implementation Examples

### Python
```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(task, arg) for arg in args]
    results = [f.result() for f in futures]
```

### Java
```java
ExecutorService pool = Executors.newFixedThreadPool(4);

// Or with more control:
ExecutorService pool = new ThreadPoolExecutor(
    4,                      // core pool size
    8,                      // max pool size
    60, TimeUnit.SECONDS,   // keep-alive time
    new LinkedBlockingQueue<>(100)  // bounded queue
);
```

### Go
```go
// Go uses goroutines + worker pool pattern
jobs := make(chan Job, 100)

// Start workers
for i := 0; i < 4; i++ {
    go worker(jobs)
}

// Submit jobs
for _, job := range jobList {
    jobs <- job
}
```

---

## Advanced Patterns

### Work Stealing
```
Each worker has own queue
Idle workers "steal" from busy workers' queues

✅ Better load balancing
✅ Cache locality
Used by: Java ForkJoinPool, Go runtime
```

### Dynamic Sizing
```
Core threads: Always running
Max threads: Created under load
Keep-alive: Idle threads terminated after timeout

Adapts to workload automatically
```

---

## Common Pitfalls

```
1. Pool too small → Tasks queue up, high latency
2. Pool too large → Context switching, memory waste
3. Unbounded queue → OOM under load
4. Blocking in pool → Deadlock risk
5. Not shutting down → Resource leak
```

---

## When to Use

### ✅ Good Fit:
- Web servers (request handling)
- Batch processing
- Parallel computation
- Background tasks

### ❌ Consider Alternatives:
- Very short tasks (overhead dominates) → Batch tasks
- Async I/O heavy → Event loop (asyncio, Node.js)
- CPU-bound parallelism → Process pool (avoid GIL)

---

## Interview Tips

When discussing Thread Pools:
1. Explain the overhead reduction benefit
2. Know sizing formulas (CPU vs I/O bound)
3. Discuss queue strategies and rejection policies
4. Mention work stealing for advanced pools
5. Give language-specific examples

