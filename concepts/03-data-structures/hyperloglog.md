# HyperLogLog

## What It Is

**HyperLogLog** is a probabilistic data structure for estimating the cardinality (count of unique elements) of very large datasets using minimal memory. It can count billions of unique items using only ~12KB of memory with 97% accuracy.

---

## The Analogy 🎲

Imagine **estimating how many times a coin was flipped** by looking at the longest streak of heads:
- If longest streak is 3 heads in a row → probably ~8 flips
- If longest streak is 10 heads → probably ~1000 flips
- Rare events indicate many attempts

HyperLogLog uses this principle: rare hash patterns indicate many unique items.

---

## Why It Exists

### The Problem: Counting Unique Items at Scale
```
Task: Count unique visitors to a website
Naive approach: Store all visitor IDs in a set

1 billion unique visitors × 8 bytes = 8 GB memory!

What if we could estimate with 97% accuracy using 12 KB?
```

### What HyperLogLog Solves:
- **Memory efficiency** - O(1) space, not O(n)
- **Speed** - O(1) add and count operations
- **Mergeability** - Combine counts from different sources
- **Accuracy** - ~2% standard error with 12KB

---

## How It Works

### Core Intuition:
```
1. Hash each element to get uniform random bits
2. Count leading zeros in the hash
3. More leading zeros = rarer event = more unique elements

Example:
hash("user1") = 0001... (3 leading zeros)
hash("user2") = 01...   (1 leading zero)
hash("user3") = 00001.. (4 leading zeros)

Max leading zeros = 4
Estimate ≈ 2^4 = 16 unique elements
```

### Algorithm:
```
┌─────────────────────────────────────────────────────────────────┐
│                    HyperLogLog Structure                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   hash(element) = [bucket bits][position bits]                   │
│                   (first b bits)  (remaining bits)               │
│                                                                  │
│   Example with 4 buckets (b=2):                                  │
│   hash = 01|00010110...                                          │
│          ↓  ↓                                                    │
│   bucket=1  leading zeros in rest = 3                            │
│                                                                  │
│   Buckets: [max_zeros_0, max_zeros_1, max_zeros_2, max_zeros_3] │
│            [5, 3, 7, 2]                                          │
│                                                                  │
│   Estimate = α × m² / Σ(2^(-bucket[i]))                         │
│   (harmonic mean with correction factor α)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why Multiple Buckets?
Single bucket has high variance. Using m buckets and taking harmonic mean:
- Reduces variance by √m
- Standard error ≈ 1.04/√m
- 16,384 buckets → ~0.8% error

---

## Operations

### Add Element:
```python
def add(element):
    h = hash(element)
    bucket = h >> (64 - b)  # First b bits
    remaining = h & ((1 << (64 - b)) - 1)
    leading_zeros = count_leading_zeros(remaining) + 1
    buckets[bucket] = max(buckets[bucket], leading_zeros)
```

### Count:
```python
def count():
    # Harmonic mean
    indicator = sum(2**(-bucket) for bucket in buckets)
    estimate = alpha * m * m / indicator
    
    # Apply corrections for small/large counts
    return apply_corrections(estimate)
```

### Merge:
```python
def merge(hll1, hll2):
    # Simply take max of each bucket
    return [max(hll1[i], hll2[i]) for i in range(m)]
```

---

## Memory vs Accuracy

| Buckets (m) | Memory | Error Rate |
|-------------|--------|------------|
| 16 | 12 bytes | 26% |
| 256 | 192 bytes | 6.5% |
| 4,096 | 3 KB | 1.6% |
| 16,384 | 12 KB | 0.8% |
| 65,536 | 48 KB | 0.4% |

---

## When to Use

### ✅ Good Fit:
- **Unique visitor counting** - Web analytics
- **Distinct value estimation** - Database query planning
- **Cardinality in streams** - Real-time analytics
- **Distributed counting** - Merge counts from shards

### ❌ Avoid When:
- Exact count required
- Small datasets (just use a set)
- Need to list unique elements
- Low cardinality (< 1000 items)

---

## Real-World Examples

| System | Usage |
|--------|-------|
| **Redis** | PFADD, PFCOUNT, PFMERGE commands |
| **BigQuery** | APPROX_COUNT_DISTINCT |
| **Presto** | approx_distinct() |
| **Spark** | approxCountDistinct() |
| **Elasticsearch** | Cardinality aggregation |

---

## Redis Example

```redis
# Add elements
PFADD visitors user1 user2 user3 user1  # user1 counted once

# Get count
PFCOUNT visitors  # Returns ~3

# Merge multiple HLLs
PFMERGE total visitors_page1 visitors_page2
PFCOUNT total
```

---

## Comparison with Alternatives

| Approach | Memory | Accuracy | Mergeable |
|----------|--------|----------|-----------|
| **HashSet** | O(n) | 100% | Yes (union) |
| **Bloom Filter** | O(n) bits | No count | No |
| **HyperLogLog** | O(1) | ~98% | Yes |
| **Linear Counting** | O(n) bits | High | No |

---

## Interview Tips

When discussing HyperLogLog:
1. Use the coin flip/leading zeros analogy
2. Explain why multiple buckets reduce variance
3. Mention 12KB for billions with 2% error
4. Give real examples (Redis PFCOUNT)
5. Discuss when NOT to use (exact counts, small data)

