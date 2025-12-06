# Mixed Precision Training

## What It Is

**Mixed Precision Training** uses lower-precision data types (FP16/BF16) for most operations while keeping critical calculations in FP32, reducing memory usage and increasing training speed without sacrificing model quality.

---

## The Analogy 🎨

Think of **painting a picture**:
- **FP32**: Use finest brush for every stroke (slow, precise)
- **Mixed Precision**: Fine brush for details, broad brush for backgrounds
- Result looks the same, but much faster

---

## Why It Exists

### Precision Types
```
FP32 (Single Precision):
┌─────┬──────────┬─────────────────────────┐
│Sign │ Exponent │        Mantissa         │
│ 1   │    8     │           23            │ = 32 bits
└─────┴──────────┴─────────────────────────┘
Range: ±3.4×10³⁸, 7 decimal digits

FP16 (Half Precision):
┌─────┬──────────┬───────────┐
│Sign │ Exponent │ Mantissa  │
│ 1   │    5     │    10     │ = 16 bits
└─────┴──────────┴───────────┘
Range: ±65504, 3 decimal digits

BF16 (Brain Float):
┌─────┬──────────┬─────────┐
│Sign │ Exponent │Mantissa │
│ 1   │    8     │    7    │ = 16 bits
└─────┴──────────┴─────────┘
Range: ±3.4×10³⁸, 2 decimal digits (same range as FP32!)
```

### Benefits
```
Memory: FP16 uses 2 bytes vs 4 bytes (50% less)
Speed: Tensor cores process FP16 2-8x faster
Bandwidth: Half the data to transfer
```

---

## How It Works

### The Challenge: Numerical Stability
```
Problem with pure FP16:
- Gradients can underflow (too small)
- Weights can overflow (too large)
- Loss of precision accumulates

Solution: Strategic mixing of precisions
```

### Mixed Precision Strategy
```
┌─────────────────────────────────────────────────────────────────┐
│                   Mixed Precision Training                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   FP32 Master Weights ──copy──▶ FP16 Weights                    │
│                                      │                           │
│                            Forward Pass (FP16)                   │
│                                      │                           │
│                            Compute Loss (FP32)                   │
│                                      │                           │
│                            Backward Pass (FP16)                  │
│                                      │                           │
│   FP32 Master Weights ◀──update── FP16 Gradients               │
│                                                                  │
│   Critical operations in FP32:                                  │
│   - Weight updates                                              │
│   - Loss computation                                            │
│   - Softmax                                                     │
│   - Layer normalization                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Loss Scaling

### Why It's Needed
```
FP16 range: [6×10⁻⁸, 65504]
Many gradients: 10⁻⁶ to 10⁻⁸ (underflow in FP16!)

Solution: Scale up loss, gradients stay in range
```

### Implementation
```python
# Manual loss scaling
loss = model(x, y)
scaled_loss = loss * 1024  # Scale up
scaled_loss.backward()

# Unscale gradients before weight update
for param in model.parameters():
    param.grad /= 1024
optimizer.step()
```

### Dynamic Loss Scaling
```python
scaler = torch.cuda.amp.GradScaler()

for data, target in dataloader:
    optimizer.zero_grad()
    
    with torch.cuda.amp.autocast():
        output = model(data)
        loss = criterion(output, target)
    
    # Scales loss, calls backward
    scaler.scale(loss).backward()
    
    # Unscales gradients, clips, updates
    scaler.step(optimizer)
    
    # Updates scale for next iteration
    scaler.update()
```

---

## PyTorch Implementation

### Automatic Mixed Precision (AMP)
```python
import torch
from torch.cuda.amp import autocast, GradScaler

model = MyModel().cuda()
optimizer = torch.optim.Adam(model.parameters())
scaler = GradScaler()

for epoch in range(num_epochs):
    for inputs, targets in dataloader:
        inputs, targets = inputs.cuda(), targets.cuda()
        
        optimizer.zero_grad()
        
        # Runs forward pass in FP16
        with autocast():
            outputs = model(inputs)
            loss = loss_fn(outputs, targets)
        
        # Backward pass with scaling
        scaler.scale(loss).backward()
        scaler.step(optimizer)
        scaler.update()
```

### BF16 Training
```python
# BF16 doesn't need loss scaling (same range as FP32)
with autocast(dtype=torch.bfloat16):
    outputs = model(inputs)
    loss = loss_fn(outputs, targets)

loss.backward()
optimizer.step()
```

---

## FP16 vs BF16

| Aspect | FP16 | BF16 |
|--------|------|------|
| **Range** | ±65504 | ±3.4×10³⁸ |
| **Precision** | 3 digits | 2 digits |
| **Loss scaling** | Required | Not needed |
| **Hardware** | V100, A100, T4 | A100, H100, TPU |
| **Stability** | Needs care | More stable |

---

## Hugging Face Integration

```python
from transformers import TrainingArguments, Trainer

training_args = TrainingArguments(
    output_dir="./output",
    fp16=True,                    # Use FP16
    # OR
    bf16=True,                    # Use BF16 (better if hardware supports)
    fp16_full_eval=True,          # FP16 for eval too
    gradient_checkpointing=True,  # Combine with checkpointing
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
)
trainer.train()
```

---

## Performance Gains

```
Typical improvements (A100 GPU):

╔════════════════════════════════════════════════╗
║  Metric          │  FP32    │  Mixed Precision ║
╠════════════════════════════════════════════════╣
║  Memory usage    │  16GB    │  10GB (-37%)     ║
║  Training speed  │  1x      │  1.5-3x          ║
║  Model quality   │  baseline│  ~same           ║
╚════════════════════════════════════════════════╝
```

---

## Common Issues

```
1. NaN/Inf values
   - Reduce learning rate
   - Increase initial loss scale
   - Check for numerical instability

2. Accuracy degradation
   - Keep more operations in FP32
   - Use BF16 instead of FP16

3. Hardware doesn't support
   - Check for Tensor Cores (V100+)
   - Fallback to FP32
```

---

## Best Practices

```
1. Use BF16 if hardware supports (A100+)
2. Always use automatic loss scaling with FP16
3. Combine with gradient checkpointing
4. Monitor for NaN/Inf during training
5. Keep batch norms and softmax in FP32
6. Validate quality on held-out set
```

---

## Interview Tips

When discussing Mixed Precision:
1. Explain FP32 vs FP16 vs BF16 trade-offs
2. Describe loss scaling and why it's needed
3. Know which operations stay in FP32
4. Mention hardware requirements (Tensor Cores)
5. Give performance improvement numbers

