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

## Implementation

### Simplified LSM Tree

```python
import os
import json
from typing import Optional, Dict, List
from sortedcontainers import SortedDict

class MemTable:
    """In-memory sorted buffer."""

    def __init__(self, max_size: int = 1000):
        self.data = SortedDict()
        self.max_size = max_size

    def put(self, key: str, value: str):
        self.data[key] = value

    def get(self, key: str) -> Optional[str]:
        return self.data.get(key)

    def delete(self, key: str):
        self.data[key] = None  # Tombstone

    def is_full(self) -> bool:
        return len(self.data) >= self.max_size

    def flush(self) -> Dict[str, str]:
        data = dict(self.data)
        self.data.clear()
        return data

class SSTable:
    """Immutable sorted file on disk."""

    def __init__(self, filepath: str, data: Dict[str, str] = None):
        self.filepath = filepath
        if data:
            self._write(data)
        self._load_index()

    def _write(self, data: Dict[str, str]):
        with open(self.filepath, 'w') as f:
            for key in sorted(data.keys()):
                f.write(f"{key}\t{data[key]}\n")

    def _load_index(self):
        """Sparse index for faster lookups."""
        self.index = {}
        with open(self.filepath, 'r') as f:
            offset = 0
            for line in f:
                key = line.split('\t')[0]
                self.index[key] = offset
                offset = f.tell()

    def get(self, key: str) -> Optional[str]:
        if key not in self.index:
            return None
        with open(self.filepath, 'r') as f:
            f.seek(self.index[key])
            line = f.readline()
            k, v = line.strip().split('\t', 1)
            return None if v == 'None' else v

class LSMTree:
    """Simplified LSM tree with compaction."""

    def __init__(self, data_dir: str):
        self.data_dir = data_dir
        self.memtable = MemTable()
        self.sstables: List[SSTable] = []
        self.sstable_counter = 0

    def put(self, key: str, value: str):
        self.memtable.put(key, value)
        if self.memtable.is_full():
            self._flush()

    def get(self, key: str) -> Optional[str]:
        # Check MemTable first (most recent)
        result = self.memtable.get(key)
        if result is not None:
            return result

        # Check SSTables (newest to oldest)
        for sstable in reversed(self.sstables):
            result = sstable.get(key)
            if result is not None:
                return result

        return None

    def delete(self, key: str):
        self.memtable.delete(key)  # Write tombstone

    def _flush(self):
        """Flush MemTable to disk as new SSTable."""
        data = self.memtable.flush()
        filepath = f"{self.data_dir}/sstable_{self.sstable_counter}.dat"
        self.sstables.append(SSTable(filepath, data))
        self.sstable_counter += 1

        # Trigger compaction if too many SSTables
        if len(self.sstables) > 4:
            self._compact()

    def _compact(self):
        """Merge oldest SSTables."""
        merged = {}
        for sstable in self.sstables[:2]:
            for key in sstable.index:
                value = sstable.get(key)
                if value is not None:  # Skip tombstones
                    merged[key] = value
            os.remove(sstable.filepath)

        # Create compacted SSTable
        filepath = f"{self.data_dir}/sstable_{self.sstable_counter}.dat"
        self.sstables = self.sstables[2:]
        self.sstables.insert(0, SSTable(filepath, merged))
        self.sstable_counter += 1
```

---

## Interview Tips

When discussing LSM Trees:
1. Contrast with B-trees (random vs sequential I/O)
2. Explain MemTable → SSTable flow
3. Discuss compaction (size-tiered vs leveled)
4. Mention read amplification trade-off (check multiple levels)
5. Give real examples (RocksDB, Cassandra, LevelDB)

