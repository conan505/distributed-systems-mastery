# Long-Context Attention

## What It Is

**Long-Context Attention** refers to techniques that enable transformer models to handle sequences far beyond their original training context length, from 4K to 100K+ tokens, while maintaining quality and efficiency.

---

## The Analogy 📚

Think of **reading a novel vs a tweet**:
- Tweet: See everything at once (standard attention)
- Novel: Need strategies to track plot, characters
- Long-context: Techniques to "read" entire books
- Must remember chapter 1 while reading chapter 50

---

## Why It Exists

### Context Window Limitations
```
Original GPT-3: 2K tokens (~1,500 words)
GPT-4: 8K-128K tokens
Claude: 100K-200K tokens

Use cases requiring long context:
- Entire codebases
- Legal documents
- Book summarization
- Multi-document QA
- Long conversations

Challenge: O(n²) attention doesn't scale!
```

---

## Key Techniques

### 1. Position Encoding Extensions

```
RoPE (Rotary Position Embeddings):
- Encodes position as rotation
- Naturally extends to longer sequences

Position Interpolation:
- Scale positions to fit trained range
- Position 10000 → treated as position 2048

NTK-Aware Scaling:
- Adjust RoPE base frequency
- Better extrapolation than interpolation
```

```python
def apply_rope_scaling(positions, original_max=4096, target_max=32768):
    """Position interpolation for extended context."""
    scale_factor = original_max / target_max
    scaled_positions = positions * scale_factor
    return scaled_positions

def ntk_aware_scaling(base=10000, scale=4):
    """NTK-aware RoPE scaling."""
    new_base = base * (scale ** (dim / (dim - 2)))
    return new_base
```

### 2. Attention Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│              Long-Context Attention Patterns                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Full Attention (Original):                                    │
│   ████████████  Every token attends to all                      │
│   ████████████  O(n²) - doesn't scale                          │
│   ████████████                                                  │
│                                                                  │
│   Sliding Window:                                               │
│   ███░░░░░░░░░  Local attention only                           │
│   ░███░░░░░░░░  O(n×w) - scales linearly                       │
│   ░░███░░░░░░░                                                  │
│                                                                  │
│   Sparse + Global:                                              │
│   █░█░█░█░█░█░  Sparse global + local                          │
│   ░███░░░░░░░░  Best of both worlds                            │
│   █░███░░░░░░░                                                  │
│                                                                  │
│   Dilated:                                                      │
│   █░░█░░█░░█░░  Increasing strides per layer                   │
│   ░█░░█░░█░░█░  Exponential receptive field                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Memory-Efficient Implementations

```python
# Flash Attention: Fused CUDA kernel
from flash_attn import flash_attn_func

# Memory: O(n) instead of O(n²)
# Speed: 2-4x faster for long sequences
output = flash_attn_func(q, k, v, causal=True)

# Ring Attention: Distributed across devices
# Each device holds a chunk, passes KV in ring
# Enables context > single GPU memory
```

---

## Model-Specific Approaches

| Model | Max Context | Technique |
|-------|-------------|-----------|
| **GPT-4** | 128K | Unknown (proprietary) |
| **Claude** | 200K | Likely sliding window + sparse |
| **Llama 2 Long** | 32K | Position interpolation |
| **Mistral** | 32K | Sliding window (4K) + layers |
| **LongLoRA** | 100K+ | Shifted sparse attention |
| **YaRN** | 128K | NTK-aware + temperature |

---

## YaRN (Yet another RoPE extensioN)

```python
def yarn_scaling(dim, original_max, target_max, beta_fast=32, beta_slow=1):
    """YaRN: Combines interpolation strategies."""
    scale = target_max / original_max
    
    # Different scaling for different frequency bands
    low_freq_factor = 1.0  # Keep low frequencies
    high_freq_factor = scale  # Interpolate high frequencies
    
    # Smooth transition between strategies
    freqs = compute_rope_frequencies(dim)
    scaled_freqs = []
    
    for freq in freqs:
        if freq < beta_fast:
            # Low frequency: no scaling
            scaled_freqs.append(freq)
        elif freq > beta_slow:
            # High frequency: full interpolation
            scaled_freqs.append(freq * scale)
        else:
            # Smooth interpolation
            t = (freq - beta_fast) / (beta_slow - beta_fast)
            scaled_freqs.append(freq * (1 + (scale - 1) * t))
    
    return scaled_freqs
```

---

## Chunked Processing

```python
def process_long_document(model, document, chunk_size=4096, overlap=512):
    """Process document in overlapping chunks."""
    chunks = []
    for i in range(0, len(document), chunk_size - overlap):
        chunk = document[i:i + chunk_size]
        chunks.append(chunk)
    
    # Process each chunk
    chunk_outputs = []
    for chunk in chunks:
        output = model(chunk)
        chunk_outputs.append(output)
    
    # Merge overlapping regions
    return merge_chunk_outputs(chunk_outputs, overlap)
```

---

## Quality Considerations

```
Long-context challenges:

1. "Lost in the Middle"
   - Models focus on beginning and end
   - Middle content often ignored
   - Solution: Strategic placement, multiple passes

2. Attention Dilution
   - More tokens = less attention per token
   - Important info gets less weight
   - Solution: Hierarchical attention, retrieval

3. Computational Cost
   - Even O(n) is slow for 100K tokens
   - Memory bandwidth limited
   - Solution: Caching, streaming
```

---

## Best Practices

```
1. Chunk strategically
   - Respect document structure
   - Overlap to maintain context

2. Place important info wisely
   - Beginning and end get most attention
   - Critical info not in middle

3. Use retrieval for very long docs
   - RAG often beats brute-force context
   - Retrieve relevant sections

4. Test at target length
   - Models degrade at untrained lengths
   - Evaluate on realistic documents
```

---

## Interview Tips

When discussing Long-Context:
1. Explain O(n²) attention problem
2. Describe position encoding extensions (RoPE, NTK)
3. Know efficient attention patterns
4. Mention "lost in the middle" problem
5. Discuss when RAG might be better

