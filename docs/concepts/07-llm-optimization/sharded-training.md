# Sharded Training (FSDP/ZeRO)

## What It Is

**Sharded Training** distributes model states (parameters, gradients, optimizer states) across multiple GPUs, enabling training of models much larger than single-GPU memory allows.

---

## The Analogy 📚

Think of **team studying a huge textbook**:
- **Data Parallel**: Everyone has full copy, studies different chapters
- **Sharded**: Each person holds few pages, shares when needed
- Sharded uses less paper (memory) but needs more passing around (communication)

---

## Why It Exists

### Memory Breakdown for Training
```
Training a 7B parameter model:

╔════════════════════════════════════════════════╗
║  Component            │  Memory (FP32)         ║
╠════════════════════════════════════════════════╣
║  Parameters           │  28 GB (7B × 4 bytes)  ║
║  Gradients            │  28 GB (7B × 4 bytes)  ║
║  Optimizer (Adam)     │  56 GB (7B × 8 bytes)  ║
║  Activations          │  ~20-50 GB             ║
╠════════════════════════════════════════════════╣
║  Total                │  ~130+ GB              ║
╚════════════════════════════════════════════════╝

A100 GPU: 80GB → Can't fit!

Solution: Shard across multiple GPUs
```

---

## ZeRO Stages

### ZeRO-1: Shard Optimizer States
```
┌─────────────────────────────────────────────────────────────────┐
│                        ZeRO-1                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   GPU 0: [Full Params] [Full Grads] [Optimizer 1/4]            │
│   GPU 1: [Full Params] [Full Grads] [Optimizer 2/4]            │
│   GPU 2: [Full Params] [Full Grads] [Optimizer 3/4]            │
│   GPU 3: [Full Params] [Full Grads] [Optimizer 4/4]            │
│                                                                  │
│   Memory per GPU: P + G + O/N                                   │
│   Savings: Optimizer memory ÷ N                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ZeRO-2: + Shard Gradients
```
┌─────────────────────────────────────────────────────────────────┐
│                        ZeRO-2                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   GPU 0: [Full Params] [Gradients 1/4] [Optimizer 1/4]         │
│   GPU 1: [Full Params] [Gradients 2/4] [Optimizer 2/4]         │
│   GPU 2: [Full Params] [Gradients 3/4] [Optimizer 3/4]         │
│   GPU 3: [Full Params] [Gradients 4/4] [Optimizer 4/4]         │
│                                                                  │
│   Memory per GPU: P + (G + O)/N                                 │
│   Additional: Reduce-scatter for gradients                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ZeRO-3: + Shard Parameters
```
┌─────────────────────────────────────────────────────────────────┐
│                        ZeRO-3 (Full Sharding)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   GPU 0: [Params 1/4] [Gradients 1/4] [Optimizer 1/4]          │
│   GPU 1: [Params 2/4] [Gradients 2/4] [Optimizer 2/4]          │
│   GPU 2: [Params 3/4] [Gradients 3/4] [Optimizer 3/4]          │
│   GPU 3: [Params 4/4] [Gradients 4/4] [Optimizer 4/4]          │
│                                                                  │
│   Memory per GPU: (P + G + O)/N                                 │
│   Gather params during forward/backward (all-gather)           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Memory Comparison

```
7B model, 4 GPUs, FP32:

╔════════════════════════════════════════════════╗
║  Strategy      │  Per-GPU Memory              ║
╠════════════════════════════════════════════════╣
║  DDP           │  112 GB (doesn't fit!)       ║
║  ZeRO-1        │  70 GB                       ║
║  ZeRO-2        │  56 GB                       ║
║  ZeRO-3        │  28 GB ✓                     ║
╚════════════════════════════════════════════════╝
```

---

## PyTorch FSDP

```python
import torch
from torch.distributed.fsdp import (
    FullyShardedDataParallel as FSDP,
    ShardingStrategy,
)
from torch.distributed.fsdp.wrap import transformer_auto_wrap_policy

# Define wrapping policy
wrap_policy = transformer_auto_wrap_policy(
    transformer_layer_cls={TransformerBlock}
)

# Wrap model with FSDP
model = FSDP(
    model,
    sharding_strategy=ShardingStrategy.FULL_SHARD,  # ZeRO-3
    auto_wrap_policy=wrap_policy,
    mixed_precision=MixedPrecision(
        param_dtype=torch.float16,
        reduce_dtype=torch.float16,
        buffer_dtype=torch.float16,
    ),
)

# Training loop (same as DDP)
for batch in dataloader:
    loss = model(batch)
    loss.backward()
    optimizer.step()
```

---

## DeepSpeed ZeRO

```python
import deepspeed

# config.json
config = {
    "zero_optimization": {
        "stage": 3,  # ZeRO-3
        "offload_param": {"device": "cpu"},
        "offload_optimizer": {"device": "cpu"},
    },
    "fp16": {"enabled": True},
    "train_batch_size": 32,
}

# Initialize
model, optimizer, _, _ = deepspeed.initialize(
    model=model,
    model_parameters=model.parameters(),
    config=config,
)

# Training
for batch in dataloader:
    loss = model(batch)
    model.backward(loss)
    model.step()
```

---

## CPU Offloading

```
ZeRO-Offload: Move optimizer states to CPU

GPU Memory: Only current layer's params
CPU Memory: Optimizer states, unused params
PCIe Transfer: Move as needed

Enables 10x larger models on same GPU!
Trade-off: Slower due to CPU-GPU transfer
```

---

## Communication Patterns

```
ZeRO-3 Communication:

Forward Pass:
1. All-gather params for layer N
2. Compute layer N
3. Discard params (keep only shard)
4. Repeat for layer N+1

Backward Pass:
1. All-gather params for layer N
2. Compute gradients
3. Reduce-scatter gradients
4. Discard non-shard gradients
5. Repeat for layer N-1
```

---

## FSDP vs DDP vs DeepSpeed

| Aspect | DDP | FSDP | DeepSpeed |
|--------|-----|------|-----------|
| **Sharding** | None | Full | Configurable |
| **Memory** | High | Low | Lowest |
| **Compute overhead** | None | Some | Some |
| **CPU offload** | No | Yes | Yes |
| **NVLink optimized** | Yes | Yes | Yes |
| **Ease of use** | Easy | Medium | Medium |

---

## Best Practices

```
1. Start with FSDP/ZeRO-2 for most cases
2. Use ZeRO-3 only when memory-constrained
3. Enable CPU offload for very large models
4. Combine with gradient checkpointing
5. Use activation checkpointing for activations
6. Profile communication vs compute time
7. Use appropriate batch sizes per GPU
```

---

## When to Use

### ✅ Good Fit:
- Models larger than single GPU memory
- Multi-GPU training clusters
- Production LLM training

### ❌ Avoid When:
- Model fits in single GPU (use DDP)
- Small-scale experiments
- Inference (not needed)

---

## Interview Tips

When discussing Sharded Training:
1. Explain the memory breakdown (params, grads, optimizer)
2. Describe ZeRO stages 1, 2, 3
3. Know FSDP vs DeepSpeed
4. Discuss communication overhead
5. Mention CPU offloading for very large models

