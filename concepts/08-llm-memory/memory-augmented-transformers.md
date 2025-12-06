# Memory-Augmented Transformers

## What It Is

**Memory-Augmented Transformers** are architectures that extend standard transformers with explicit external memory modules, enabling the model to read from and write to persistent memory beyond the context window.

---

## The Analogy 🗄️

Think of a **researcher with a notebook**:
- Brain (model): Limited working memory
- Notebook (external memory): Unlimited storage
- Can write important findings
- Can look up past notes
- Combines reasoning with retrieval

---

## Why It Exists

### Transformer Limitations
```
Standard Transformer:
- Fixed context window
- All "memory" in attention
- No persistent state
- Can't learn from interactions

Memory-Augmented:
- Explicit memory slots
- Read/write operations
- Persists across contexts
- Accumulates knowledge
```

---

## Architectures

### 1. Memory Networks

```
┌─────────────────────────────────────────────────────────────────┐
│                    Memory Network                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Query: "What color is the ball?"                              │
│                                                                  │
│   Memory Slots:                                                 │
│   [M1]: "John picked up the ball"  → attention: 0.3            │
│   [M2]: "The ball is red"          → attention: 0.6 ←          │
│   [M3]: "Mary went to the kitchen" → attention: 0.1            │
│                                                                  │
│   Output: Weighted sum of memories + query                      │
│   Answer: "red"                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
class MemoryNetwork(nn.Module):
    def __init__(self, memory_size, embed_dim):
        super().__init__()
        self.memory = nn.Parameter(torch.randn(memory_size, embed_dim))
        self.query_proj = nn.Linear(embed_dim, embed_dim)
        self.output_proj = nn.Linear(embed_dim * 2, embed_dim)
    
    def forward(self, query):
        # Compute attention over memory
        query_proj = self.query_proj(query)
        attention = F.softmax(query_proj @ self.memory.T, dim=-1)
        
        # Read from memory
        memory_output = attention @ self.memory
        
        # Combine query and memory
        output = self.output_proj(torch.cat([query, memory_output], dim=-1))
        return output
```

### 2. Transformer-XL (Recurrence)

```
Segment-level recurrence:

Segment 1: [a b c d] → hidden states h1
Segment 2: [e f g h] → attend to h1 (cached) + current
Segment 3: [i j k l] → attend to h2 (cached) + current

Memory = cached hidden states from previous segments
Enables unbounded context with fixed memory
```

```python
class TransformerXL(nn.Module):
    def __init__(self, d_model, n_heads, mem_len):
        super().__init__()
        self.mem_len = mem_len
        self.attention = MultiHeadAttention(d_model, n_heads)
        self.memory = None
    
    def forward(self, x):
        if self.memory is not None:
            # Concatenate memory with current input for attention
            mem_and_x = torch.cat([self.memory, x], dim=1)
        else:
            mem_and_x = x
        
        # Attention over extended context
        output = self.attention(x, mem_and_x, mem_and_x)
        
        # Update memory (detach to stop gradients)
        self.memory = mem_and_x[:, -self.mem_len:].detach()
        
        return output
```

### 3. Compressive Transformer

```
Hierarchical memory compression:

Recent tokens: Full attention (high fidelity)
Older tokens: Compressed (summarized)
Ancient tokens: Highly compressed (key points only)

┌────────────────────────────────────────────────────┐
│ [Compressed²] [Compressed] [   Full Memory   ] [x] │
│     16 slots    64 slots       256 slots      curr │
│   Very old      Old            Recent        Now   │
└────────────────────────────────────────────────────┘
```

### 4. Memorizing Transformers

```python
class MemorizingTransformer(nn.Module):
    """kNN-augmented attention with external memory."""
    
    def __init__(self, d_model, memory_size=65536):
        super().__init__()
        self.local_attention = MultiHeadAttention(d_model)
        self.memory_keys = torch.zeros(memory_size, d_model)
        self.memory_values = torch.zeros(memory_size, d_model)
        self.memory_ptr = 0
    
    def forward(self, x):
        # Standard local attention
        local_out = self.local_attention(x, x, x)
        
        # kNN lookup in external memory
        queries = x
        similarities = queries @ self.memory_keys.T
        top_k_indices = similarities.topk(k=32).indices
        
        retrieved_keys = self.memory_keys[top_k_indices]
        retrieved_values = self.memory_values[top_k_indices]
        
        # Attention over retrieved memories
        memory_attention = F.softmax(queries @ retrieved_keys.T, dim=-1)
        memory_out = memory_attention @ retrieved_values
        
        # Combine local and memory outputs
        output = local_out + memory_out
        
        # Write current states to memory
        self._update_memory(x)
        
        return output
```

---

## Read/Write Operations

```
Memory Operations:

READ (Content-based addressing):
1. Generate query from current state
2. Compute similarity with all memory slots
3. Softmax attention over memories
4. Return weighted combination

WRITE (Various strategies):
1. Append: Add new memories
2. Update: Modify existing slots
3. Erase: Remove old/irrelevant memories
4. Compress: Merge similar memories
```

---

## Comparison

| Model | Memory Type | Context Extension |
|-------|-------------|-------------------|
| **Memory Networks** | Explicit slots | Slot count |
| **Transformer-XL** | Hidden state cache | Segment recurrence |
| **Compressive** | Hierarchical compressed | Compression ratio |
| **Memorizing** | kNN retrieval | Database size |
| **RETRO** | Chunk retrieval | Retrieval corpus |

---

## RETRO Architecture

```
Retrieval-Enhanced Transformer:

1. Chunk input into segments
2. For each chunk, retrieve similar chunks from corpus
3. Cross-attend to retrieved chunks
4. Generate output

Benefits:
- Smaller model, larger "memory" (corpus)
- Factual knowledge in retrieval, not params
- Easy to update (modify corpus)
```

---

## Trade-offs

```
Pros:
+ Unbounded context (theoretically)
+ Explicit memory operations
+ Can learn to use memory
+ Persistence across sessions

Cons:
- Added complexity
- Memory management overhead
- Training challenges
- Harder to scale
```

---

## Use Cases

```
1. Long document processing
   - Books, codebases, legal docs

2. Knowledge-intensive tasks
   - QA over large knowledge bases

3. Continual learning
   - Remember past interactions

4. Multi-session conversations
   - Persistent user memory
```

---

## Interview Tips

When discussing Memory-Augmented Transformers:
1. Explain limitations of standard attention
2. Describe read/write operations
3. Know key architectures (Transformer-XL, RETRO)
4. Discuss compression strategies
5. Compare with RAG approaches

