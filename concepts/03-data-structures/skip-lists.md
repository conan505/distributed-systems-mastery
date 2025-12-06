# Skip Lists

## What It Is

A **Skip List** is a probabilistic data structure that allows O(log n) search, insert, and delete operations. It's like a linked list with multiple levels of "express lanes" for faster traversal.

---

## The Analogy 🚇

Think of a **subway system**:
- **Local train**: Stops at every station (linked list)
- **Express train**: Skips stations, stops at major hubs
- **Super express**: Only stops at a few key stations

To get somewhere fast:
1. Take super express as far as possible
2. Switch to express
3. Take local for the last few stops

Skip lists work the same way with multiple levels!

---

## Why It Exists

### The Problem: Linked List Limitations
```
Linked List:
- Search: O(n) - must traverse sequentially
- Insert: O(1) after finding position
- Delete: O(1) after finding position

Need: O(log n) operations like balanced trees
But: Trees are complex (rotations, rebalancing)
```

### What Skip Lists Solve:
- **O(log n) operations** - Like balanced trees
- **Simpler implementation** - No rotations
- **Probabilistic balance** - No explicit rebalancing
- **Lock-free friendly** - Easier concurrent access

---

## How It Works

### Structure:
```
┌─────────────────────────────────────────────────────────────────┐
│                      Skip List                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Level 3:  HEAD ─────────────────────────────────▶ 50 ──▶ NIL  │
│              │                                      │           │
│   Level 2:  HEAD ──────────▶ 20 ──────────────────▶ 50 ──▶ NIL  │
│              │               │                      │           │
│   Level 1:  HEAD ──▶ 10 ──▶ 20 ──────────▶ 40 ──▶ 50 ──▶ NIL  │
│              │      │       │              │       │           │
│   Level 0:  HEAD ──▶ 10 ──▶ 20 ──▶ 30 ──▶ 40 ──▶ 50 ──▶ NIL  │
│             (base level - all elements)                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Search for 40:
```
1. Start at HEAD, Level 3
2. 50 > 40, go down to Level 2
3. 20 < 40, move right to 50
4. 50 > 40, go down to Level 1
5. 40 == 40, FOUND!

Skipped: 10, 30 (didn't need to check them)
```

---

## Operations

### Search:
```python
def search(self, target):
    current = self.head
    for level in range(self.max_level, -1, -1):
        while current.next[level] and current.next[level].value < target:
            current = current.next[level]
    current = current.next[0]
    return current if current and current.value == target else None
```

### Insert:
```python
def insert(self, value):
    # 1. Find position at each level
    update = [None] * (self.max_level + 1)
    current = self.head
    
    for level in range(self.max_level, -1, -1):
        while current.next[level] and current.next[level].value < value:
            current = current.next[level]
        update[level] = current
    
    # 2. Randomly determine height
    new_level = random_level()  # Coin flips
    
    # 3. Insert node at each level up to new_level
    new_node = Node(value, new_level)
    for level in range(new_level + 1):
        new_node.next[level] = update[level].next[level]
        update[level].next[level] = new_node
```

### Level Generation:
```python
def random_level():
    level = 0
    while random() < 0.5 and level < MAX_LEVEL:
        level += 1
    return level

# Probability:
# Level 0: 100% of nodes
# Level 1: 50% of nodes
# Level 2: 25% of nodes
# Level 3: 12.5% of nodes
```

---

## Complexity Analysis

| Operation | Average | Worst Case |
|-----------|---------|------------|
| **Search** | O(log n) | O(n) |
| **Insert** | O(log n) | O(n) |
| **Delete** | O(log n) | O(n) |
| **Space** | O(n) | O(n log n) |

Worst case is rare due to probabilistic nature.

---

## Skip List vs Balanced Trees

| Aspect | Skip List | Balanced Tree |
|--------|-----------|---------------|
| **Implementation** | Simpler | Complex |
| **Rebalancing** | None (probabilistic) | Required |
| **Concurrency** | Easier (lock-free) | Harder |
| **Cache locality** | Worse | Better |
| **Deterministic** | No | Yes |
| **Range queries** | Easy | Easy |

---

## When to Use

### ✅ Good Fit:
- **In-memory databases** - Redis sorted sets
- **Concurrent data structures** - Lock-free implementations
- **Range queries** - Ordered data
- **Simple implementation needed** - Avoid tree complexity

### ❌ Avoid When:
- Disk-based storage (poor locality)
- Deterministic performance required
- Memory is very constrained

---

## Real-World Examples

| System | Usage |
|--------|-------|
| **Redis** | Sorted sets (ZSET) |
| **LevelDB/RocksDB** | MemTable implementation |
| **Lucene** | Posting lists |
| **HBase** | MemStore |

---

## Concurrent Skip Lists

```
Why skip lists are good for concurrency:
- Insert only modifies local pointers
- No global rebalancing
- Can use CAS (Compare-And-Swap) operations
- Java's ConcurrentSkipListMap uses this

Lock-free insert:
1. Find position
2. Create new node
3. CAS to insert at each level
4. Retry if CAS fails
```

---

## Interview Tips

When discussing Skip Lists:
1. Use the subway/express train analogy
2. Explain probabilistic level assignment
3. Compare with balanced trees (simpler, concurrent)
4. Mention Redis sorted sets as real example
5. Discuss O(log n) expected complexity

