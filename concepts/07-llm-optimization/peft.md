# Parameter-Efficient Fine-Tuning (PEFT)

## What It Is

**Parameter-Efficient Fine-Tuning (PEFT)** is a family of techniques that adapt large pre-trained models to new tasks by training only a small subset of parameters, dramatically reducing compute, memory, and storage requirements.

---

## The Analogy 🎨

Think of **customizing a car**:
- Full fine-tuning: Rebuild the entire engine
- PEFT: Add a turbocharger (small addition, big impact)
- Same base car, different performance characteristics
- Easy to swap modifications for different needs

---

## Why It Exists

### Full Fine-Tuning Problems
```
GPT-3 175B parameters:
- Full fine-tuning: 175B × 4 bytes = 700GB
- Optimizer states: 2-3x more = 2.1TB
- Per-task storage: 700GB each

PEFT solution:
- Train 0.1-1% of parameters
- Storage: 0.7-7GB per task
- Same base model, multiple adapters
```

---

## PEFT Methods

### 1. LoRA (Low-Rank Adaptation)
```
┌─────────────────────────────────────────────────────────────────┐
│                         LoRA                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Original: W (d × d matrix)                                    │
│   LoRA: W + BA where B (d × r), A (r × d), r << d              │
│                                                                  │
│   Example: d=4096, r=8                                          │
│   Original params: 16M                                          │
│   LoRA params: 65K (0.4%)                                       │
│                                                                  │
│   Freeze W, train only A and B                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Prefix Tuning
```python
class PrefixTuning(nn.Module):
    def __init__(self, num_layers, hidden_dim, prefix_length):
        super().__init__()
        # Learnable prefix tokens
        self.prefix = nn.Parameter(
            torch.randn(num_layers, 2, prefix_length, hidden_dim)
        )  # 2 for key and value
    
    def forward(self, layer_idx, key, value):
        # Prepend learned prefix to key and value
        prefix_k = self.prefix[layer_idx, 0]
        prefix_v = self.prefix[layer_idx, 1]
        
        key = torch.cat([prefix_k, key], dim=1)
        value = torch.cat([prefix_v, value], dim=1)
        return key, value
```

### 3. Prompt Tuning
```python
class PromptTuning(nn.Module):
    def __init__(self, num_tokens, embed_dim):
        super().__init__()
        # Learnable soft prompts
        self.soft_prompts = nn.Parameter(
            torch.randn(num_tokens, embed_dim)
        )
    
    def forward(self, input_embeds):
        # Prepend soft prompts to input
        batch_size = input_embeds.shape[0]
        prompts = self.soft_prompts.unsqueeze(0).expand(batch_size, -1, -1)
        return torch.cat([prompts, input_embeds], dim=1)
```

### 4. Adapters
```
Insert small bottleneck layers:

Original:  Input → Attention → FFN → Output
Adapter:   Input → Attention → [Adapter] → FFN → [Adapter] → Output

Adapter structure:
Input → Down-project → Activation → Up-project → + Input
(d → r → d, where r << d)
```

### 5. IA³ (Infused Adapter by Inhibiting and Amplifying)
```python
class IA3(nn.Module):
    def __init__(self, hidden_dim):
        super().__init__()
        # Learned scaling vectors
        self.l_k = nn.Parameter(torch.ones(hidden_dim))  # Key scaling
        self.l_v = nn.Parameter(torch.ones(hidden_dim))  # Value scaling
        self.l_ff = nn.Parameter(torch.ones(hidden_dim)) # FFN scaling
    
    def scale_attention(self, key, value):
        return key * self.l_k, value * self.l_v
    
    def scale_ffn(self, hidden):
        return hidden * self.l_ff
```

---

## Comparison

| Method | Params | Memory | Quality | Inference |
|--------|--------|--------|---------|-----------|
| **Full FT** | 100% | Very High | Best | Same |
| **LoRA** | 0.1-1% | Low | Near-best | Mergeable |
| **Prefix** | 0.1% | Low | Good | Overhead |
| **Prompt** | <0.1% | Very Low | OK | Overhead |
| **Adapter** | 1-5% | Medium | Good | Overhead |
| **IA³** | <0.01% | Very Low | Good | Minimal |

---

## Using PEFT with Hugging Face

```python
from peft import LoraConfig, get_peft_model, TaskType
from transformers import AutoModelForCausalLM

# Load base model
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b-hf")

# Configure LoRA
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=8,                      # Rank
    lora_alpha=32,            # Scaling factor
    lora_dropout=0.1,
    target_modules=["q_proj", "v_proj"],  # Which layers
)

# Create PEFT model
peft_model = get_peft_model(model, lora_config)

# Check trainable parameters
peft_model.print_trainable_parameters()
# Output: trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.06%
```

---

## Multi-Task with PEFT

```python
# Train different adapters for different tasks
adapters = {
    "sentiment": LoraConfig(...),
    "summarization": LoraConfig(...),
    "translation": LoraConfig(...),
}

# Load specific adapter at inference
model.load_adapter("sentiment")
output = model.generate(sentiment_input)

model.load_adapter("translation")
output = model.generate(translation_input)
```

---

## Best Practices

```
1. Choose right method for task
   - LoRA: General purpose, best quality
   - Prompt tuning: Very low resource
   - Adapters: When inference overhead OK

2. Target important layers
   - Attention: q, k, v projections
   - FFN: Often less important

3. Tune rank (r) carefully
   - Higher r = more capacity, more params
   - Start with r=8, increase if needed

4. Merge for inference
   - LoRA can merge into base weights
   - Zero inference overhead
```

---

## When to Use

### ✅ Good Fit:
- Limited GPU memory
- Multiple tasks from one base model
- Quick experimentation
- Edge deployment

### ❌ Consider Full FT:
- Maximum quality needed
- Single-task deployment
- Abundant compute resources

---

## Interview Tips

When discussing PEFT:
1. Explain the memory/storage problem it solves
2. Describe LoRA's low-rank decomposition
3. Compare different PEFT methods
4. Know when to use which method
5. Mention multi-task adapter switching

