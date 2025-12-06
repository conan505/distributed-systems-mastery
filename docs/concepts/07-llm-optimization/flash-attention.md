# Flash Attention

## What It Is

**Flash Attention** is a memory-efficient and faster implementation of attention that avoids materializing the full attention matrix by computing attention in blocks and fusing operations.

---

## The Analogy 📖

Think of **reading a book**:
- **Standard attention**: Photocopy every page, spread on floor, compare all at once
- **Flash Attention**: Read one chapter at a time, take notes, move to next

Flash Attention processes data in chunks that fit in fast memory (SRAM) instead of slow memory (HBM).

---

## Why It Exists

### The Problem: Memory Bottleneck
```
Standard attention for sequence length N:
1. Compute Q × K^T → N × N matrix (4GB for 32K seq!)
2. Apply softmax → another N × N matrix
3. Multiply by V → final output

Memory: O(N²)
For 32K context: 32K × 32K × 2 bytes = 2GB per head!
```

### GPU Memory Hierarchy:
```
┌─────────────────────────────────────────────────────────────────┐
│                   GPU Memory Hierarchy                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   SRAM (On-chip)          HBM (Off-chip)                        │
│   ┌─────────────┐         ┌─────────────────┐                   │
│   │  20 MB      │         │    80 GB        │                   │
│   │  19 TB/s    │  ◀────▶ │    2 TB/s       │                   │
│   │  (FAST!)    │         │   (10x slower)  │                   │
│   └─────────────┘         └─────────────────┘                   │
│                                                                  │
│   Standard attention: Moves N² data through HBM multiple times │
│   Flash attention: Keeps data in SRAM as much as possible       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### What Flash Attention Solves:
- **Memory efficiency** - O(N) instead of O(N²)
- **Speed** - 2-4x faster training
- **Longer contexts** - 16x longer sequences
- **Exact computation** - No approximation

---

## How It Works

### Core Ideas:

**1. Tiling**: Process attention in blocks
```
Instead of:     Q × K^T (full N×N matrix)

Do:             For each block of Q:
                  For each block of K:
                    Compute partial attention
                    Update running statistics
```

**2. Recomputation**: Don't store intermediate results
```
Forward pass: Compute output, don't save attention matrix
Backward pass: Recompute attention on-the-fly

Memory saved: O(N²) → O(N)
Extra compute: ~20% more FLOPS (worth it!)
```

**3. Kernel Fusion**: One GPU kernel for all operations
```
Standard: 
  Load Q, K → Compute QK^T → Store → Load → Softmax → Store → Load → ×V

Flash:
  Load Q, K, V blocks → Compute everything → Store output only
```

### Algorithm Sketch:
```
┌─────────────────────────────────────────────────────────────────┐
│                   Flash Attention Algorithm                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   For each block Q_i of queries:                                │
│       Initialize: running_max = -∞, running_sum = 0            │
│       For each block K_j, V_j of keys/values:                   │
│           1. Load Q_i, K_j, V_j to SRAM                         │
│           2. Compute S_ij = Q_i × K_j^T                         │
│           3. Compute local_max = max(S_ij)                      │
│           4. Compute local_exp = exp(S_ij - local_max)          │
│           5. Update running statistics                          │
│           6. Update output: O_i += local_exp × V_j              │
│       Normalize O_i by running_sum                              │
│       Store O_i to HBM                                          │
│                                                                  │
│   Key insight: Online softmax computation                       │
│   (update max and sum incrementally)                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Performance Gains

### Memory:
| Sequence Length | Standard | Flash Attention |
|-----------------|----------|-----------------|
| 2K | 16 MB | 2 MB |
| 8K | 256 MB | 8 MB |
| 32K | 4 GB | 32 MB |
| 128K | 64 GB | 128 MB |

### Speed (A100):
| Operation | Standard | Flash-2 |
|-----------|----------|---------|
| Forward | 1x | 2-4x faster |
| Backward | 1x | 2-4x faster |
| Training throughput | 1x | 2-3x more |

---

## Flash Attention Versions

| Version | Features |
|---------|----------|
| **Flash-1** | Basic tiling and recomputation |
| **Flash-2** | Better parallelism, more hardware |
| **Flash-3** | Hopper GPU optimizations (H100) |
| **FlashDecoding** | Optimized for inference |

---

## When to Use

### ✅ Always Use When:
- Training transformers (no reason not to)
- Long-context inference
- Memory-constrained situations
- Need faster training

### Implementation:
```python
# PyTorch 2.0+ (built-in)
from torch.nn.functional import scaled_dot_product_attention
# Automatically uses Flash Attention when available

# Explicit Flash Attention
from flash_attn import flash_attn_func
output = flash_attn_func(q, k, v, causal=True)
```

---

## Compatibility

| Framework | Support |
|-----------|---------|
| **PyTorch 2.0+** | Built-in (SDPA) |
| **Transformers** | Automatic with `attn_implementation="flash_attention_2"` |
| **vLLM** | Default |
| **TensorRT-LLM** | Default |

---

## Interview Tips

When discussing Flash Attention:
1. Explain the GPU memory hierarchy (SRAM vs HBM)
2. Describe tiling and recomputation trade-off
3. Emphasize O(N²) → O(N) memory reduction
4. Mention it's exact (not approximate)
5. Know it's now default in PyTorch 2.0

