# Differentiable Neural Computers (DNC)

## What It Is

A **Differentiable Neural Computer** is a neural network architecture with an external memory matrix that can be read from and written to through differentiable attention mechanisms, enabling the network to learn algorithmic tasks requiring memory.

---

## The Analogy 💻

Think of a **computer with RAM**:
- CPU (neural network): Processes information
- RAM (memory matrix): Stores data
- Read/Write heads: Access memory locations
- Pointers: Track where to read/write
- Unlike computers, it's fully differentiable (trainable)

---

## Why It Exists

### Neural Network Memory Limitations
```
Standard RNN/LSTM:
- Fixed-size hidden state
- Information gets compressed/lost
- Can't perform precise lookups

DNC Adds:
- External memory matrix (N × W)
- Content-based addressing
- Temporal linking
- Multiple read/write heads

Enables: Graph traversal, sorting, copying, reasoning
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                DNC Architecture                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Input → [Controller] → Output                                 │
│              ↑↓                                                  │
│   ┌──────────────────────────────────────┐                      │
│   │        Memory Matrix (N × W)         │                      │
│   │   ┌───┬───┬───┬───┬───┬───┬───┐     │                      │
│   │   │   │   │   │   │   │   │   │ W   │ N = memory slots     │
│   │   ├───┼───┼───┼───┼───┼───┼───┤     │ W = word size        │
│   │   │   │   │   │ ✓ │   │   │   │     │                      │
│   │   └───┴───┴───┴───┴───┴───┴───┘     │                      │
│   │          ↑            ↑              │                      │
│   │     Write Head    Read Heads         │                      │
│   └──────────────────────────────────────┘                      │
│                                                                  │
│   Additional State:                                             │
│   - Usage vector (tracks used slots)                            │
│   - Precedence weights (last written)                           │
│   - Temporal link matrix (order of writes)                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Operations

### Memory Reading
```python
def read_memory(memory, read_weights):
    """
    Content-based reading from memory.
    
    memory: (N, W) memory matrix
    read_weights: (N,) attention over memory slots
    returns: (W,) read vector
    """
    # Weighted sum of memory rows
    read_vector = torch.matmul(read_weights, memory)  # (W,)
    return read_vector
```

### Memory Writing
```python
def write_memory(memory, write_weights, erase_vector, add_vector):
    """
    Write to memory with erase and add operations.
    
    memory: (N, W) current memory
    write_weights: (N,) where to write
    erase_vector: (W,) what to forget
    add_vector: (W,) what to add
    """
    # Erase: memory *= (1 - w ⊗ e)
    erase = torch.outer(write_weights, erase_vector)
    memory = memory * (1 - erase)
    
    # Add: memory += w ⊗ a
    add = torch.outer(write_weights, add_vector)
    memory = memory + add
    
    return memory
```

### Addressing Mechanisms

```python
def content_based_addressing(memory, key, strength):
    """Find memory slots similar to key."""
    # Cosine similarity
    similarity = F.cosine_similarity(memory, key.unsqueeze(0), dim=1)
    
    # Sharpen with strength parameter
    weights = F.softmax(strength * similarity, dim=0)
    return weights

def location_based_addressing(prev_weights, shift, sharpen):
    """Shift attention to nearby locations."""
    # Circular convolution for shifting
    shifted = circular_conv(prev_weights, shift)
    
    # Sharpen
    weights = (shifted ** sharpen) / (shifted ** sharpen).sum()
    return weights
```

---

## Temporal Link Matrix

```
Tracks the order of writes for sequential access:

After writing to slots: [3, 1, 4, 2]

Link matrix L[i,j] = "slot j was written after slot i"

L = [       slot 0  1  2  3  4
    slot 0    0    0  0  0  0
    slot 1    0    0  0  0  1   ← 4 after 1
    slot 2    0    0  0  0  0
    slot 3    0    1  0  0  0   ← 1 after 3
    slot 4    0    0  1  0  0   ← 2 after 4
]

Enables: "Read what was written next/previous"
```

---

## Complete DNC Forward Pass

```python
class DNC(nn.Module):
    def __init__(self, input_size, output_size, 
                 memory_size=128, word_size=64, num_reads=4):
        super().__init__()
        self.memory_size = memory_size
        self.word_size = word_size
        self.num_reads = num_reads
        
        # Controller (LSTM)
        controller_size = 256
        self.controller = nn.LSTM(
            input_size + num_reads * word_size,
            controller_size
        )
        
        # Interface (controller → memory operations)
        interface_size = num_reads * word_size + 3 * word_size + \
                        5 * num_reads + 3
        self.interface = nn.Linear(controller_size, interface_size)
        
        # Output
        self.output = nn.Linear(
            controller_size + num_reads * word_size,
            output_size
        )
    
    def forward(self, x, prev_state):
        # Unpack previous state
        memory, read_weights, write_weights, link_matrix, \
            usage, controller_state = prev_state
        
        # Previous read vectors
        read_vectors = self.read(memory, read_weights)
        
        # Controller input: x + previous reads
        controller_input = torch.cat([x, read_vectors.flatten()])
        controller_out, controller_state = self.controller(
            controller_input.unsqueeze(0), controller_state
        )
        
        # Parse interface vector
        interface = self.interface(controller_out.squeeze(0))
        read_keys, read_strengths, write_key, write_strength, \
            erase, add, free_gates, alloc_gate, write_gate, \
            read_modes = self.parse_interface(interface)
        
        # Memory operations
        usage = self.update_usage(usage, read_weights, write_weights, free_gates)
        write_weights = self.compute_write_weights(...)
        memory = self.write(memory, write_weights, erase, add)
        link_matrix = self.update_links(link_matrix, write_weights)
        read_weights = self.compute_read_weights(...)
        read_vectors = self.read(memory, read_weights)
        
        # Output
        output = self.output(torch.cat([controller_out.squeeze(0), 
                                         read_vectors.flatten()]))
        
        # Pack new state
        new_state = (memory, read_weights, write_weights, link_matrix,
                    usage, controller_state)
        
        return output, new_state
```

---

## Capabilities

| Task | Performance |
|------|-------------|
| **Copy** | Perfect |
| **Repeat Copy** | Near-perfect |
| **Associative Recall** | Excellent |
| **Graph Traversal** | Works |
| **Shortest Path** | Works with training |
| **Sorting** | Learns basic sorting |

---

## DNC vs Alternatives

| Model | Memory | Addressing |
|-------|--------|------------|
| **LSTM** | Hidden state | Implicit |
| **NTM** | External matrix | Content + location |
| **DNC** | External + links | Content + temporal |
| **Transformers** | Attention | All-to-all |

---

## Limitations

```
1. Training difficulty
   - Many moving parts
   - Gradients through memory complex

2. Scalability
   - O(N²) for link matrix
   - N = memory size

3. Interpretability
   - Hard to understand what's stored

4. Mostly superseded by transformers
   - Attention is simpler and scales better
```

---

## Interview Tips

When discussing DNCs:
1. Explain the memory + controller architecture
2. Describe read/write attention mechanisms
3. Know temporal linking purpose
4. Compare with transformers
5. Mention it's more historical now

