# Local + Global Memory Fusion

## What It Is

**Local + Global Memory Fusion** is an architecture pattern that combines fine-grained local attention with coarse-grained global memory access, enabling efficient processing of long sequences while maintaining both detail and context.

---

## The Analogy 🔍

Think of **reading a newspaper**:
- **Local focus**: Read the current paragraph carefully
- **Global context**: Remember the article headline, section
- **Fusion**: Understand paragraph meaning in context
- Both needed for comprehension

---

## Why It Exists

### Pure Local vs Pure Global Problems
```
LOCAL ONLY (Sliding Window):
+ O(n×w) complexity
- Loses global context
- Can't connect distant concepts

GLOBAL ONLY (Full Attention):
+ Perfect context
- O(n²) complexity
- Memory explosion

FUSION:
+ Efficient O(n×w) + O(n×g)
+ Local detail preserved
+ Global context available
```

---

## Architecture Patterns

### 1. Longformer-Style (Global Tokens)

```
┌─────────────────────────────────────────────────────────────────┐
│                 Longformer Attention Pattern                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [CLS]  tok1  tok2  tok3  tok4  tok5  tok6  tok7  [SEP]       │
│    │      │     │     │     │     │     │     │     │          │
│    ▼      ▼     ▼     ▼     ▼     ▼     ▼     ▼     ▼          │
│   ███    ██░   ░██   ░███  ░░███ ░░░██ ░░░░█ ░░░░█ ███         │
│    │                                                │           │
│    └────── Global attention ────────────────────────┘           │
│                                                                  │
│   [CLS] and [SEP] attend to ALL tokens (global)                │
│   Other tokens: sliding window (local)                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
class LongformerAttention(nn.Module):
    def __init__(self, d_model, window_size, num_global_tokens):
        super().__init__()
        self.window_size = window_size
        self.num_global = num_global_tokens
        self.local_attention = SlidingWindowAttention(d_model, window_size)
        self.global_attention = nn.MultiheadAttention(d_model, num_heads=8)
    
    def forward(self, x, global_mask):
        """
        x: (batch, seq_len, d_model)
        global_mask: (seq_len,) - True for global tokens
        """
        # Split into global and local tokens
        global_tokens = x[:, global_mask]  # CLS, SEP, etc.
        local_tokens = x[:, ~global_mask]
        
        # Local attention within windows
        local_out = self.local_attention(local_tokens)
        
        # Global tokens attend to all
        global_out = self.global_attention(
            query=global_tokens,
            key=x,
            value=x
        )[0]
        
        # All tokens attend to global tokens
        global_context = self.global_attention(
            query=local_tokens,
            key=global_tokens,
            value=global_tokens
        )[0]
        
        # Fuse local and global
        local_out = local_out + global_context
        
        # Reassemble
        output = torch.zeros_like(x)
        output[:, global_mask] = global_out
        output[:, ~global_mask] = local_out
        
        return output
```

### 2. BigBird-Style (Sparse Global)

```
Three attention patterns combined:

1. RANDOM: Sample random token pairs
2. LOCAL: Sliding window
3. GLOBAL: First few tokens attend globally

┌────────────────────────────────────────┐
│   R = Random, L = Local, G = Global    │
│                                        │
│     G G L R . . . . . R                │
│     G G L L R . . . R .                │
│     L L G L L R . R . .                │
│     R L L G L L R . . .                │
│     . R L L G L L R . .                │
│     . . R L L G L L R .                │
│     . . . R L L G L L .                │
│     . . R . R L L G L L                │
│     . R . . . R L L G L                │
│     R . . . . . R L L G                │
│                                        │
└────────────────────────────────────────┘
```

### 3. Hierarchical Memory Fusion

