# Sliding Window Attention

## What It Is

**Sliding Window Attention** is an efficient attention mechanism where each token only attends to a fixed-size local window of nearby tokens, reducing quadratic complexity to linear while maintaining the ability to propagate information across long sequences.

---

## The Analogy 🔭

Think of **passing messages in a line**:
- You can only whisper to 5 people on each side
- But they can whisper to their neighbors too
- After several rounds, your message reaches everyone
- Local communication enables global information flow

---

## Why It Exists

### The Full Attention Problem
```
Full attention: Every token attends to every token

Sequence length: 4096 tokens
Attention matrix: 4096 × 4096 = 16M entries
Memory: ~64MB per layer (FP32)

Sequence length: 100K tokens
Attention matrix: 10B entries
Memory: ~40GB per layer!

Problem: Quadratic scaling O(n²)
```

### Sliding Window Solution
```
Window size: 4096 tokens
Each token attends to 4096 neighbors (not all)

Sequence length: 100K tokens
Attention per token: 4096 entries
Total: 100K × 4096 = 409M entries (not 10B!)

Memory: O(n × w) where w = window size
```

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                  Sliding Window Attention                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Token positions: 0  1  2  3  4  5  6  7  8  9  10 11 12      │
│   Window size: 4                                                 │
│                                                                  │
│   Token 6 attends to:                                           │
│   [  ][  ][ 2][ 3][ 4][ 5][ 6][ 7][ 8][ 9][  ][  ][  ]        │
│              └──────────┬──────────┘                            │
│                    window = 4                                   │
│                                                                  │
│   Information propagation across layers:                        │
│   Layer 1: Token 0 info reaches token 4                        │
│   Layer 2: Token 0 info reaches token 8                        │
│   Layer L: Token 0 info reaches token L × w                    │
│                                                                  │
│   32 layers × 4096 window = 131K effective context!             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation

```python
import torch
import torch.nn.functional as F

def sliding_window_attention(
    query: torch.Tensor,
    key: torch.Tensor,
    value: torch.Tensor,
    window_size: int
) -> torch.Tensor:
    """
    Efficient sliding window attention.
    
    Args:
        query, key, value: (batch, heads, seq_len, d)
        window_size: Number of tokens to attend to on each side
    """
    batch, heads, seq_len, d = query.shape
    
    # Create attention scores
    scores = torch.matmul(query, key.transpose(-2, -1)) / (d ** 0.5)
    
    # Create sliding window mask
    mask = torch.ones(seq_len, seq_len, device=query.device)
    for i in range(seq_len):
        start = max(0, i - window_size)
        end = min(seq_len, i + window_size + 1)
        mask[i, :start] = 0
        mask[i, end:] = 0
    
    # For causal models, also mask future
    causal_mask = torch.tril(torch.ones(seq_len, seq_len, device=query.device))
    mask = mask * causal_mask
    
    # Apply mask
    scores = scores.masked_fill(mask == 0, float('-inf'))
    
    # Softmax and apply to values
    attn_weights = F.softmax(scores, dim=-1)
    output = torch.matmul(attn_weights, value)
    
    return output
```

---

## Mistral's Implementation

```
Mistral-7B uses sliding window with:
- Window size: 4096 tokens
- 32 layers
- Effective attention span: 4096 × 32 = 131K tokens

With Rolling Buffer KV Cache:
- Only store last 4096 KV entries
- Constant memory regardless of sequence length
- Position i writes to cache[i mod window_size]
```

```python
class RollingKVCache:
    def __init__(self, window_size: int, num_heads: int, head_dim: int):
        self.window_size = window_size
        self.cache_k = torch.zeros(window_size, num_heads, head_dim)
        self.cache_v = torch.zeros(window_size, num_heads, head_dim)
    
    def update(self, position: int, k: torch.Tensor, v: torch.Tensor):
        cache_pos = position % self.window_size
        self.cache_k[cache_pos] = k
        self.cache_v[cache_pos] = v
    
    def get(self, current_pos: int):
        # Get relevant window of cache
        start = max(0, current_pos - self.window_size + 1)
        # Handle wraparound...
        return self.cache_k, self.cache_v
```

---

## Variants

### 1. Local Attention (Basic Sliding Window)
```
Each token attends to fixed local window
Used in: Longformer, BigBird (local part)
```

### 2. Dilated Sliding Window
```
Attend to every nth token in window
Increases effective receptive field
Layer 1: stride 1, Layer 2: stride 2, ...
```

### 3. Global + Local (Longformer)
```
Most tokens: Local attention (window)
Special tokens [CLS]: Global attention (all tokens)

Enables classification while keeping efficiency
```

### 4. Block Sparse
```
Divide into blocks, attend within blocks + some cross-block

BigBird:
- Local window attention
- Random attention (some random blocks)
- Global attention (first few tokens)
```

---

## Complexity Comparison

```
╔════════════════════════════════════════════════╗
║  Method           │  Time      │  Space        ║
╠════════════════════════════════════════════════╣
║  Full Attention   │  O(n²)     │  O(n²)        ║
║  Sliding Window   │  O(n × w)  │  O(n × w)     ║
║  Dilated Window   │  O(n × w)  │  O(n × w)     ║
║  Longformer       │  O(n × w)  │  O(n × w)     ║
╚════════════════════════════════════════════════╝

n = sequence length, w = window size
```

---

## When to Use

### ✅ Good Fit:
- Long document processing
- Streaming/real-time generation
- Memory-constrained inference
- Tasks with locality (nearby context matters)

### ❌ Avoid When:
- Short sequences (full attention is fine)
- Tasks requiring global reasoning
- Need exact long-range dependencies

---

## Trade-offs

```
Pros:
+ Linear memory scaling
+ Constant KV cache size
+ Enables very long sequences

Cons:
- Information must propagate through layers
- Some long-range patterns harder to learn
- May miss direct long-distance connections
```

---

## Real-World Usage

| Model | Window Size | Max Sequence |
|-------|-------------|--------------|
| **Mistral 7B** | 4096 | 32K+ |
| **Longformer** | 512 | 4096 |
| **BigBird** | 64 blocks | 4096 |
| **LongT5** | 256 | 16K |

---

## Interview Tips

When discussing Sliding Window Attention:
1. Explain the O(n²) problem it solves
2. Describe how information propagates across layers
3. Know the memory/quality trade-off
4. Mention Mistral's rolling buffer cache
5. Compare with full attention and sparse variants

