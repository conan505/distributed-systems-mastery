# Consistent Hashing

## What It Is

**Consistent Hashing** is a distributed hashing technique that minimizes key redistribution when nodes are added or removed. Instead of `hash(key) % n` which breaks when `n` changes, it maps both keys and nodes onto a ring structure.

---

## The Analogy 🎯

Imagine a **circular clock**:
- Hours 1-12 around the edge
- Servers placed at specific hours (Server A at 3, Server B at 7, etc.)
- Each key (data) hashes to an hour on the clock
- Key belongs to the **next server clockwise**

Traditional hashing: If you remove an hour, ALL assignments change
Consistent hashing: Only keys near that hour need to move

---

## Why It Exists

### The Problem with Traditional Hashing:
```
Normal: hash(key) % 3 servers
Server 0: key1, key4, key7...
Server 1: key2, key5, key8...
Server 2: key3, key6, key9...

Add 1 server: hash(key) % 4 servers
Almost ALL keys need to move! 💥

Cache invalidation = Cache miss storm
Database resharding = Massive data movement
```

### What Consistent Hashing Solves:
- **Minimal disruption** - Only K/n keys move (K = keys, n = nodes)
- **Scalability** - Add/remove nodes gracefully
- **Load distribution** - Keys spread across nodes
- **No central directory** - Any node can compute key location

---

## How It Works

### The Ring Structure:
```
┌─────────────────────────────────────────────────────────────────┐
│                      Consistent Hash Ring                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                         0° (top)                                 │
│                           │                                      │
│                    ┌──────┴──────┐                              │
│                   /               \                              │
│                  /                 \                             │
│           Node A (45°)          Node B (135°)                   │
│                │                   │                             │
│                │     key1 (60°)    │                            │
│                │        ↓         │                             │
│                │    → Node B      │                             │
│                 \                 /                              │
│                  \               /                               │
│                   \    Node C   /                                │
│                    \   (270°) /                                  │
│                     ─────────                                    │
│                                                                  │
│  key1 hashes to 60° → walks clockwise → finds Node B (135°)     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Virtual Nodes (Vnodes):
Each physical node has multiple positions on the ring:
```
Physical Node A → Virtual: A1(45°), A2(120°), A3(250°)
Physical Node B → Virtual: B1(80°), B2(180°), B3(320°)

Benefits:
- Better load distribution
- Smooth transition when nodes fail
- Handle heterogeneous nodes (powerful = more vnodes)
```

---

## Step-by-Step Example

### Initial State (3 nodes):
```
Ring positions: A(100), B(200), C(300)

Keys:
- key1 hash=150 → assigned to B (next node clockwise)
- key2 hash=250 → assigned to C
- key3 hash=350 → assigned to A (wraps around)
```

### Adding Node D at position 175:
```
Ring positions: A(100), D(175), B(200), C(300)

Only key1 (hash=150) moves from B to D
key2 and key3 stay where they are!
```

### Removing Node B:
```
Ring positions: A(100), D(175), C(300)

Only keys in range (175, 200] move to C
Everything else unchanged!
```

---

## When to Use It

### ✅ Good Fit:
- **Distributed caches** - Memcached, Redis Cluster
- **CDNs** - Route content to edge servers
- **Database sharding** - DynamoDB, Cassandra
- **Load balancing** - Sticky sessions without central state
- **Distributed storage** - Riak, Voldemort

### ❌ Avoid When:
- Single node systems
- Data locality matters more than distribution
- Need strict ordering or transactions
- Very small, static cluster

---

## How to Use It Effectively

### Best Practices:

1. **Use virtual nodes**
   - Minimum 100-200 vnodes per physical node
   - Better load distribution

2. **Choose hash function wisely**
   - MD5 or SHA are common
   - Must be uniform distribution

3. **Replication strategy**
   - Store on next N nodes clockwise
   - Handle node failures gracefully

4. **Handle hotspots**
   - Monitor key distribution
   - Adjust vnodes for heavy keys

5. **Bounded load** (Google's improvement)
   - Cap load at 1 + ε times average
   - Redirect overflow to next node

---

## Real-World Implementations

| System | Usage |
|--------|-------|
| **DynamoDB** | Partition key distribution |
| **Cassandra** | Token ring for data distribution |
| **Memcached** | Client-side key distribution |
| **Akamai CDN** | Content routing |
| **Discord** | User session routing |

---

## Implementation Deep Dive

### Core Data Structures
```python
import hashlib
import bisect

