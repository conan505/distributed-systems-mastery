# B-Trees and B+ Trees

## What They Are

**B-Trees** are self-balancing tree data structures optimized for systems that read and write large blocks of data, particularly databases and file systems.

**B+ Trees** are a variant where all values are stored in leaf nodes, with internal nodes only containing keys for navigation.

---

## The Analogy 📚

Think of a **library catalog system**:
- **Binary tree**: One card per drawer, deep hierarchy
- **B-Tree**: Multiple cards per drawer, shallow hierarchy
- Each drawer (node) holds many entries
- Fewer drawers to open = faster search

---

## Why They Exist

### The Problem: Disk I/O
```
Binary Search Tree:
- Height: O(log₂ N)
- 1 billion records = 30 levels
- 30 disk reads per search!

B-Tree (order 1000):
- Height: O(log₁₀₀₀ N)
- 1 billion records = 3 levels
- 3 disk reads per search!
```

### What B-Trees Solve:
- **Minimize disk I/O** - Fewer, larger reads
- **Efficient range queries** - Sequential leaf access
- **Self-balancing** - Guaranteed O(log N) operations
- **High fanout** - Shallow trees

---

## B-Tree Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    B-Tree (Order 3)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                        [30 | 70]                                │
│                       /    |    \                               │
│                      /     |     \                              │
│              [10|20]    [40|50|60]    [80|90]                   │
│                                                                  │
│   Properties:                                                    │
│   - Each node has at most m children (m = order)                │
│   - Each node has at least ⌈m/2⌉ children (except root)        │
│   - All leaves at same level                                    │
│   - Keys in sorted order within node                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### B-Tree Properties:
```
Order m B-Tree:
- Max keys per node: m - 1
- Max children per node: m
- Min keys (non-root): ⌈m/2⌉ - 1
- Min children (non-root): ⌈m/2⌉
```

---

## B+ Tree Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    B+ Tree (Order 3)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                        [30 | 70]                                │
│                       /    |    \                               │
│                      /     |     \                              │
│              [10|20]    [30|50]    [70|90]                      │
│                │           │          │                         │
│                ▼           ▼          ▼                         │
│   Leaf:    [10→20→] ──▶ [30→50→] ──▶ [70→90→]                  │
│            (data)       (data)       (data)                     │
│                                                                  │
│   Key differences from B-Tree:                                  │
│   - ALL data in leaf nodes                                      │
│   - Internal nodes = index only                                 │
│   - Leaves linked for range scans                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## B-Tree vs B+ Tree

| Aspect | B-Tree | B+ Tree |
|--------|--------|---------|
| **Data location** | All nodes | Leaves only |
| **Leaf linking** | No | Yes (linked list) |
| **Range queries** | Slower | Fast (sequential) |
| **Space** | Less duplication | Keys duplicated |
| **Point queries** | Can be faster | Always to leaf |

### Why Databases Use B+ Trees:
```
1. Range queries are common (WHERE x BETWEEN a AND b)
2. Sequential leaf scan is cache-friendly
3. Internal nodes smaller = more keys = shallower tree
4. Predictable performance (always same depth)
```

---

## Operations

### Search:
```
search(key):
    node = root
    while node is not leaf:
        find child pointer for key
        node = child
    search key in leaf node
    
Time: O(log_m N) disk reads
```

### Insert:
```
insert(key, value):
    1. Find leaf node for key
    2. If leaf has space: insert
    3. If leaf full: split
       - Create new leaf
       - Move half keys to new leaf
       - Insert median key in parent
       - Recursively split parent if needed
```

### Delete:
```
delete(key):
    1. Find and remove key from leaf
    2. If leaf underflows (< min keys):
       - Borrow from sibling, OR
       - Merge with sibling
    3. Recursively fix parent if needed
```

---

## Real-World Usage

| System | Tree Type | Notes |
|--------|-----------|-------|
| **MySQL InnoDB** | B+ Tree | Primary index structure |
| **PostgreSQL** | B+ Tree | Default index type |
| **SQLite** | B+ Tree | File-based storage |
| **MongoDB** | B-Tree | WiredTiger engine |
| **File Systems** | B+ Tree | NTFS, ext4, HFS+ |

---

## B+ Tree in Databases

### Clustered Index:
```
Table data stored in B+ tree leaf order
- Primary key determines physical order
- Range scans on PK are sequential I/O
- Only one clustered index per table
```

### Secondary Index:
```
Separate B+ tree with:
- Key: indexed column
- Value: pointer to primary key

Lookup: Secondary index → Primary key → Clustered index
```

---

## When to Use

### ✅ Good Fit:
- **Databases** - Primary storage structure
- **File systems** - Directory indexing
- **Range queries** - Efficient sequential access
- **Disk-based storage** - Minimize I/O

### ❌ Avoid When:
- In-memory only (hash tables faster for point queries)
- Write-heavy with random keys (LSM trees better)
- Simple key-value (hash index sufficient)

---

## Comparison with Other Structures

| Structure | Point Query | Range Query | Insert | Use Case |
|-----------|-------------|-------------|--------|----------|
| **Hash** | O(1) | O(N) | O(1) | Key-value |
| **B+ Tree** | O(log N) | O(log N + K) | O(log N) | Databases |
| **LSM Tree** | O(log N) | O(log N + K) | O(1)* | Write-heavy |

---

## Interview Tips

When discussing B-Trees:
1. Explain why high fanout matters (disk I/O)
2. Know B-Tree vs B+ Tree differences
3. Describe split/merge operations
4. Mention database index usage
5. Compare with LSM trees for write-heavy workloads

