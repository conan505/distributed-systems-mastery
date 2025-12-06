# KV-Cache (Key-Value Cache)

## What It Is

**KV-Cache** is an optimization technique that caches the Key and Value tensors computed during attention for previously generated tokens, avoiding redundant computation during autoregressive generation.

---

## The Analogy 📝

Think of **taking notes during a lecture**:
- **Without cache**: Re-read all previous notes for each new sentence
- **With cache**: Just reference your existing notes, add new ones

KV-Cache stores "notes" about past tokens so you don't recompute them.

---

## Why It Exists

### The Problem: Quadratic Attention
```
Autoregressive generation without cache:

Token 1: Compute attention over [1]                    → 1 op
Token 2: Compute attention over [1, 2]                 → 2 ops
Token 3: Compute attention over [1, 2, 3]              → 3 ops
...
Token N: Compute attention over [1, 2, ..., N]         → N ops

Total: 1 + 2 + 3 + ... + N = N(N+1)/2 = O(N²)

For 4096 tokens: ~8 million redundant computations!
```

### What KV-Cache Solves:
```
With KV-Cache:

Token 1: Compute K,V for [1], cache them               → 1 op
Token 2: Use cached K,V[1], compute new K,V[2]         → 1 op
Token 3: Use cached K,V[1,2], compute new K,V[3]       → 1 op
...
Token N: Use cached K,V[1..N-1], compute K,V[N]        → 1 op

Total: N operations = O(N)

Speedup: O(N²) → O(N)
```

---

## How It Works

### Attention Without Cache:
```
For each new token:
Q = token × W_q    (query)
K = ALL_tokens × W_k (keys - recomputed!)
V = ALL_tokens × W_v (values - recomputed!)
Output = softmax(Q × K^T) × V
```

### Attention With Cache:
```
┌─────────────────────────────────────────────────────────────────┐
│                     KV-Cache Flow                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Step 1 (token: "The"):                                        │
│   K_cache = [K_the]                                             │
│   V_cache = [V_the]                                             │
│                                                                  │
│   Step 2 (token: "quick"):                                      │
│   K_cache = [K_the, K_quick]                                    │
│   V_cache = [V_the, V_quick]                                    │
│                                                                  │
│   Step 3 (token: "brown"):                                      │
│   K_cache = [K_the, K_quick, K_brown]                           │
│   V_cache = [V_the, V_quick, V_brown]                           │
│                                                                  │
│   Only compute K, V for NEW token each step                     │
│   Reuse cached K, V for previous tokens                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Memory Requirements

### KV-Cache Size:
```
Per token per layer:
- K: hidden_dim × head_dim bytes
- V: hidden_dim × head_dim bytes

For LLaMA-7B:
- Layers: 32
- Heads: 32
- Head dim: 128
- FP16: 2 bytes

Per token: 32 × 32 × 128 × 2 × 2 = 524KB

For 4096 tokens: 524KB × 4096 = 2GB just for KV-Cache!
```

### Memory Scaling:
| Model Size | Context Length | KV-Cache Memory |
|------------|----------------|-----------------|
| 7B | 2K | 1 GB |
| 7B | 8K | 4 GB |
| 70B | 2K | 10 GB |
| 70B | 8K | 40 GB |

---

## Optimization Techniques

### 1. Multi-Query Attention (MQA)
```
Standard: Each head has own K, V
MQA: All heads share single K, V

Memory reduction: num_heads × (down to 1/num_heads)
Used by: PaLM, Falcon
```

### 2. Grouped-Query Attention (GQA)
```
Middle ground:
- 32 query heads
- 8 key-value heads (4 Q heads share each KV)

Memory: 1/4 of standard multi-head attention
Used by: LLaMA-2, Mistral
```

### 3. Sliding Window Attention
```
Only cache last W tokens
Older tokens discarded

Memory: O(W) instead of O(N)
Trade-off: Can't attend to distant context
Used by: Mistral (W=4096)
```

### 4. PagedAttention (vLLM)
```
┌─────────────────────────────────────────────────────────────────┐
│                   PagedAttention                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Problem: Fixed KV-cache allocation wastes memory              │
│   - Request 1: Uses 100 tokens, allocated 2048                  │
│   - 1948 tokens wasted!                                         │
│                                                                  │
│   Solution: Allocate in pages (like OS memory)                  │
│   - 16-token pages allocated on demand                          │
│   - Non-contiguous storage                                       │
│   - 95%+ memory utilization                                     │
│                                                                  │
│   Benefit: 2-4x more concurrent requests                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## When to Optimize

### ✅ Focus on KV-Cache When:
- Long context applications (RAG, chat)
- High concurrency serving
- Memory-constrained deployment
- Batch inference

### Trade-offs:
- MQA/GQA: Slight quality loss for big memory savings
- Sliding window: Loses long-range dependencies
- Quantization: 4-bit KV-cache (INT4) reduces memory further

---

## Real-World Usage

| System | Optimization |
|--------|--------------|
| **vLLM** | PagedAttention |
| **TensorRT-LLM** | FP8 KV-cache |
| **Mistral** | Sliding window + GQA |
| **LLaMA-2** | Grouped-query attention |

---

## Interview Tips

When discussing KV-Cache:
1. Explain the O(N²) → O(N) speedup
2. Calculate memory requirements
3. Know MQA vs GQA trade-offs
4. Mention PagedAttention for serving
5. Discuss sliding window limitations