```python
class HierarchicalMemoryFusion(nn.Module):
    """Multi-level memory with local and global components."""
    
    def __init__(self, d_model, local_window, num_memory_tokens):
        super().__init__()
        self.local_window = local_window
        
        # Local attention
        self.local_attn = SlidingWindowAttention(d_model, local_window)
        
        # Global memory tokens (learnable)
        self.memory_tokens = nn.Parameter(
            torch.randn(num_memory_tokens, d_model)
        )
        
        # Cross-attention to memory
        self.memory_read = nn.MultiheadAttention(d_model, num_heads=8)
        self.memory_write = nn.MultiheadAttention(d_model, num_heads=8)
        
        # Fusion layer
        self.fusion = nn.Linear(d_model * 2, d_model)
    
    def forward(self, x):
        batch_size = x.size(0)
        
        # Expand memory for batch
        memory = self.memory_tokens.unsqueeze(0).expand(batch_size, -1, -1)
        
        # Local processing
        local_out = self.local_attn(x)
        
        # Write to global memory (tokens → memory)
        memory, _ = self.memory_write(
            query=memory,
            key=x,
            value=x
        )
        
        # Read from global memory (memory → tokens)
        global_context, _ = self.memory_read(
            query=x,
            key=memory,
            value=memory
        )
        
        # Fuse local and global
        fused = self.fusion(torch.cat([local_out, global_context], dim=-1))
        
        return fused
```

---

## Compression-Based Global Memory

```python
class CompressedGlobalMemory(nn.Module):
    """Compress sequence into global summary tokens."""
    
    def __init__(self, d_model, compression_ratio=8, num_summary=16):
        super().__init__()
        self.compression_ratio = compression_ratio
        
        # Compress local chunks into summary
        self.compressor = nn.Sequential(
            nn.Conv1d(d_model, d_model, kernel_size=compression_ratio, 
                     stride=compression_ratio),
            nn.GELU()
        )
        
        # Cross-attention between local and compressed global
        self.fusion_attn = nn.MultiheadAttention(d_model, num_heads=8)
    
    def forward(self, x):
        # x: (batch, seq_len, d_model)
        
        # Local processing (full attention within chunks)
        chunk_size = self.compression_ratio
        chunks = x.unfold(1, chunk_size, chunk_size)  # (batch, num_chunks, chunk, d)
        
        # Compress to global summaries
        x_transposed = x.transpose(1, 2)  # (batch, d, seq)
        global_summary = self.compressor(x_transposed).transpose(1, 2)
        
        # Each token attends to global summaries
        fused, _ = self.fusion_attn(
            query=x,
            key=global_summary,
            value=global_summary
        )
        
        return x + fused  # Residual
```

---

## Comparison

| Method | Global Mechanism | Complexity |
|--------|-----------------|------------|
| **Longformer** | Special tokens | O(n×w + n×g) |
| **BigBird** | Random + global | O(n×w + n×r) |
| **Performer** | Kernel approximation | O(n×d) |
| **Hierarchical** | Memory tokens | O(n×w + n×m) |
| **Compressive** | Compressed summaries | O(n×w + n/c) |

---

## When to Use

### ✅ Good Fit:
- Long document understanding
- Document QA (need both detail and overview)
- Multi-document summarization
- Code understanding (local syntax, global structure)

### ❌ Avoid When:
- Short sequences (full attention is fine)
- Pure local tasks (no global context needed)
- Extreme latency requirements

---

## Best Practices

```
1. Choose global token strategy
   - Special tokens (CLS) for classification
   - Learned memory for general tasks
   - Compression for very long docs

2. Balance local/global
   - More local for detail-heavy tasks
   - More global for reasoning tasks

3. Layer-wise variation
   - Early layers: more local
   - Later layers: more global

4. Combine with caching
   - Cache global summaries
   - Update incrementally
```

---

## Interview Tips

When discussing Local+Global Fusion:
1. Explain why both are needed
2. Describe Longformer's global tokens
3. Know BigBird's three-pattern approach
4. Discuss complexity benefits
5. Compare with full attention trade-offs

