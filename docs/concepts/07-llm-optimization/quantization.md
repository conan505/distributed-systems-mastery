# Quantization for LLMs

## What It Is

**Quantization** reduces the precision of model weights and activations from high-precision (32-bit or 16-bit) to lower precision (8-bit, 4-bit, or even lower), dramatically reducing memory footprint and inference costs.

---

## The Analogy 📐

Think of **map resolution**:
- **32-bit**: Street-level detail, huge file size
- **16-bit**: City-level detail, medium file
- **8-bit**: Country outlines, small file
- **4-bit**: Continental shapes, tiny file

You lose some precision, but for many tasks, the approximation is good enough.

---

## Why It Matters

### The Problem: Model Size
```
LLaMA-70B in FP32:
- 70B × 4 bytes = 280GB
- Need 4× A100 80GB GPUs just to load!

Same model in INT4:
- 70B × 0.5 bytes = 35GB
- Fits on single GPU!
```

### What Quantization Solves:
- **Memory reduction** - 4-8x smaller models
- **Faster inference** - Fewer bits = faster compute
- **Lower cost** - Use smaller/fewer GPUs
- **Edge deployment** - Run on consumer hardware

---

## Precision Levels

| Precision | Bits | Range | Memory/Param |
|-----------|------|-------|--------------|
| **FP32** | 32 | ±3.4e38 | 4 bytes |
| **FP16** | 16 | ±65,504 | 2 bytes |
| **BF16** | 16 | ±3.4e38 | 2 bytes |
| **INT8** | 8 | -128 to 127 | 1 byte |
| **INT4** | 4 | -8 to 7 | 0.5 bytes |

### BFloat16 vs Float16:
```
FP16: More precision, smaller range
      Sign: 1, Exponent: 5, Mantissa: 10

BF16: Less precision, same range as FP32
      Sign: 1, Exponent: 8, Mantissa: 7

BF16 preferred for training (handles gradients better)
```

---

## Quantization Methods

### 1. Post-Training Quantization (PTQ)
```
┌─────────────────────────────────────────────────────────────────┐
│                Post-Training Quantization                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Train model normally (FP32)                                │
│   2. Calibrate: Run sample data to find value ranges            │
│   3. Quantize: Map weights to lower precision                   │
│                                                                  │
│   ✅ No retraining needed                                        │
│   ✅ Fast and simple                                             │
│   ❌ Some accuracy loss                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Quantization-Aware Training (QAT)
```
┌─────────────────────────────────────────────────────────────────┐
│              Quantization-Aware Training                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Insert fake quantization during training                   │
│   2. Model learns to handle quantization noise                  │
│   3. Final model is quantization-friendly                       │
│                                                                  │
│   ✅ Better accuracy                                             │
│   ❌ Requires retraining                                         │
│   ❌ More complex                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Popular Quantization Techniques

### GPTQ (GPT Quantization)
```
- Post-training weight quantization
- Layer-by-layer optimization
- Uses calibration data
- Popular for 4-bit quantization

Accuracy: Very good for INT4
Speed: Fast inference
```

### AWQ (Activation-aware Weight Quantization)
```
- Identifies important weights (by activation magnitude)
- Protects important channels from quantization
- Better accuracy than naive quantization

Insight: Not all weights are equal
```

### GGML/GGUF (llama.cpp)
```
- CPU-focused quantization
- Multiple quantization levels (Q4_0, Q4_K, Q8_0)
- Run LLMs on MacBooks, no GPU!
```

### bitsandbytes
```python
from transformers import BitsAndBytesConfig

# 4-bit quantization
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_quant_type="nf4",  # Normal Float 4
    bnb_4bit_use_double_quant=True
)

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config
)
```

---

## Quantization Trade-offs

| Precision | Memory Reduction | Quality Impact |
|-----------|------------------|----------------|
| FP32 → FP16 | 2x | Minimal |
| FP16 → INT8 | 2x | Slight |
| INT8 → INT4 | 2x | Noticeable |
| FP32 → INT4 | 8x | Moderate |

### What Degrades:
```
- Rare vocabulary words
- Complex reasoning
- Mathematical accuracy
- Long-context coherence

Works well for:
- Common language patterns
- Short responses
- Classification tasks
```

---

## When to Use

### ✅ Good Fit:
- **Inference at scale** - Reduce serving costs
- **Edge deployment** - Mobile, IoT, laptops
- **Memory-constrained** - Limited GPU RAM
- **Latency-sensitive** - Faster inference

### ❌ Avoid When:
- Training (use FP16/BF16)
- Need maximum accuracy
- Scientific/mathematical applications

---

## Real-World Examples

| Model | Quantization | Benefit |
|-------|--------------|---------|
| **Llama.cpp** | 4-bit GGML | Run 70B on MacBook |
| **ExLlama** | GPTQ | Fast 4-bit inference |
| **TensorRT-LLM** | INT8/FP8 | Production serving |
| **vLLM** | AWQ/GPTQ | High-throughput serving |

---

## Interview Tips

When discussing Quantization:
1. Explain bit precision trade-offs
2. Know PTQ vs QAT differences
3. Mention popular methods (GPTQ, AWQ, GGML)
4. Discuss memory savings (4-8x)
5. Understand quality degradation scenarios

