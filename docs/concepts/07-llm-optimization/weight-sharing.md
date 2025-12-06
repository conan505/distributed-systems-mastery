# Weight Sharing

## What It Is

**Weight Sharing** is a technique where multiple parts of a neural network share the same parameters, reducing model size, memory usage, and often improving generalization.

---

## The Analogy 🎭

Think of **actors playing multiple roles**:
- Same actor (weights) plays different characters (layers)
- Reduces cast size (parameters)
- Each performance differs based on context (input)
- Saves budget (memory) while maintaining quality

---

## Why It Exists

### The Size Problem
```
Large models have many repeated structures:

GPT-3:
- 96 transformer layers
- ~1.8B params per layer
- Total: 175B parameters

What if layers could share weights?
- 12 unique layers × 8 repeats = 96 effective layers
- Parameters: ~22B (87% reduction!)
```

---

## Types of Weight Sharing

### 1. Cross-Layer Sharing (ALBERT)
```
┌─────────────────────────────────────────────────────────────────┐
│                 ALBERT: Cross-Layer Sharing                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Standard BERT:                                                │
│   Layer 1: [Weights A] → Layer 2: [Weights B] → Layer 3: [C]   │
│   Parameters: 3 × W                                             │
│                                                                  │
│   ALBERT:                                                       │
│   Layer 1: [Weights A] → Layer 2: [Weights A] → Layer 3: [A]   │
│   Parameters: 1 × W                                             │
│                                                                  │
│   Same weights, different layer positions!                      │
│   ALBERT-xxlarge: 235M params vs BERT-large 340M               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Embedding Sharing
```python
class SharedEmbeddingModel(nn.Module):
    def __init__(self, vocab_size, embed_dim):
        super().__init__()
        # Single embedding matrix
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        
        # Output layer shares weights with embedding
        # (tied embeddings)
    
    def forward(self, x):
        # Input embedding
        embedded = self.embedding(x)
        
        # Process through layers...
        hidden = self.transformer(embedded)
        
        # Output uses SAME embedding weights (transposed)
        logits = F.linear(hidden, self.embedding.weight)
        return logits
```

### 3. Attention Head Sharing (MQA/GQA)
```
Multi-Query Attention (MQA):
- Multiple query heads
- SINGLE shared key-value head
- 10x memory reduction for KV cache

Grouped-Query Attention (GQA):
- Groups of query heads share K,V
- Balance between MHA and MQA
```

---

## Implementation

### Cross-Layer Parameter Sharing
```python
class SharedTransformer(nn.Module):
    def __init__(self, num_layers, hidden_dim, num_shared_layers=1):
        super().__init__()
        # Only create num_shared_layers unique layers
        self.shared_layers = nn.ModuleList([
            TransformerBlock(hidden_dim)
            for _ in range(num_shared_layers)
        ])
        self.num_layers = num_layers
        self.num_shared = num_shared_layers
    
    def forward(self, x):
        for i in range(self.num_layers):
            # Cycle through shared layers
            layer_idx = i % self.num_shared
            x = self.shared_layers[layer_idx](x)
        return x

# 12 effective layers, 1 unique layer
model = SharedTransformer(num_layers=12, hidden_dim=768, num_shared_layers=1)
```

### Tied Embeddings
```python
from transformers import GPT2LMHeadModel

model = GPT2LMHeadModel.from_pretrained("gpt2")

# In GPT2, output projection shares weights with embeddings
# Automatically tied:
assert model.lm_head.weight is model.transformer.wte.weight
```

---

## Weight Sharing Strategies

| Strategy | Memory Savings | Quality Impact |
|----------|---------------|----------------|
| **All layers shared** | Maximum (~10x) | Can degrade |
| **Groups of layers** | Medium (~3-5x) | Minimal |
| **Tied embeddings** | Moderate (~1.5x) | Often improves |
| **Attention sharing** | Large for inference | Minimal (GQA) |

---

## ALBERT Architecture

```python
class ALBERTConfig:
    # Factorized embedding
    embedding_size = 128  # Small embedding
    hidden_size = 4096    # Large hidden
    
    # Cross-layer sharing
    num_hidden_layers = 12      # Effective depth
    num_hidden_groups = 1       # Unique layers
    inner_group_num = 1         # Layers per group

# Result:
# Parameters: 235M (vs 340M BERT-large)
# Performance: Comparable to BERT
```

---

## Benefits and Trade-offs

### Benefits
```
1. Reduced parameters
   - Smaller model files
   - Faster downloads
   - Less storage

2. Better generalization
   - Acts as regularization
   - Fewer parameters to overfit

3. Memory efficiency
   - Lower GPU memory for training
   - Smaller activation memory
```

### Trade-offs
```
1. May limit expressiveness
   - Shared layers can't specialize

2. Training challenges
   - Gradient updates affect all usages
   - May need careful learning rate

3. Not always faster
   - Same compute, just fewer params
```

---

## Real-World Examples

| Model | Sharing Type | Impact |
|-------|--------------|--------|
| **ALBERT** | Cross-layer + embeddings | 18x fewer params |
| **GPT-2/3** | Tied embeddings | ~30% fewer params |
| **Llama 2** | GQA (KV sharing) | 8x KV cache reduction |
| **Universal Transformers** | Full layer sharing | Variable depth |

---

## When to Use

### ✅ Good Fit:
- Memory-constrained deployment
- Mobile/edge inference
- Transfer learning (smaller base)
- Regularization needed

### ❌ Avoid When:
- Maximum accuracy needed
- Sufficient memory available
- Model already small

---

## Interview Tips

When discussing Weight Sharing:
1. Explain different sharing types (layer, embedding, attention)
2. Know ALBERT's approach
3. Discuss trade-offs (size vs expressiveness)
4. Mention GQA/MQA for attention
5. Give parameter reduction examples