class ConsistentHash:
    def __init__(self, vnodes=150):
        self.vnodes = vnodes           # Virtual nodes per physical node
        self.ring = {}                  # hash_value -> node_id
        self.sorted_keys = []           # Sorted hash positions for binary search
        self.nodes = set()              # Track physical nodes

    def _hash(self, key: str) -> int:
        """Use MD5 for uniform distribution (not cryptographic security)"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """Add physical node with virtual nodes to the ring"""
        self.nodes.add(node)
        for i in range(self.vnodes):
            vnode_key = f"{node}:vnode{i}"
            h = self._hash(vnode_key)
            self.ring[h] = node
            bisect.insort(self.sorted_keys, h)  # Maintain sorted order

    def remove_node(self, node: str):
        """Remove node and all its virtual nodes"""
        self.nodes.discard(node)
        for i in range(self.vnodes):
            h = self._hash(f"{node}:vnode{i}")
            if h in self.ring:
                del self.ring[h]
                self.sorted_keys.remove(h)

    def get_node(self, key: str) -> str:
        """Find responsible node using binary search - O(log n)"""
        if not self.ring:
            return None
        h = self._hash(key)
        # Binary search for first position >= hash
        idx = bisect.bisect_left(self.sorted_keys, h)
        if idx == len(self.sorted_keys):
            idx = 0  # Wrap around to first node
        return self.ring[self.sorted_keys[idx]]

    def get_replicas(self, key: str, n: int = 3) -> list:
        """Get n replica nodes for replication (walk clockwise)"""
        if not self.ring or n > len(self.nodes):
            return list(self.nodes)

        h = self._hash(key)
        idx = bisect.bisect_left(self.sorted_keys, h)
        replicas = []
        seen = set()

        while len(replicas) < n:
            if idx >= len(self.sorted_keys):
                idx = 0
            node = self.ring[self.sorted_keys[idx]]
            if node not in seen:
                replicas.append(node)
                seen.add(node)
            idx += 1
        return replicas
```

### Key Implementation Details

| Aspect | Implementation Choice | Why |
|--------|----------------------|-----|
| **Hash Function** | MD5/SHA1 | Uniform distribution, fast |
| **Virtual Nodes** | 100-200 per node | Balance load variance |
| **Lookup** | Binary search (bisect) | O(log n) instead of O(n) |
| **Ring Storage** | Dict + Sorted List | Fast lookup + ordered traversal |

### Production Considerations

```python
# 1. Bounded Load (Google's improvement)
def get_node_bounded(self, key, max_load_factor=1.25):
    """Redirect to next node if current is overloaded"""
    node = self.get_node(key)
    avg_load = total_keys / len(self.nodes)
    while self.load[node] > avg_load * max_load_factor:
        node = self.get_next_node(node)
    return node

# 2. Weighted nodes (heterogeneous clusters)
def add_node(self, node, weight=1):
    vnodes = self.base_vnodes * weight  # More vnodes = more keys
    for i in range(vnodes):
        ...
```

---

## Interview Tips

When discussing Consistent Hashing:
1. Start with the problem (traditional hash redistribution)
2. Explain the ring structure visually
3. Emphasize K/n redistribution (minimal movement)
4. Mention virtual nodes for load balancing
5. Give real examples (DynamoDB, Cassandra)

