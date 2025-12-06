# Sparse Mixture of Experts (MoE)

## What It Is

**Sparse Mixture of Experts (MoE)** is an architecture where only a subset of model parameters (experts) are activated for each input, enabling massive model capacity with efficient computation.

---

## The Analogy 🏥

Think of a **hospital with specialists**:
- **Dense model**: Every patient sees ALL doctors
- **MoE**: Router directs patient to 2 relevant specialists
- Same hospital capacity, but each patient processed faster

---

## Why It Exists

### The Problem: Scaling Laws
```
Dense models:
- More parameters = better quality
- More parameters = more compute
- 10x params = 10x compute cost

MoE insight:
- 10x params ≠ 10x compute
- Only activate subset per token
- Get capacity benefits without full cost
```

### What MoE Solves:
- **Efficient scaling** - 8x parameters, 2x compute
- **Specialization** - Experts learn different patterns
- **Better quality** - More capacity per FLOP

---

## How It Works

### Architecture:
```
┌─────────────────────────────────────────────────────────────────┐
│                    MoE Layer Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Input token                                                    │
│       │                                                          │
│       ▼                                                          │
│   ┌─────────┐                                                   │
│   │ Router  │ ──▶ Computes scores for each expert               │
│   │ (Gating)│     Selects top-K experts (usually K=2)           │
│   └─────────┘                                                   │
│       │                                                          │
│       ▼                                                          │
│   ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐            │
│   │ E1  │ E2  │ E3  │ E4  │ E5  │ E6  │ E7  │ E8  │            │
│   │     │ ✓   │     │     │ ✓   │     │     │     │            │
│   └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘            │
│       │                                                          │
│       ▼                                                          │
│   Weighted sum of selected expert outputs                       │
│                                                                  │
│   Output = w2 × E2(input) + w5 × E5(input)                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Router (Gating Network):
```python
# Simplified router
def router(x, num_experts=8, top_k=2):
    # Linear layer to score experts
    scores = linear(x)  # [batch, num_experts]
    
    # Select top-k experts
    weights, indices = topk(softmax(scores), k=top_k)
    
    return weights, indices  # Which experts, how much weight
```

---

## Key Concepts

### 1. Expert Capacity
```
Problem: Some experts might get all tokens (imbalanced)

Solution: Capacity factor limits tokens per expert
- Capacity = (tokens / num_experts) × capacity_factor
- Overflow tokens dropped or sent to other experts
```

### 2. Load Balancing Loss
```
Encourage even distribution across experts:

L_balance = α × Σ(fraction_i × routing_prob_i)

Where:
- fraction_i = tokens routed to expert i
- routing_prob_i = average routing probability to expert i
- α = balancing coefficient (0.01 typical)
```

### 3. Expert Parallelism
```
Distribute experts across GPUs:

GPU 0: Experts 0, 1
GPU 1: Experts 2, 3
GPU 2: Experts 4, 5
GPU 3: Experts 6, 7

All-to-all communication to route tokens
```

---

## MoE vs Dense Models

| Aspect | Dense 7B | MoE 8x7B (Mixtral) |
|--------|----------|---------------------|
| **Total params** | 7B | 47B |
| **Active params** | 7B | 13B |
| **Quality** | Baseline | Better |
| **Inference FLOPS** | 1x | ~2x |
| **Memory** | 14GB | 94GB |

### The Trade-off:
```
MoE gives you:
✅ Better quality per FLOP
✅ More capacity
❌ Higher memory (all experts loaded)
❌ Communication overhead
❌ Load balancing complexity
```

---

## Real-World MoE Models

| Model | Experts | Active | Total Params |
|-------|---------|--------|--------------|
| **Mixtral 8x7B** | 8 | 2 | 47B |
| **Mixtral 8x22B** | 8 | 2 | 141B |
| **GPT-4** (rumored) | 16 | 2 | ~1.8T |
| **Switch Transformer** | 2048 | 1 | 1.6T |
| **Grok-1** | 8 | 2 | 314B |

---

## When to Use

### ✅ Good Fit:
- **Large-scale training** - When you can afford memory
- **Diverse tasks** - Experts specialize
- **Quality-focused** - Best quality per FLOP
- **Inference at scale** - Amortize memory cost

### ❌ Avoid When:
- Memory-constrained deployment
- Simple, single-domain tasks
- Low-latency requirements (routing overhead)

---

## Implementation Challenges

### 1. All-to-All Communication
```
Tokens must be routed to correct expert GPU
Expensive communication pattern
Solution: Expert parallelism + tensor parallelism
```

### 2. Load Imbalance
```
Some experts overloaded, others idle
Solution: Auxiliary loss + capacity limits
```

### 3. Training Instability
```
Router can collapse to few experts
Solution: Noise injection, load balancing
```

---

## Inference Optimization

### Expert Offloading:
```
Not all experts fit in GPU memory?
- Keep hot experts in GPU
- Offload cold experts to CPU
- Prefetch based on routing predictions
```

### Expert Pruning:
```
Some experts rarely used?
- Identify low-usage experts
- Merge or remove them
- Reduce memory footprint
```

---

## Interview Tips

When discussing MoE:
1. Explain the capacity vs compute trade-off
2. Describe router and top-k selection
3. Discuss load balancing challenges
4. Compare with dense models (Mixtral vs LLaMA)
5. Mention memory vs compute trade-off

