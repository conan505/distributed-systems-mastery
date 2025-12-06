# Reservoir Sampling

## What It Is

**Reservoir Sampling** is an algorithm for randomly selecting k items from a stream of n items, where n is unknown or very large, using only O(k) memory.

---

## The Analogy 🎣

Imagine **fishing from an infinite stream**:
- You can only keep k fish in your bucket
- You don't know how many fish will come
- Each fish must have equal chance of being kept
- You can't go back upstream

Reservoir sampling ensures fair selection with limited storage.

---

## Why It Exists

### The Problem: Sampling from Streams
```
Scenario: Sample 1000 users from event stream

Naive approach:
1. Store all events in memory
2. Randomly select 1000
Problem: Stream has billions of events!

What we need:
- Fixed memory (only k items)
- Fair probability for all items
- Single pass through data
```

### What Reservoir Sampling Solves:
- **Memory bounded** - O(k) regardless of stream size
- **Single pass** - Process each item once
- **Uniform sampling** - Each item has equal probability
- **Unknown size** - Works without knowing n

---

## Algorithm

### Algorithm R (Simple)
```python
import random

def reservoir_sample(stream, k: int) -> list:
    """
    Sample k items uniformly from stream of unknown size.
    
    Each item has exactly k/n probability of being selected,
    where n is the total stream size.
    """
    reservoir = []
    
    for i, item in enumerate(stream):
        if i < k:
            # Fill reservoir initially
            reservoir.append(item)
        else:
            # Replace with decreasing probability
            j = random.randint(0, i)
            if j < k:
                reservoir[j] = item
    
    return reservoir
```

### Why It Works (Proof Sketch)
```
For item at position i to be in final sample:
1. Must be selected: probability k/(i+1)
2. Must NOT be replaced by items i+1 to n

P(item i in final) = k/(i+1) × (i+1)/(i+2) × ... × (n-1)/n
                    = k/n

All items have equal probability k/n! ✓
```

---

## Weighted Reservoir Sampling

```python
import random
import math

def weighted_reservoir_sample(stream, k: int) -> list:
    """
    Sample k items with probability proportional to weight.
    Uses Algorithm A-Res.
    """
    reservoir = []  # (key, item) pairs
    
    for item, weight in stream:
        # Generate random key
        key = random.random() ** (1.0 / weight)
        
        if len(reservoir) < k:
            reservoir.append((key, item))
            reservoir.sort(reverse=True)
        elif key > reservoir[-1][0]:
            reservoir[-1] = (key, item)
            reservoir.sort(reverse=True)
    
    return [item for _, item in reservoir]
```

---

## Distributed Reservoir Sampling

```python
def distributed_sample(partitions: list, k: int) -> list:
    """
    Combine reservoir samples from multiple partitions.
    """
    # Each partition samples k items with random keys
    all_samples = []
    for partition in partitions:
        samples = reservoir_sample_with_keys(partition, k)
        all_samples.extend(samples)
    
    # Sort by key, take top k
    all_samples.sort(key=lambda x: x[0], reverse=True)
    return [item for _, item in all_samples[:k]]

def reservoir_sample_with_keys(stream, k: int):
    """Sample with random keys for distributed merging."""
    reservoir = []
    
    for i, item in enumerate(stream):
        key = random.random()
        if i < k:
            reservoir.append((key, item))
        else:
            min_key = min(r[0] for r in reservoir)
            if key > min_key:
                reservoir = [(k, v) for k, v in reservoir if k != min_key]
                reservoir.append((key, item))
    
    return reservoir
```

---

## Use Cases

### 1. Log Sampling
```python
def sample_logs(log_stream, sample_size=1000):
    """Sample representative logs for analysis."""
    return reservoir_sample(log_stream, sample_size)
```

### 2. A/B Test User Selection
```python
def select_test_users(user_stream, test_group_size):
    """Randomly select users for A/B test."""
    return reservoir_sample(user_stream, test_group_size)
```

### 3. Database Sampling
```sql
-- PostgreSQL: Sample 1% of rows
SELECT * FROM large_table TABLESAMPLE SYSTEM (1);

-- Uses reservoir sampling internally for BERNOULLI
SELECT * FROM large_table TABLESAMPLE BERNOULLI (1);
```

### 4. Approximate Queries
```python
def approximate_median(stream, sample_size=10000):
    """Estimate median using sampled data."""
    sample = reservoir_sample(stream, sample_size)
    sample.sort()
    return sample[len(sample) // 2]
```

---

## Variations

| Variant | Use Case |
|---------|----------|
| **Algorithm R** | Basic uniform sampling |
| **Algorithm L** | Optimized (fewer random calls) |
| **A-Res** | Weighted sampling |
| **A-ExpJ** | Weighted with exponential jumps |
| **Min-wise** | Distributed sampling |

---

## Complexity

```
Time: O(n) - single pass
Space: O(k) - reservoir size only

For Algorithm L (optimized):
- Skips items that won't be selected
- Time: O(k(1 + log(n/k)))
```

---

## Real-World Applications

| System | Usage |
|--------|-------|
| **Spark** | Sample RDD data |
| **Flink** | Stream sampling |
| **BigQuery** | Approximate queries |
| **Monitoring** | Sample metrics/logs |
| **ML** | Training data selection |

---

## Interview Tips

When discussing Reservoir Sampling:
1. Explain the single-pass, O(k) memory constraint
2. Walk through Algorithm R
3. Explain why probability is k/n
4. Mention weighted and distributed variants
5. Give practical use cases (logs, A/B tests)

