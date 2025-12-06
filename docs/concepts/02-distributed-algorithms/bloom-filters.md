# Bloom Filters

## What It Is

A **Bloom Filter** is a space-efficient probabilistic data structure that tests whether an element is a member of a set. It can tell you:
- "Definitely NOT in set" (100% certain)
- "Probably in set" (false positives possible)

---

## The Analogy 🎯

Imagine a **security checkpoint** with multiple guards:
- Person arrives, 3 guards check different lists
- Guard 1 checks list A, Guard 2 checks list B, Guard 3 checks list C
- If ANY guard says "not on my list" → Person definitely not allowed
- If ALL guards say "on my list" → Person probably allowed (might be wrong)

The guards might confuse similar names, but if even one says "no", the answer is definitive.

---

## Why It Exists

### The Problem: Membership Testing at Scale
```
Traditional Set: Store all elements
- 1 billion URLs = ~50GB memory
- Lookup: O(1) hash lookup, but huge storage

Question: "Have I seen this URL before?"
Need: Fast answer with minimal memory
```

### What Bloom Filter Solves:
- **Space efficiency** - 1 billion items in ~1GB (10+ bits per item)
- **Fast lookups** - O(k) where k is number of hash functions
- **No false negatives** - If it says "no", it's definitely no
- **Trade-off accepted** - Some false positives okay for savings

---

## How It Works

### Structure:
```
┌─────────────────────────────────────────────────────────────────┐
│                      Bloom Filter                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Bit Array (m bits):                                            │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐        │
│   │ 0 │ 1 │ 0 │ 1 │ 0 │ 0 │ 1 │ 0 │ 1 │ 0 │ 0 │ 1 │ 0 │        │
│   └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘        │
│     0   1   2   3   4   5   6   7   8   9  10  11  12           │
│                                                                  │
│   Insert "apple":                                                │
│   h1("apple") = 1  → set bit 1                                  │
│   h2("apple") = 6  → set bit 6                                  │
│   h3("apple") = 11 → set bit 11                                 │
│                                                                  │
│   Check "apple": bits 1, 6, 11 all set? → Probably yes          │
│   Check "banana": bit 4 not set? → Definitely no                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Operations:

**Insert(element):**
1. Hash element with k different hash functions
2. Set all k bit positions to 1

**Check(element):**
1. Hash element with same k functions
2. If ANY bit is 0 → Element definitely not present
3. If ALL bits are 1 → Element probably present

---

## False Positive Rate

```
Formula: (1 - e^(-kn/m))^k

Where:
- m = number of bits
- n = number of elements inserted
- k = number of hash functions

Optimal k = (m/n) * ln(2) ≈ 0.693 * (m/n)
```

### Practical Example:
| Items (n) | Bits (m) | Hash funcs (k) | FP Rate |
|-----------|----------|----------------|---------|
| 1M | 10M | 7 | ~1% |
| 1M | 15M | 10 | ~0.1% |
| 1M | 20M | 14 | ~0.01% |

---

## When to Use It

### ✅ Good Fit:
- **Web crawlers** - "Have I visited this URL?"
- **Spell checkers** - "Is this a valid word?"
- **Database optimization** - Skip disk reads for non-existent keys
- **CDN caching** - Check if content might be cached
- **Malware detection** - Quick signature lookup
- **Username availability** - Fast "definitely taken" check

### ❌ Avoid When:
- False positives are unacceptable
- Need to delete elements (use Counting Bloom Filter)
- Need to enumerate members
- Small datasets (just use a hash set)

---

## Real-World Examples

| System | Usage |
|--------|-------|
| **Google Chrome** | Safe browsing (malicious URL check) |
| **Cassandra** | Skip SSTable reads |
| **HBase** | Block filter for row existence |
| **Medium** | Article recommendation dedup |
| **Akamai** | One-hit-wonder cache avoidance |

---

## Implementation Deep Dive

### Production-Ready Implementation

```python
import math
import hashlib
from typing import Union

