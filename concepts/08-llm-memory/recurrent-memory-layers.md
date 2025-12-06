# Recurrent Memory Layers

## What It Is

**Recurrent Memory Layers** are components that add RNN-like state propagation to transformer architectures, enabling efficient processing of long sequences by maintaining a compressed memory state across segments.

---

## The Analogy 📝

Think of **taking notes while reading a textbook**:
- Don't remember every word
- Keep running summary (state)
- Update notes with each chapter
- Final notes capture key concepts
- Memory compresses as you go

---

## Why It Exists

### Transformer vs RNN Trade-off
```
Pure Transformer:
+ Parallel processing
+ Direct long-range access
- O(n²) complexity
- Limited context window

Pure RNN:
+ O(n) complexity
+ Unbounded context (theoretically)
- Sequential processing
- Information bottleneck

Recurrent Memory Layers: Best of both!
- Parallel within segments
- State propagation between segments
- Unbounded context
- Compressed memory
```

---

## Architectures

### 1. Transformer-XL Style Recurrence

```
Segment-level recurrence with caching:

┌─────────────────────────────────────────────────────────────────┐
│              Transformer-XL Memory Flow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Segment 1          Segment 2          Segment 3               │
│   [a b c d]          [e f g h]          [i j k l]              │
│       │                  │                  │                   │
│       ▼                  ▼                  ▼                   │
│   [Layer 1] ─────►  [Layer 1] ─────►  [Layer 1]               │
│       │                  │                  │                   │
│     memory            memory             memory                 │
│       │                  │                  │                   │
│       ▼                  ▼                  ▼                   │
│   [Layer 2] ─────►  [Layer 2] ─────►  [Layer 2]               │
│                                                                  │
│   Memory = cached hidden states from previous segment           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
class TransformerXLLayer(nn.Module):
    def __init__(self, d_model, n_heads, mem_len):
        super().__init__()
        self.attention = MultiHeadAttention(d_model, n_heads)
        self.ffn = FeedForward(d_model)
        self.mem_len = mem_len
        self.memory = None
    
    def forward(self, x):
        # Concatenate memory with current input
        if self.memory is not None:
            extended_context = torch.cat([self.memory, x], dim=1)
        else:
            extended_context = x
        
        # Attention with extended context
        # Query from current, Key/Value from extended
        attn_out = self.attention(
            query=x,
            key=extended_context,
            value=extended_context
        )
        
        output = self.ffn(attn_out)
        
        # Update memory with current hidden states
        self.memory = output[:, -self.mem_len:].detach()
        
        return output
```

### 2. Block Recurrent Transformer

```python
class RecurrentBlock(nn.Module):
    """RNN-style state update within transformer."""
    
    def __init__(self, d_model, state_size):
        super().__init__()
        self.state_size = state_size
        
        # State update network
        self.state_gate = nn.Linear(d_model + state_size, state_size)
        self.state_update = nn.Linear(d_model + state_size, state_size)
        
        # Output projection
        self.output_proj = nn.Linear(state_size, d_model)
    
    def forward(self, x, prev_state):
        batch_size = x.size(0)
        
        if prev_state is None:
            prev_state = torch.zeros(batch_size, self.state_size)
        
        # Process each position, updating state
        outputs = []
        state = prev_state
        
        for t in range(x.size(1)):
            x_t = x[:, t, :]
            
            # Combine input with state
            combined = torch.cat([x_t, state], dim=-1)
            
            # Gated state update (LSTM-like)
            gate = torch.sigmoid(self.state_gate(combined))
            update = torch.tanh(self.state_update(combined))
            state = gate * state + (1 - gate) * update
            
            # Output
            output = self.output_proj(state)
            outputs.append(output)
        
        return torch.stack(outputs, dim=1), state
```

### 3. RWKV (Receptance Weighted Key Value)

```
Linear attention with RNN-like computation:

Traditional attention: softmax(QK^T/√d) V → O(n²)
RWKV: linear recurrence → O(n)

Key insight: Reformulate attention as recurrent computation
Each position updates a running state
Final output computed from state + current input
```

```python
class RWKVLayer(nn.Module):
    """RWKV: RNN-like efficient transformer."""
    
    def __init__(self, d_model):
        super().__init__()
        self.time_mix = nn.Parameter(torch.zeros(d_model))
        self.key = nn.Linear(d_model, d_model)
        self.value = nn.Linear(d_model, d_model)
        self.receptance = nn.Linear(d_model, d_model)
        self.output = nn.Linear(d_model, d_model)
    
    def forward(self, x, state=None):
        batch, seq_len, dim = x.shape
        
        if state is None:
            state = torch.zeros(batch, dim, device=x.device)
        
        outputs = []
        for t in range(seq_len):
            # Time mixing with previous state
            x_t = x[:, t]
            mixed = x_t * self.time_mix + state * (1 - self.time_mix)
            
            # Compute components
            k = self.key(mixed)
            v = self.value(mixed)
            r = torch.sigmoid(self.receptance(mixed))
            
            # Update state and compute output
            state = k * v  # Simplified; actual RWKV is more complex
            out = r * self.output(state)
            outputs.append(out)
        
        return torch.stack(outputs, dim=1), state
```

---

## Comparison

| Model | Attention | Memory | Complexity |
|-------|-----------|--------|------------|
| **Transformer-XL** | Segment recurrence | Cached states | O(n × m) |
| **Compressive** | + Compression | Compressed states | O(n × m) |
| **RWKV** | Linear/RNN | Running state | O(n) |
| **RetNet** | Decay-based | Recurrent state | O(n) |
| **Mamba** | Selective SSM | State space | O(n) |

---

## Benefits of Recurrent Memory

```
1. Unbounded context (theoretically)
   - State carries information from arbitrarily far

2. Constant memory per step
   - Fixed state size regardless of sequence length

3. Efficient inference
   - O(1) per token after initial processing

4. Natural streaming
   - Process tokens as they arrive
   - No need to re-encode history
```

---

## Trade-offs

```
Pros:
+ Linear complexity
+ Infinite context (in theory)
+ Efficient inference
+ Streaming capable

Cons:
- Information bottleneck (state size)
- Sequential processing
- Harder to train
- May lose fine-grained details
```

---

## Use Cases

```
1. Very long sequences
   - Books, code repositories

2. Streaming applications
   - Real-time translation
   - Live transcription

3. Resource-constrained deployment
   - Mobile, edge devices

4. Continuous interaction
   - Long conversations
   - Session-persistent memory
```

---

## Interview Tips

When discussing Recurrent Memory:
1. Explain transformer context limitations
2. Describe segment-level recurrence
3. Know RWKV/RetNet for linear complexity
4. Discuss information bottleneck trade-off
5. Compare with pure attention approaches

