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

## Implementation

```python
import mmh3  # MurmurHash
from bitarray import bitarray

class BloomFilter:
    def __init__(self, size, num_hashes):
        self.size = size
        self.num_hashes = num_hashes
        self.bits = bitarray(size)
        self.bits.setall(0)
    
    def add(self, item):
        for seed in range(self.num_hashes):
            index = mmh3.hash(item, seed) % self.size
            self.bits[index] = 1
    
    def contains(self, item):
        for seed in range(self.num_hashes):
            index = mmh3.hash(item, seed) % self.size
            if self.bits[index] == 0:
                return False  # Definitely not present
        return True  # Probably present

# Usage
bf = BloomFilter(size=1000000, num_hashes=7)
bf.add("apple")
bf.contains("apple")   # True (correct)
bf.contains("orange")  # False (definitely not present)
bf.contains("apricot") # True or False (might be false positive)
```

---

## Variants

| Variant | Feature |
|---------|---------|
| **Counting Bloom Filter** | Supports deletion (uses counters instead of bits) |
| **Scalable Bloom Filter** | Grows as elements added |
| **Cuckoo Filter** | Better deletion, lower FP for high load |

---

## Interview Tips

When discussing Bloom Filters:
1. Emphasize NO false negatives
2. Explain the space-time trade-off
3. Mention real examples (Chrome Safe Browsing)
4. Know the formula for optimal hash functions
5. Discuss when NOT to use (need deletion, low tolerance for FP)

