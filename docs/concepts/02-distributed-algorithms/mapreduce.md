# MapReduce

## What It Is

**MapReduce** is a programming model for processing large datasets in parallel across a distributed cluster. It breaks computation into two phases: Map (transform) and Reduce (aggregate).

---

## The Analogy 📊

Think of **counting votes in an election**:
- **Map phase**: Each polling station counts their ballots by candidate
- **Shuffle**: Group all counts by candidate
- **Reduce phase**: Sum up each candidate's total votes

Parallel counting at stations, then aggregate results.

---

## Why It Exists

### The Problem: Big Data Processing
```
Process 1 PB of data:
- Single machine: 1 TB/hour = 1000 hours (41 days!)
- 1000 machines: 1 hour (with MapReduce)

Challenges:
- Data too big for one machine
- Need fault tolerance
- Coordinate parallel work
```

### What MapReduce Solves:
- **Parallelization** - Automatic distribution
- **Fault tolerance** - Retry failed tasks
- **Scalability** - Add machines linearly
- **Simplicity** - Just write map and reduce functions

---

## How It Works

### The Model:
```
┌─────────────────────────────────────────────────────────────────┐
│                    MapReduce Flow                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Input Data                                                     │
│   ┌─────┬─────┬─────┬─────┐                                     │
│   │Split│Split│Split│Split│                                     │
│   └──┬──┴──┬──┴──┬──┴──┬──┘                                     │
│      │     │     │     │                                         │
│      ▼     ▼     ▼     ▼                                         │
│   ┌─────┬─────┬─────┬─────┐                                     │
│   │ Map │ Map │ Map │ Map │  ← Transform each record            │
│   └──┬──┴──┬──┴──┬──┴──┬──┘                                     │
│      │     │     │     │                                         │
│      └─────┴──┬──┴─────┘                                         │
│               │                                                  │
│               ▼                                                  │
│         ┌─────────┐                                             │
│         │ Shuffle │  ← Group by key                             │
│         │  Sort   │                                             │
│         └────┬────┘                                             │
│              │                                                   │
│      ┌───────┼───────┐                                          │
│      ▼       ▼       ▼                                          │
│   ┌──────┬──────┬──────┐                                        │
│   │Reduce│Reduce│Reduce│  ← Aggregate per key                   │
│   └──┬───┴──┬───┴──┬───┘                                        │
│      │      │      │                                             │
│      ▼      ▼      ▼                                             │
│   Output Files                                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Word Count Example:
```python
# Input: "hello world hello"

# MAP PHASE
def map(document):
    for word in document.split():
        emit(word, 1)

# Output: [("hello", 1), ("world", 1), ("hello", 1)]

# SHUFFLE PHASE (automatic)
# Groups by key: {"hello": [1, 1], "world": [1]}

# REDUCE PHASE
def reduce(word, counts):
    emit(word, sum(counts))

# Output: [("hello", 2), ("world", 1)]
```

---

## Key Concepts

### 1. Data Locality
```
Move computation to data, not data to computation

- Data stored on HDFS across nodes
- Map tasks scheduled on nodes with data
- Minimizes network transfer
```

### 2. Fault Tolerance
```
Worker fails during map:
- Master detects via heartbeat
- Reschedules map task on another worker
- Re-executes from input split

Worker fails during reduce:
- Reschedule reduce task
- Re-fetch map outputs (stored on disk)
```

### 3. Combiners (Local Reduce)
```
Optimization: Reduce locally before shuffle

Without combiner:
  Map outputs: ("hello", 1), ("hello", 1), ("hello", 1)
  Network transfer: 3 records

With combiner:
  Local combine: ("hello", 3)
  Network transfer: 1 record
```

---

## Limitations

```
1. Disk I/O heavy
   - Map writes to disk
   - Shuffle reads/writes disk
   - Reduce reads from disk

2. Not suitable for:
   - Iterative algorithms (ML)
   - Interactive queries
   - Real-time processing

3. Two-stage only
   - Complex pipelines need chaining
   - Each stage = full disk round-trip
```

---

## MapReduce vs Modern Alternatives

| Aspect | MapReduce | Spark | Flink |
|--------|-----------|-------|-------|
| **Processing** | Batch | Batch + Stream | Stream-first |
| **Speed** | Slow (disk) | Fast (memory) | Fast (memory) |
| **Iterations** | Poor | Excellent | Excellent |
| **Ease of use** | Low-level | High-level APIs | High-level APIs |
| **Latency** | High | Medium | Low |

### Why Spark Won:
```
Spark keeps data in memory between stages
- 10-100x faster for iterative algorithms
- Same fault tolerance guarantees
- Richer API (SQL, ML, Graph)
```

---

## Real-World Usage

| Use Case | Example |
|----------|---------|
| **Log analysis** | Count errors by type |
| **ETL** | Transform and load data |
| **Indexing** | Build search indexes |
| **Analytics** | Aggregate metrics |
| **ML preprocessing** | Feature extraction |

---

## When to Use

### ✅ Good Fit:
- Batch processing of huge datasets
- Simple transformations
- Fault tolerance critical
- Cost-sensitive (disk cheaper than RAM)

### ❌ Avoid When:
- Need low latency
- Iterative algorithms (use Spark)
- Real-time processing (use Flink/Kafka)
- Interactive queries (use Presto/Trino)

---

## Interview Tips

When discussing MapReduce:
1. Explain the map-shuffle-reduce flow
2. Give word count as example
3. Discuss fault tolerance mechanism
4. Know limitations (disk I/O, iterations)
5. Compare with Spark (in-memory advantage)

