# CPU Offloading

## What It Is

**CPU Offloading** is a technique that moves model parameters and optimizer states from GPU to CPU memory, enabling training and inference of models larger than available GPU memory.

---

## The Analogy 🏠

Think of **a small desk (GPU) with a large closet (CPU)**:
- You can only work on a few papers at a time on the desk
- Store the rest in the closet
- Fetch papers as needed, return when done
- Slower than having everything on desk, but accommodates more

---

## Why It Exists

### The Memory Gap
```
GPU Memory:
- A100: 40-80GB
- Consumer GPUs: 8-24GB

Model Requirements:
- 7B model: ~28GB (parameters only)
- 70B model: ~280GB (parameters only)
- Training: 4-8x parameter size!

Solution: Use CPU RAM (128-512GB typical)
```

---

## What Can Be Offloaded

```
┌─────────────────────────────────────────────────────────────────┐
│                 Memory Components (Training)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Component         │  Size (7B FP32)  │  Offload Priority     │
│   ─────────────────│─────────────────│──────────────────────   │
│   Optimizer States  │  56 GB           │  First (rarely used)   │
│   Parameters        │  28 GB           │  Second                │
│   Gradients         │  28 GB           │  Third                 │
│   Activations       │  Variable        │  Last (need frequently)│
│                                                                  │
│   Offload optimizer → saves most memory, least impact          │
│   Offload params → larger models, more transfer overhead       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Strategies

### 1. DeepSpeed ZeRO-Offload
```python
# deepspeed_config.json
{
    "zero_optimization": {
        "stage": 2,
        "offload_optimizer": {
            "device": "cpu",
            "pin_memory": true
        },
        "offload_param": {
            "device": "cpu",
            "pin_memory": true
        }
    },
    "train_batch_size": 32
}

# Initialize DeepSpeed
import deepspeed

model, optimizer, _, _ = deepspeed.initialize(
    model=model,
    config="deepspeed_config.json"
)
```

### 2. PyTorch FSDP with CPU Offload
```python
from torch.distributed.fsdp import (
    FullyShardedDataParallel as FSDP,
    CPUOffload,
)

model = FSDP(
    model,
    cpu_offload=CPUOffload(offload_params=True),
    sharding_strategy=ShardingStrategy.FULL_SHARD,
)
```

### 3. Accelerate Library
```python
from accelerate import Accelerator, DeepSpeedPlugin

deepspeed_plugin = DeepSpeedPlugin(
    zero_stage=2,
    offload_optimizer_device="cpu",
    offload_param_device="cpu",
)

accelerator = Accelerator(deepspeed_plugin=deepspeed_plugin)
model = accelerator.prepare(model)
```

---

## Pinned Memory

```python
# Pinned (page-locked) memory enables faster CPU-GPU transfer

# Without pinned memory:
# CPU → Pageable Memory → Pinned Buffer → GPU
# Extra copy needed!

# With pinned memory:
# CPU (Pinned) → GPU
# Direct transfer, 2-3x faster

# Implementation
tensor_cpu = torch.empty(size, pin_memory=True)
```

---

## Async Transfer (Prefetching)

```
Overlap computation with data transfer:

┌─────────────────────────────────────────────────────────────────┐
│                    Prefetch Strategy                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Time →                                                         │
│   GPU:  [Compute Layer N  ][Compute Layer N+1][Compute N+2]     │
│   PCIe: [Fetch N+1 params ][Fetch N+2 params ][Fetch N+3]       │
│                                                                  │
│   While GPU computes layer N:                                   │
│   - Prefetch layer N+1 params from CPU                          │
│   - Evict layer N-1 params back to CPU                          │
│                                                                  │
│   Hides transfer latency!                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
# Manual prefetching
class OffloadedLayer:
    def __init__(self, layer):
        self.layer = layer
        self.params_on_gpu = False
        self.stream = torch.cuda.Stream()
    
    def prefetch(self):
        """Move params to GPU asynchronously."""
        with torch.cuda.stream(self.stream):
            for p in self.layer.parameters():
                p.data = p.data.to('cuda', non_blocking=True)
        self.params_on_gpu = True
    
    def offload(self):
        """Move params back to CPU."""
        for p in self.layer.parameters():
            p.data = p.data.to('cpu', non_blocking=True)
        self.params_on_gpu = False
    
    def forward(self, x):
        self.stream.synchronize()  # Wait for prefetch
        return self.layer(x)
```

---

## Inference Offloading

### BitsAndBytes with Offloading
```python
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-70b",
    device_map="auto",      # Automatic CPU/GPU split
    offload_folder="offload",  # Disk offload if needed
    offload_state_dict=True,
    torch_dtype=torch.float16,
)

# Model automatically moves layers as needed
output = model.generate(input_ids)
```

### llama.cpp CPU Inference
```bash
# Run 70B model on CPU with mmap
./main -m llama-70b.gguf \
    --mlock \           # Keep in RAM
    --threads 16 \      # Use all cores
    -n 100              # Generate 100 tokens
```

---

## Performance Impact

```
CPU Offload Overhead (typical):

╔════════════════════════════════════════════════╗
║  Scenario             │  Slowdown              ║
╠════════════════════════════════════════════════╣
║  Optimizer offload    │  1.2-1.5x              ║
║  + Param offload      │  2-3x                  ║
║  + Activation offload │  3-5x                  ║
║  Full CPU inference   │  10-20x                ║
╚════════════════════════════════════════════════╝

Trade-off: Slower but enables larger models
```

---

## NVMe Offloading

```
For extremely large models: Offload to SSD

Hierarchy:
GPU (80GB) → CPU (256GB) → NVMe (2TB)

DeepSpeed ZeRO-Infinity:
- Offload optimizer states to NVMe
- Offload parameters to NVMe
- Enables trillion-parameter training

Speed: NVMe ~3GB/s vs PCIe ~25GB/s
Use only when necessary
```

---

## Best Practices

```
1. Start with optimizer offload only
2. Add param offload if still OOM
3. Use pinned memory for faster transfer
4. Enable async prefetching
5. Combine with gradient checkpointing
6. Use mixed precision to reduce transfer size
7. Profile to identify bottlenecks
8. Use NVMe only as last resort
```

---

## When to Use

### ✅ Good Fit:
- Model larger than GPU memory
- Limited GPU budget
- Research/experimentation
- Single GPU training

### ❌ Avoid When:
- Model fits in GPU memory
- Speed is critical
- Multi-GPU available (use sharding instead)

---

## Interview Tips

When discussing CPU Offloading:
1. Explain what gets offloaded (optimizer first)
2. Describe prefetching for latency hiding
3. Know the performance trade-offs
4. Mention pinned memory benefits
5. Compare with model sharding

