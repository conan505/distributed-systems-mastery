# Bulkhead Pattern

## What It Is

The **Bulkhead Pattern** isolates elements of an application into pools so that if one fails, the others continue to function. It prevents a failure in one part of the system from cascading to other parts.

---

## The Analogy 🚢

The name comes from **ship bulkheads** - watertight compartments in a ship's hull:
- If one compartment floods, the watertight doors prevent water from spreading
- The ship stays afloat even with damage to one section
- The Titanic had bulkheads, but they weren't tall enough (lessons learned!)

In software:
- If one service/component fails, it shouldn't sink the entire application
- Resources are partitioned so failures are contained

---

## Why It Exists

### The Problem: Cascading Failures
```
Normal:     ServiceA (10 threads) ──▶ Database
                                         │
Slow DB:    ServiceA (10 threads) ──▶ Database (slow)
                 │                       │
                 └── All threads stuck waiting!
                 └── New requests fail
                 └── Other features break too
```

Without bulkheads, a slow dependency can consume ALL resources:
- Thread pools exhausted
- Memory filled with pending requests
- Entire application becomes unresponsive

### What Bulkhead Solves:
- **Resource isolation** - Each dependency gets its own resource pool
- **Failure containment** - Problems don't spread
- **Graceful degradation** - Healthy parts keep working
- **Predictable behavior** - Failures affect only related functions

---

## How It Works

### Thread Pool Isolation Example:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Without Bulkhead                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────────────────────────────────┐                  │
│   │           Shared Thread Pool (20)         │                  │
│   │  ┌────┬────┬────┬────┬────┬────┬────┐   │                  │
│   │  │ DB │ DB │ DB │ DB │API │API │ .. │   │                  │
│   │  └────┴────┴────┴────┴────┴────┴────┘   │                  │
│   └──────────────────────────────────────────┘                  │
│   If DB is slow → ALL threads blocked → API calls fail too!     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       With Bulkhead                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────┐    ┌─────────────────┐                    │
│   │ DB Pool (10)    │    │ API Pool (10)   │                    │
│   │ ┌──┬──┬──┬──┐   │    │ ┌──┬──┬──┬──┐   │                    │
│   │ │DB│DB│DB│DB│   │    │ │✓ │✓ │✓ │✓ │   │                    │
│   │ └──┴──┴──┴──┘   │    │ └──┴──┴──┴──┘   │                    │
│   └─────────────────┘    └─────────────────┘                    │
│   DB slow? Only DB calls affected. API calls work fine!         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Types of Bulkheads

### 1. Thread Pool Bulkhead
Separate thread pools for different operations:
```java
// Resilience4j example
ThreadPoolBulkhead dbBulkhead = ThreadPoolBulkhead.of("db", 
    ThreadPoolBulkheadConfig.custom()
        .maxThreadPoolSize(10)
        .coreThreadPoolSize(5)
        .build());

ThreadPoolBulkhead apiBulkhead = ThreadPoolBulkhead.of("api",
    ThreadPoolBulkheadConfig.custom()
        .maxThreadPoolSize(20)
        .build());
```

### 2. Semaphore Bulkhead
Limit concurrent calls without dedicated threads:
```java
SemaphoreBulkhead bulkhead = SemaphoreBulkhead.of("name",
    SemaphoreBulkheadConfig.custom()
        .maxConcurrentCalls(25)
        .maxWaitDuration(Duration.ofMillis(500))
        .build());
```

### 3. Connection Pool Bulkhead
Separate database connection pools:
```yaml
# Separate pools for critical vs non-critical queries
datasource:
  critical:
    maximum-pool-size: 20
  analytics:
    maximum-pool-size: 5
```

---

## When to Use It

### ✅ Good Fit:
- **Multiple external dependencies** - APIs, databases, caches
- **Mixed criticality** - Some features are more important
- **Unreliable dependencies** - Third-party services that may fail
- **Microservices** - Isolate inter-service communication

### ❌ Avoid When:
- Simple applications with single dependency
- When resource overhead is unacceptable
- Already using other isolation (separate processes/containers)

---

## How to Use It Effectively

### Best Practices:

1. **Size pools appropriately**
   - Too small: Requests rejected unnecessarily
   - Too large: Defeats the purpose of isolation

2. **Combine with Circuit Breaker**
   - Bulkhead limits concurrent calls
   - Circuit breaker stops calls to failing services

3. **Monitor each bulkhead separately**
   - Track queue depths, rejection rates
   - Alert on high utilization

4. **Prioritize critical paths**
   - Core checkout: larger pool
   - Analytics: smaller pool

5. **Set appropriate timeouts**
   - Don't let threads wait forever
   - Fail fast, free resources

---

## Real-World Examples

| Company | Use Case |
|---------|----------|
| **Netflix** | Hystrix library with bulkhead per command |
| **Amazon** | Separate pools for different AWS services |
| **Banks** | Critical transactions vs reporting queries |
| **E-commerce** | Checkout vs recommendation engine |

---

## Interview Tips

When discussing Bulkhead in interviews:
1. Use the ship analogy - it's memorable and accurate
2. Explain cascading failure scenario first
3. Distinguish from Circuit Breaker (they complement each other)
4. Mention real implementation (Resilience4j, Hystrix)

