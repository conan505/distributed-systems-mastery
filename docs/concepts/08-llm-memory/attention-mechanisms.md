# Attention Mechanisms

## What It Is

**Attention** is a mechanism that allows neural networks to focus on relevant parts of the input when producing each output, enabling models to capture long-range dependencies and contextual relationships.

---

## The Analogy 🔍

Think of **reading a book to answer a question**:
- You don't read every word equally
- Focus on relevant paragraphs (attention)
- Different questions focus on different parts
- Weight importance based on relevance to query

---

## Why It Exists

### Before Attention: RNN Limitations
```
RNN processing "The cat sat on the mat":

Hidden state carries ALL information
h1 → h2 → h3 → h4 → h5 → h6

Problems:
- Information bottleneck
- Long-range dependencies lost
- Sequential processing (slow)

Solution: Attend to all positions directly
```

---

## Self-Attention (Transformer)

### The Core Formula
```
Attention(Q, K, V) = softmax(QK^T / √d_k) × V

Q = Query: "What am I looking for?"
K = Key: "What do I contain?"
V = Value: "What information do I provide?"
```

### Visual Representation
```
┌─────────────────────────────────────────────────────────────────┐
│                    Self-Attention                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Input: [The] [cat] [sat] [on] [the] [mat]                     │
│                                                                  │
│   For "sat":                                                    │
│   Q_sat • K_The  = 0.1  (low attention)                        │
│   Q_sat • K_cat  = 0.5  (high - subject!)                      │
│   Q_sat • K_sat  = 0.2  (self)                                 │
│   Q_sat • K_mat  = 0.2  (related)                              │
│                                                                  │
│   Output_sat = 0.1×V_The + 0.5×V_cat + 0.2×V_sat + 0.2×V_mat  │
│                                                                  │
│   "sat" now contains context from "cat"!                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation

```python
import torch
import torch.nn.functional as F
import math

def scaled_dot_product_attention(
    query: torch.Tensor,
    key: torch.Tensor,
    value: torch.Tensor,
    mask: torch.Tensor = None
) -> torch.Tensor:
    """
    Args:
        query: (batch, heads, seq_len, d_k)
        key: (batch, heads, seq_len, d_k)
        value: (batch, heads, seq_len, d_v)
        mask: Optional attention mask
    Returns:
        Attention output: (batch, heads, seq_len, d_v)
    """
    d_k = query.size(-1)
    
    # Compute attention scores
    scores = torch.matmul(query, key.transpose(-2, -1)) / math.sqrt(d_k)
    
    # Apply mask (for causal attention)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))
    
    # Softmax to get attention weights
    attention_weights = F.softmax(scores, dim=-1)
    
    # Apply attention to values
    output = torch.matmul(attention_weights, value)
    
    return output
```

---

## Multi-Head Attention

```
Multiple attention "perspectives":

┌─────────────────────────────────────────────────────────────────┐
│                  Multi-Head Attention                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Head 1: Focuses on subject-verb relationships                │
│   Head 2: Focuses on adjective-noun relationships              │
│   Head 3: Focuses on positional patterns                       │
│   Head 4: Focuses on semantic similarity                       │
│   ...                                                           │
│                                                                  │
│   Each head learns different patterns!                          │
│   Concat(head1, head2, ..., headH) × W_O                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model: int, num_heads: int):
        super().__init__()
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
    
    def forward(self, x, mask=None):
        batch_size, seq_len, _ = x.shape
        
        # Project and reshape for multi-head
        Q = self.W_q(x).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(x).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(x).view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        
        # Attention
        attn_output = scaled_dot_product_attention(Q, K, V, mask)
        
        # Concat and project
        output = attn_output.transpose(1, 2).contiguous().view(batch_size, seq_len, -1)
        return self.W_o(output)
```

---

## Attention Variants

### 1. Causal (Autoregressive)
```
For language models - can only attend to past:

    The  cat  sat  mat
The  ✓    ✗    ✗    ✗
cat  ✓    ✓    ✗    ✗
sat  ✓    ✓    ✓    ✗
mat  ✓    ✓    ✓    ✓
```

### 2. Cross-Attention
```
Attend to different sequence (encoder-decoder):

Encoder output: [French sentence]
Decoder query: [English word being generated]

Each English word attends to relevant French words
```

### 3. Sparse Attention
```
Don't attend to all positions:
- Local: Nearby tokens only
- Strided: Every nth token
- Block: Blocks of tokens

Reduces O(n²) → O(n√n) or O(n)
```

---

## Complexity

```
Standard Self-Attention:
Time: O(n² × d)  - n² attention scores
Space: O(n² + nd) - attention matrix + values

Problematic for long sequences:
- n = 4096: 16M attention entries
- n = 100K: 10B attention entries!

Solutions:
- Flash Attention (memory-efficient)
- Sparse Attention (fewer connections)
- Linear Attention (approximate)
```

---

## Key Innovations

| Technique | Description | Used In |
|-----------|-------------|---------|
| **MQA** | Shared K,V heads | PaLM, Falcon |
| **GQA** | Grouped K,V heads | Llama 2 |
| **Flash Attention** | IO-aware algorithm | Most modern LLMs |
| **Sliding Window** | Local attention | Mistral |
| **RoPE** | Rotary position encoding | Llama, Mistral |

---

## Interview Tips

When discussing Attention:
1. Explain Q, K, V intuition
2. Walk through scaled dot-product formula
3. Describe multi-head benefits
4. Know O(n²) complexity issue
5. Mention efficiency improvements (Flash, Sparse)

