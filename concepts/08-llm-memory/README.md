# LLM Memory Systems

This section covers memory and attention mechanisms used in Large Language Models to handle context, improve efficiency, and enable long-range understanding.

## Topics Covered

### Attention Mechanisms
- **[Attention Mechanisms](attention-mechanisms.md)** - Self-attention, cross-attention, and their variants
- **[Sliding Window Attention](sliding-window-attention.md)** - Efficient attention for long sequences
- **[Flash Attention](../07-llm-optimization/flash-attention.md)** - Memory-efficient attention computation

### Memory Architectures
- **[Vector Databases](vector-databases.md)** - External memory for retrieval-augmented generation
- **[Hierarchical Memory](hierarchical-memory.md)** - Multi-level memory systems
- **[Working Memory Buffers](working-memory-buffers.md)** - Short-term context management

### Retrieval and Context
- **[Retrieval-Augmented Memory](retrieval-augmented-memory.md)** - Combining retrieval with generation
- **[KV-Cache](../07-llm-optimization/kv-cache.md)** - Caching attention states for inference

## Key Concepts

### The Memory Challenge
```
LLMs face fundamental memory limitations:

1. Context window: Limited tokens (4K-128K)
2. Attention: O(n²) complexity
3. KV Cache: Memory grows with sequence length
4. Long-term: No persistent memory between sessions

Solutions:
- Efficient attention (sliding window, sparse)
- External memory (vector DBs, RAG)
- Compression (summarization, distillation)
- Caching (KV cache optimization)
```

### Memory Hierarchy
```
┌─────────────────────────────────────────────────────────────────┐
│                    LLM Memory Hierarchy                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Immediate: Attention (current context)                        │
│   ↓                                                              │
│   Short-term: KV Cache (recent tokens)                          │
│   ↓                                                              │
│   Working: Conversation buffer                                  │
│   ↓                                                              │
│   Long-term: Vector database / RAG                              │
│   ↓                                                              │
│   Persistent: Fine-tuned knowledge                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Related Optimizations

- [Flash Attention](../07-llm-optimization/flash-attention.md) - IO-aware attention
- [KV-Cache](../07-llm-optimization/kv-cache.md) - MQA, GQA, PagedAttention
- [Sparse MoE](../07-llm-optimization/sparse-moe.md) - Conditional computation

