# LoRA (Low-Rank Adaptation)

## What It Is

**LoRA** is a parameter-efficient fine-tuning technique for large language models. Instead of updating all model weights, it injects trainable low-rank matrices into each transformer layer, reducing trainable parameters by 10,000x.

---

## The Analogy 🎨

Think of **customizing a car**:
- **Full fine-tuning**: Rebuild the entire engine (expensive, time-consuming)
- **LoRA**: Add a turbocharger (small addition, big impact)

LoRA adds small "adapters" to the existing model rather than modifying everything.

---

## Why It Exists

### The Problem: Fine-Tuning at Scale
```
GPT-3 175B parameters:
- Full fine-tuning: Store 175B × 4 bytes = 700GB per task
- 10 tasks = 7TB storage!
- Training requires massive GPU memory
- Each task needs complete model copy

Need: Efficient adaptation for specific tasks
```

### What LoRA Solves:
- **Parameter efficiency** - 0.01% trainable parameters
- **Memory efficiency** - Much smaller GPU requirement
- **Storage efficiency** - ~10MB per adapter vs 700GB
- **No inference latency** - Merge weights at deployment

---

## How It Works

### Core Idea: Low-Rank Decomposition
```
┌─────────────────────────────────────────────────────────────────┐
│                    LoRA Architecture                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Original weight matrix W: d × k (e.g., 4096 × 4096)           │
│                                                                  │
│   Instead of updating W directly:                                │
│   W' = W + ΔW                                                   │
│                                                                  │
│   LoRA decomposes ΔW as:                                        │
│   ΔW = B × A                                                    │
│        (d × r) × (r × k)                                        │
│                                                                  │
│   Where r << d, k (e.g., r = 8)                                 │
│                                                                  │
│   Parameters:                                                    │
│   - Original: 4096 × 4096 = 16.7M                               │
│   - LoRA: (4096 × 8) + (8 × 4096) = 65K                         │
│   - Reduction: 256x fewer parameters!                            │
│                                                                  │
│           ┌───────────────────────┐                             │
│   Input ──▶  W (frozen)           │                             │
│      │   └───────────────────────┘                              │
│      │              │                                            │
│      │              ▼                                            │
│      │        Add outputs                                        │
│      │              ▲                                            │
│      │   ┌───┐    ┌───┐                                         │
│      └──▶│ A │───▶│ B │ (trainable, rank r)                     │
│          └───┘    └───┘                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Mathematical Formulation:
```
Forward pass:
h = Wx + BAx
  = Wx + ΔWx
  = (W + BA)x

During training:
- W is frozen
- Only A and B are updated

After training:
- Merge: W' = W + BA
- No additional latency!
```

---

## Key Hyperparameters

| Parameter | Description | Typical Values |
|-----------|-------------|----------------|
| **r (rank)** | Rank of decomposition | 4, 8, 16, 32 |
| **α (alpha)** | Scaling factor | 16, 32 |
| **target_modules** | Which layers to adapt | q_proj, v_proj, k_proj |
| **dropout** | LoRA dropout | 0.05, 0.1 |

### Scaling:
```python
# Actual update is scaled by α/r
ΔW = (α/r) × B × A

# Higher α = stronger adaptation
# Common: α = 2r (so scaling = 2)
```

---

## Implementation

```python
from peft import LoraConfig, get_peft_model

# Configuration
lora_config = LoraConfig(
    r=16,                      # Low-rank dimension
    lora_alpha=32,             # Scaling factor
    target_modules=["q_proj", "v_proj"],  # Which weights
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)

# Apply LoRA to model
model = get_peft_model(base_model, lora_config)

# Check trainable parameters
model.print_trainable_parameters()
# "trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.062"
```

---

## When to Use

### ✅ Good Fit:
- **Task-specific adaptation** - Customer service, coding
- **Limited compute** - Consumer GPUs (24GB)
- **Multiple adapters** - Different tasks, same base
- **Quick experiments** - Faster training iterations

### ❌ Avoid When:
- Task requires massive behavior change
- Have unlimited compute resources
- Need maximum quality regardless of cost

---

## LoRA Variants

| Variant | Description |
|---------|-------------|
| **QLoRA** | LoRA + 4-bit quantization |
| **AdaLoRA** | Adaptive rank allocation |
| **LoRA-FA** | Frozen-A variant |
| **DoRA** | Decomposed LoRA |

### QLoRA (Quantized LoRA):
```
Base model: 4-bit quantized (huge memory savings)
LoRA adapters: 16-bit (trainable)

Train 65B model on single 48GB GPU!
```

---

## Comparison with Alternatives

| Method | Trainable Params | Quality | Memory |
|--------|------------------|---------|--------|
| **Full Fine-tuning** | 100% | Best | Very High |
| **LoRA** | 0.1% | Very Good | Low |
| **Prompt Tuning** | 0.01% | Good | Very Low |
| **Adapters** | 1-5% | Good | Medium |

---

## Real-World Usage

| Application | Benefit |
|-------------|---------|
| **ChatGPT fine-tuning** | Custom personalities |
| **Stable Diffusion** | Style adapters (artist styles) |
| **Code assistants** | Language-specific tuning |
| **Domain adaptation** | Legal, medical, finance |

---

## Interview Tips

When discussing LoRA:
1. Explain the low-rank decomposition concept
2. Highlight parameter efficiency (10,000x reduction)
3. Mention no inference latency (weights merge)
4. Know QLoRA for even more memory savings
5. Discuss rank selection trade-offs

