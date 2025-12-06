# LSM Trees (Log-Structured Merge Trees)

## What It Is

**LSM Tree** is a data structure optimized for write-heavy workloads. Instead of updating data in place (like B-trees), it buffers writes in memory and periodically flushes them to disk in sorted batches.

---

## The Analogy 📝

Think of **taking notes in class**:
- **B-tree approach**: Find the right page, erase, rewrite (slow, disruptive)
- **LSM approach**: Write notes on new paper, sort and merge later

LSM trades immediate organization for fast writes, organizing data in the background.

---

## Why It Exists

### The Problem: Write Amplification
```
B-tree random writes:
- Find page on disk (seek)
- Read page
- Modify in memory
- Write page back
- Each write = random I/O = SLOW

SSD/HDD reality:
- Sequential writes: 100-500 MB/s
- Random writes: 0.1-1 MB/s (100-1000x slower!)
```

### What LSM Solves:
- **Write optimization** - Sequential writes only
- **High throughput** - Buffer in memory, batch to disk
- **Space efficiency** - Compression friendly
- **SSD friendly** - Reduces write amplification

---

## How It Works

### Architecture:
```
┌─────────────────────────────────────────────────────────────────┐
│                      LSM Tree Structure                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   MEMORY                                                         │
│   ┌─────────────────────────────────────────┐                   │
│   │           MemTable (sorted)             │ ← Writes go here  │
│   │   key1:v1, key2:v2, key3:v3...         │                   │
│   └─────────────────────────────────────────┘                   │
│                      │ Flush when full                          │
│                      ▼                                          │
│   DISK                                                          │
│   ┌─────────────────────────────────────────┐                   │
│   │   Level 0: SSTable SSTable SSTable      │ ← Recent flushes  │
│   └─────────────────────────────────────────┘                   │
│                      │ Compaction                               │
│   ┌─────────────────────────────────────────┐                   │
│   │   Level 1: SSTable SSTable              │ ← Merged, sorted  │
│   └─────────────────────────────────────────┘                   │
│                      │                                          │
│   ┌─────────────────────────────────────────┐                   │
│   │   Level 2: SSTable                      │ ← Larger, fewer   │
│   └─────────────────────────────────────────┘                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components:

**MemTable:**
- In-memory sorted structure (Red-Black tree, Skip List)
- All writes go here first
- Flushed to disk when full (typically 64MB)

**SSTable (Sorted String Table):**
- Immutable, sorted file on disk
- Contains key-value pairs in order
- Includes index and bloom filter

**Write-Ahead Log (WAL):**
- Durability before MemTable flush
- Replayed on crash recovery

---

## Operations

### Write:
```
1. Append to WAL (durability)
2. Insert into MemTable
3. Return success
   (Background: flush MemTable → SSTable when full)
```

### Read:
```
1. Check MemTable
2. Check Level 0 SSTables (newest first)
3. Check Level 1, 2, ... (use bloom filters)
4. Return value or not found
```

### Compaction:
```
Merge overlapping SSTables:
- Remove deleted keys (tombstones)
- Keep only latest version
- Create larger, sorted SSTables
- Reduce read amplification
```

---

## Compaction Strategies

### Size-Tiered (STCS):
```
Merge SSTables of similar size
+ Simple, good write throughput
- Space amplification (up to 2x)
- Read amplification
```

### Leveled (LCS):
```
Each level is 10x larger than previous
+ Bounded space amplification
+ Better read performance
- More compaction work
```

---

## Trade-offs

| Aspect | LSM Tree | B-Tree |
|--------|----------|--------|
| **Write** | Fast (sequential) | Slower (random I/O) |
| **Read** | Slower (multiple levels) | Fast (single tree) |
| **Space** | More (multiple copies) | Less |
| **Write amplification** | Higher (compaction) | Lower |
| **Read amplification** | Higher (check levels) | Lower |

---

## When to Use

### ✅ Good Fit:
- **Write-heavy workloads** - Logging, time-series
- **Append-mostly data** - Events, metrics
- **Range queries** - Data is sorted
- **SSD storage** - Sequential writes

### ❌ Avoid When:
- Read-heavy, random access
- Need immediate consistency
- Limited disk space
- Latency-sensitive reads

---

## Real-World Examples

| Database | Notes |
|----------|-------|
| **LevelDB** | Google's LSM implementation |
| **RocksDB** | Facebook's fork of LevelDB |
| **Cassandra** | Uses LSM for storage |
| **HBase** | LSM-based on HDFS |
| **InfluxDB** | Time-series with LSM |
| **ScyllaDB** | LSM with better compaction |

---

## Interview Tips

When discussing LSM Trees:
1. Contrast with B-trees (random vs sequential I/O)
2. Explain MemTable → SSTable flow
3. Discuss compaction strategies
4. Mention read amplification trade-off
5. Give real examples (RocksDB, Cassandra)

