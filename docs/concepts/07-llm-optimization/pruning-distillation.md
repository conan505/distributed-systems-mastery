# Pruning and Knowledge Distillation

## What They Are

**Pruning** removes unnecessary weights or structures from a model to reduce size and computation.

**Knowledge Distillation** trains a smaller "student" model to mimic a larger "teacher" model's behavior.

---

## The Analogy 🌳

**Pruning**: Like trimming a tree
- Remove dead branches (zero/small weights)
- Tree stays healthy, becomes more manageable
- Same fruit, less maintenance

**Distillation**: Like teaching an apprentice
- Master (teacher) demonstrates skills
- Apprentice (student) learns the essence
- Apprentice becomes competent, works faster

---

## Pruning

### Why Prune?
```
Observation: Many neural network weights are near zero
- 90% of weights contribute little
- Removing them barely affects accuracy
- Huge efficiency gains possible
```

### Types of Pruning:

#### 1. Unstructured Pruning
```
┌─────────────────────────────────────────────────────────────────┐
│                   Unstructured Pruning                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Before:  [0.5, 0.01, 0.8, 0.02, 0.9, 0.03]                   │
│   After:   [0.5, 0,    0.8, 0,    0.9, 0   ]                   │
│                                                                  │
│   Remove individual weights below threshold                     │
│                                                                  │
│   ✅ High sparsity (90%+)                                        │
│   ❌ Irregular memory access                                     │
│   ❌ Needs sparse hardware/libraries                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. Structured Pruning
```
┌─────────────────────────────────────────────────────────────────┐
│                    Structured Pruning                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Remove entire structures:                                      │
│   - Attention heads                                              │
│   - Neurons/channels                                             │
│   - Layers                                                       │
│                                                                  │
│   Before: 32 attention heads                                    │
│   After:  24 attention heads                                    │
│                                                                  │
│   ✅ Works on standard hardware                                  │
│   ✅ Real speedups                                               │
│   ❌ Lower sparsity achievable                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Pruning Methods:

| Method | Description |
|--------|-------------|
| **Magnitude** | Remove smallest weights |
| **Movement** | Remove weights that move toward zero during training |
| **Lottery Ticket** | Find sparse subnetwork that trains well |
| **Gradual** | Prune incrementally during training |

### Pruning Pipeline:
```
1. Train full model
2. Identify unimportant weights
3. Remove weights (set to zero or delete)
4. Fine-tune to recover accuracy
5. Repeat if needed
```

---

## Knowledge Distillation

### Why Distill?
```
Large model: 175B params, slow, expensive
Small model: 7B params, fast, cheap

Can small model learn from large model?
Yes! Distillation transfers knowledge.
```

### How It Works:
```
┌─────────────────────────────────────────────────────────────────┐
│                  Knowledge Distillation                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Input ──────┬──────▶ Teacher (frozen)                         │
│               │              │                                   │
│               │              ▼                                   │
│               │        Soft labels (probabilities)              │
│               │              │                                   │
│               │              ▼                                   │
│               └──────▶ Student ◀── Distillation Loss            │
│                              │                                   │
│                              ▼                                   │
│                        Hard labels (optional)                   │
│                                                                  │
│   Loss = α × KL(teacher_probs, student_probs)                   │
│        + (1-α) × CrossEntropy(labels, student_probs)            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Concepts:

#### Temperature Scaling:
```python
# Soften probability distributions
soft_teacher = softmax(teacher_logits / temperature)
soft_student = softmax(student_logits / temperature)

# Higher temperature = softer distribution
# Reveals more information about relative probabilities
```

#### Why Soft Labels Help:
```
Hard label: [0, 0, 1, 0, 0]  (just "cat")
Soft label: [0.01, 0.02, 0.9, 0.05, 0.02]

Soft labels reveal:
- "This is mostly cat"
- "But slightly similar to dog"
- "Definitely not airplane"

More information for student to learn from!
```

---

## Distillation Variants

| Variant | Description |
|---------|-------------|
| **Response-based** | Match output probabilities |
| **Feature-based** | Match intermediate representations |
| **Relation-based** | Match relationships between samples |
| **Self-distillation** | Model teaches itself (different layers) |

---

## Real-World Examples

### Pruning:
| Model | Technique | Result |
|-------|-----------|--------|
| **SparseGPT** | Unstructured | 50% sparse, minimal loss |
| **LLM-Pruner** | Structured | 20% smaller, 5% quality drop |
| **Wanda** | Pruning + Quantization | 50% sparse + 4-bit |

### Distillation:
| Student | Teacher | Result |
|---------|---------|--------|
| **DistilBERT** | BERT | 60% smaller, 97% quality |
| **TinyLlama** | LLaMA | 1.1B from 7B |
| **Phi-2** | GPT-4 (synthetic data) | 2.7B, strong performance |

---

## When to Use

### Pruning ✅:
- Deployment on edge devices
- Reducing inference costs
- When you have the original model

### Distillation ✅:
- Creating smaller production models
- When teacher is too expensive to serve
- Transferring capabilities to efficient architecture

### Combine Both:
```
1. Distill large model to smaller architecture
2. Prune the distilled model
3. Quantize the pruned model
4. Result: Tiny, fast, capable model
```

---

## Interview Tips

When discussing Pruning/Distillation:
1. Explain structured vs unstructured pruning
2. Describe soft labels and temperature
3. Know real examples (DistilBERT, SparseGPT)
4. Discuss combining techniques
5. Mention accuracy vs efficiency trade-offs