class BloomFilter:
    """
    Production-ready Bloom Filter with optimal parameter calculation.
    """

    def __init__(self, expected_items: int, false_positive_rate: float = 0.01):
        """
        Auto-calculate optimal size and hash count.

        Args:
            expected_items: Expected number of items to insert
            false_positive_rate: Desired false positive rate (default 1%)
        """
        # Optimal bit array size: m = -n*ln(p) / (ln(2)^2)
        self.size = self._optimal_size(expected_items, false_positive_rate)

        # Optimal hash count: k = (m/n) * ln(2)
        self.num_hashes = self._optimal_hashes(self.size, expected_items)

        # Use bytearray for memory efficiency
        self.bit_array = bytearray((self.size + 7) // 8)
        self.count = 0

    @staticmethod
    def _optimal_size(n: int, p: float) -> int:
        """Calculate optimal bit array size."""
        return int(-n * math.log(p) / (math.log(2) ** 2))

    @staticmethod
    def _optimal_hashes(m: int, n: int) -> int:
        """Calculate optimal number of hash functions."""
        return max(1, int((m / n) * math.log(2)))

    def _get_hash_values(self, item: Union[str, bytes]) -> list[int]:
        """
        Generate k hash values using double hashing technique.
        h(i) = h1 + i*h2 (more efficient than k independent hashes)
        """
        if isinstance(item, str):
            item = item.encode('utf-8')

        # Use SHA256 for two 128-bit hashes
        digest = hashlib.sha256(item).digest()
        h1 = int.from_bytes(digest[:16], 'big')
        h2 = int.from_bytes(digest[16:], 'big')

        return [(h1 + i * h2) % self.size for i in range(self.num_hashes)]

    def _set_bit(self, index: int):
        """Set bit at index."""
        self.bit_array[index // 8] |= (1 << (index % 8))

    def _get_bit(self, index: int) -> bool:
        """Get bit at index."""
        return bool(self.bit_array[index // 8] & (1 << (index % 8)))

    def add(self, item: Union[str, bytes]):
        """Add item to filter."""
        for index in self._get_hash_values(item):
            self._set_bit(index)
        self.count += 1

    def contains(self, item: Union[str, bytes]) -> bool:
        """
        Check if item might be in filter.
        Returns False = definitely not present
        Returns True = probably present (may be false positive)
        """
        return all(self._get_bit(i) for i in self._get_hash_values(item))

    def estimated_false_positive_rate(self) -> float:
        """Calculate current estimated FP rate based on fill ratio."""
        # (1 - e^(-kn/m))^k
        return (1 - math.exp(-self.num_hashes * self.count / self.size)) ** self.num_hashes

    def __len__(self):
        return self.count

# Usage with auto-optimization
bf = BloomFilter(expected_items=1_000_000, false_positive_rate=0.01)
print(f"Size: {bf.size:,} bits ({bf.size // 8 // 1024} KB)")
print(f"Hash functions: {bf.num_hashes}")

bf.add("user@example.com")
print(bf.contains("user@example.com"))  # True
print(bf.contains("other@example.com")) # False (definitely not present)
```

### Counting Bloom Filter (Supports Deletion)

```python
class CountingBloomFilter:
    """
    Bloom filter variant that supports deletion using counters.
    Trade-off: 4x more memory (4-bit counters instead of 1-bit)
    """

    def __init__(self, expected_items: int, false_positive_rate: float = 0.01):
        self.size = int(-expected_items * math.log(false_positive_rate) / (math.log(2) ** 2))
        self.num_hashes = max(1, int((self.size / expected_items) * math.log(2)))
        # 4-bit counters (max count = 15)
        self.counters = bytearray(self.size)

    def add(self, item: str):
        for index in self._get_hash_values(item):
            if self.counters[index] < 15:  # Prevent overflow
                self.counters[index] += 1

    def remove(self, item: str):
        """Remove item (decrement counters)."""
        if not self.contains(item):
            return False
        for index in self._get_hash_values(item):
            if self.counters[index] > 0:
                self.counters[index] -= 1
        return True

    def contains(self, item: str) -> bool:
        return all(self.counters[i] > 0 for i in self._get_hash_values(item))
```

### Distributed Bloom Filter (Redis)

```python
import redis

class RedisBloomFilter:
    """
    Distributed Bloom Filter using Redis BITSET operations.
    Supports multiple application instances.
    """

    def __init__(self, redis_client: redis.Redis, key: str,
                 expected_items: int, fp_rate: float = 0.01):
        self.redis = redis_client
        self.key = key
        self.size = int(-expected_items * math.log(fp_rate) / (math.log(2) ** 2))
        self.num_hashes = max(1, int((self.size / expected_items) * math.log(2)))

    def add(self, item: str):
        """Atomic add using pipeline."""
        pipe = self.redis.pipeline()
        for index in self._get_hash_values(item):
            pipe.setbit(self.key, index, 1)
        pipe.execute()

    def contains(self, item: str) -> bool:
        """Check membership."""
        pipe = self.redis.pipeline()
        for index in self._get_hash_values(item):
            pipe.getbit(self.key, index)
        return all(pipe.execute())

    def add_batch(self, items: list[str]):
        """Efficient batch insert."""
        pipe = self.redis.pipeline()
        for item in items:
            for index in self._get_hash_values(item):
                pipe.setbit(self.key, index, 1)
        pipe.execute()
```

---

## Variants

| Variant | Feature | Use Case |
|---------|---------|----------|
| **Counting Bloom Filter** | Supports deletion (4-bit counters) | Dynamic sets |
| **Scalable Bloom Filter** | Grows as elements added | Unknown size |
| **Cuckoo Filter** | Better deletion, lower FP | High load factor |
| **Quotient Filter** | Cache-friendly, mergeable | Disk-based storage |

---

## Interview Tips

When discussing Bloom Filters:
1. Emphasize NO false negatives
2. Explain the space-time trade-off
3. Mention real examples (Chrome Safe Browsing, Cassandra)
4. Know the formula for optimal hash functions: k = (m/n) * ln(2)
5. Discuss when NOT to use (need deletion, low tolerance for FP)
6. Mention double hashing technique for efficient hash generation

