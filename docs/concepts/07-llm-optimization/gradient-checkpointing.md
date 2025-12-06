# Gradient Checkpointing

## What It Is

**Gradient Checkpointing** is a memory optimization technique that trades compute for memory by recomputing activations during backpropagation instead of storing them all, enabling training of larger models with limited GPU memory.

---

## The Analogy 📸

Think of **hiking and taking photos**:
- **Without checkpointing**: Take photos constantly (high memory)
- **With checkpointing**: Take photos at landmarks only
- When you need a specific view, walk back from nearest landmark
- Uses less storage but requires re-walking

---

## Why It Exists

### The Memory Problem
```
Forward pass through neural network:
Layer 1 → activation₁ (save for backprop)
Layer 2 → activation₂ (save for backprop)
Layer 3 → activation₃ (save for backprop)
...
Layer N → activationₙ (save for backprop)

Memory = O(N × batch_size × hidden_dim)

For GPT-3 (175B params):
- Activations: ~350GB per batch!
- GPU memory: 80GB (A100)
- Can't fit!
```

### Solution: Selective Storage
```
Save only checkpoints, recompute the rest

Forward: Save every √N layers (checkpoints)
Backward: Recompute from nearest checkpoint

Memory: O(√N) instead of O(N)
Compute: ~33% more forward passes
```

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│           Without Checkpointing (All Activations)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Forward:  L1 → L2 → L3 → L4 → L5 → L6 → Loss                 │
│   Stored:   [a1] [a2] [a3] [a4] [a5] [a6]                       │
│   Memory:   6 activations                                       │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│           With Checkpointing (√N Strategy)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Forward:  L1 → L2 → L3 → L4 → L5 → L6 → Loss                 │
│   Stored:   [a1]      [a3]      [a5]                           │
│   Memory:   3 activations (checkpoints only)                    │
│                                                                  │
│   Backward (need a4):                                           │
│   Recompute: a3 → L4 → a4                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation

### PyTorch Checkpoint
```python
import torch
from torch.utils.checkpoint import checkpoint

class TransformerBlock(nn.Module):
    def __init__(self):
        super().__init__()
        self.attention = MultiHeadAttention()
        self.ffn = FeedForward()
    
    def forward(self, x):
        x = x + self.attention(x)
        x = x + self.ffn(x)
        return x

class CheckpointedTransformer(nn.Module):
    def __init__(self, num_layers):
        super().__init__()
        self.layers = nn.ModuleList([
            TransformerBlock() for _ in range(num_layers)
        ])
    
    def forward(self, x):
        for layer in self.layers:
            # Checkpoint each layer
            x = checkpoint(layer, x, use_reentrant=False)
        return x
```

### Selective Checkpointing
```python
def forward(self, x):
    for i, layer in enumerate(self.layers):
        if i % 3 == 0:  # Checkpoint every 3rd layer
            x = checkpoint(layer, x, use_reentrant=False)
        else:
            x = layer(x)
    return x
```

---

## Checkpointing Strategies

### 1. Uniform Checkpointing
```
Checkpoint every k layers

k = √N minimizes memory × compute

Memory: O(N/k + k)
Compute: O(N × k) recomputation
```

### 2. Selective Checkpointing
```python
# Only checkpoint memory-heavy layers
def should_checkpoint(layer):
    return isinstance(layer, (Attention, LargeFFN))
```

### 3. Activation Recomputation
```
Full: Recompute all activations
Selective: Recompute only attention (most memory)
None: Store everything
```

---

## Memory Savings

```
Model: 24 layer Transformer

Without checkpointing:
- Activations: 24 × 512MB = 12.3GB

With √N checkpointing (checkpoint every 5 layers):
- Stored: 5 × 512MB = 2.5GB
- Recompute overhead: ~33%

Memory saved: ~80%!
```

---

## Trade-offs

### Memory vs Compute
```
┌────────────────────────────────────────┐
│   Strategy        Memory    Compute   │
├────────────────────────────────────────┤
│   No checkpoint   O(N)      1x        │
│   √N checkpoint   O(√N)     1.5x      │
│   Full recompute  O(1)      2x        │
└────────────────────────────────────────┘
```

---

## Hugging Face Integration

```python
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b",
    torch_dtype=torch.float16,
)

# Enable gradient checkpointing
model.gradient_checkpointing_enable()

# Training now uses less memory
trainer = Trainer(
    model=model,
    args=TrainingArguments(
        gradient_checkpointing=True,
        per_device_train_batch_size=4,  # Can use larger batch!
    ),
)
```

---

## Combining with Other Techniques

```python
# Maximum memory efficiency
model = AutoModelForCausalLM.from_pretrained(
    "model_name",
    load_in_8bit=True,           # Quantization
    device_map="auto",            # Model parallelism
)
model.gradient_checkpointing_enable()  # Checkpointing

# Now can train 70B model on single GPU!
```

---

## Best Practices

```
1. Start with gradient_checkpointing_enable()
2. Combine with mixed precision (fp16/bf16)
3. Adjust batch size after enabling
4. Profile memory to find optimal checkpoint frequency
5. Use with LoRA for even better memory efficiency
6. Monitor training speed - ensure not too slow
```

---

## When to Use

### ✅ Good Fit:
- Large models that don't fit in memory
- Fine-tuning LLMs
- Limited GPU memory
- Training with larger batch sizes

### ❌ Avoid When:
- Model already fits comfortably
- Training speed is critical
- Simple/small models

---

## Interview Tips

When discussing Gradient Checkpointing:
1. Explain the memory problem with activations
2. Describe the memory-compute trade-off
3. Know √N optimal strategy
4. Mention integration with quantization/LoRA
5. Give concrete memory savings numbers

