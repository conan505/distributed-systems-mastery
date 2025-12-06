# Parameter-Efficient Fine-Tuning (PEFT)

## What It Is

**Parameter-Efficient Fine-Tuning (PEFT)** is a family of techniques that adapt large pre-trained models to downstream tasks by training only a small fraction of parameters, dramatically reducing computational and memory requirements.

---

## The Analogy 🎨

Think of **customizing a car**:
- Don't rebuild the engine (base model)
- Just add accessories, tune settings
- New paint job, custom seats (adapters)
- Same performance, personalized style
- Much cheaper than buying new car

---

## Why It Exists

### Full Fine-Tuning Problems
```
GPT-3 (175B parameters):
- Storage: 700GB per fine-tuned copy
- Training: 100s of GPUs, days of compute
- Risk: Catastrophic forgetting

PEFT Solution:
- Train 0.1-1% of parameters
- Storage: 1-10MB per adapter
- Training: Single GPU, hours
- Preserves base capabilities
```

---

## PEFT Methods Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│              PEFT Methods Overview                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Method          │ Params │ Where           │ Training Cost   │
│   ────────────────│────────│─────────────────│─────────────────│
│   LoRA            │ 0.1%   │ Attention       │ Low             │
│   QLoRA           │ 0.1%   │ Quantized+LoRA  │ Very Low        │
│   Adapters        │ 1-5%   │ After layers    │ Medium          │
│   Prefix Tuning   │ <0.1%  │ Input prefix    │ Low             │
│   Prompt Tuning   │ <0.01% │ Soft prompts    │ Very Low        │
│   IA³             │ <0.01% │ Activation mult │ Very Low        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. LoRA (Low-Rank Adaptation)

```python
class LoRALinear(nn.Module):
    """Low-Rank Adaptation layer."""
    
    def __init__(self, original_layer, rank=8, alpha=16):
        super().__init__()
        self.original = original_layer
        self.original.weight.requires_grad = False
        
        in_features = original_layer.in_features
        out_features = original_layer.out_features
        
        # Low-rank matrices
        self.lora_A = nn.Parameter(torch.randn(in_features, rank))
        self.lora_B = nn.Parameter(torch.zeros(rank, out_features))
        
        self.scaling = alpha / rank
    
    def forward(self, x):
        original_output = self.original(x)
        lora_output = (x @ self.lora_A @ self.lora_B) * self.scaling
        return original_output + lora_output
```

See [LoRA](./lora.md) for detailed explanation.

---

## 2. Adapters

```python
class Adapter(nn.Module):
    """Bottleneck adapter inserted after attention/FFN."""
    
    def __init__(self, hidden_size, bottleneck_size=64):
        super().__init__()
        self.down_proj = nn.Linear(hidden_size, bottleneck_size)
        self.up_proj = nn.Linear(bottleneck_size, hidden_size)
        self.activation = nn.GELU()
    
    def forward(self, x):
        # Bottleneck: compress then expand
        residual = x
        x = self.down_proj(x)
        x = self.activation(x)
        x = self.up_proj(x)
        return x + residual  # Skip connection
```

---

## 3. Prefix Tuning

```python
class PrefixTuning(nn.Module):
    """Learnable prefix tokens prepended to keys/values."""
    
    def __init__(self, num_layers, num_heads, head_dim, prefix_length=20):
        super().__init__()
        self.prefix_length = prefix_length
        
        # Learnable prefix for each layer
        self.prefix_keys = nn.ParameterList([
            nn.Parameter(torch.randn(prefix_length, num_heads, head_dim))
            for _ in range(num_layers)
        ])
        self.prefix_values = nn.ParameterList([
            nn.Parameter(torch.randn(prefix_length, num_heads, head_dim))
            for _ in range(num_layers)
        ])
    
    def get_prefix(self, layer_idx, batch_size):
        # Expand prefix to batch size
        prefix_k = self.prefix_keys[layer_idx].unsqueeze(0).expand(batch_size, -1, -1, -1)
        prefix_v = self.prefix_values[layer_idx].unsqueeze(0).expand(batch_size, -1, -1, -1)
        return prefix_k, prefix_v
```

---

## 4. Prompt Tuning

```python
class PromptTuning(nn.Module):
    """Learnable soft prompts prepended to input."""
    
    def __init__(self, num_virtual_tokens=20, embedding_dim=768):
        super().__init__()
        # Virtual tokens (not in vocabulary)
        self.soft_prompt = nn.Parameter(
            torch.randn(num_virtual_tokens, embedding_dim)
        )
    
    def forward(self, input_embeddings):
        batch_size = input_embeddings.size(0)
        
        # Expand prompt for batch
        prompts = self.soft_prompt.unsqueeze(0).expand(batch_size, -1, -1)
        
        # Prepend to input
        return torch.cat([prompts, input_embeddings], dim=1)
```

---

## 5. IA³ (Infused Adapter by Inhibiting and Amplifying)

```python
class IA3Layer(nn.Module):
    """Learned rescaling vectors for activations."""
    
    def __init__(self, hidden_size):
        super().__init__()
        # Learned scaling factors (very few parameters!)
        self.l_k = nn.Parameter(torch.ones(hidden_size))  # Key scaling
        self.l_v = nn.Parameter(torch.ones(hidden_size))  # Value scaling
        self.l_ff = nn.Parameter(torch.ones(hidden_size)) # FFN scaling
    
    def rescale_attention(self, keys, values):
        return keys * self.l_k, values * self.l_v
    
    def rescale_ffn(self, activations):
        return activations * self.l_ff
```

---

## Using PEFT with Hugging Face

```python
from peft import get_peft_model, LoraConfig, TaskType

# Configure LoRA
peft_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=8,                      # Rank
    lora_alpha=16,            # Scaling factor
    lora_dropout=0.1,
    target_modules=["q_proj", "v_proj"]  # Which layers
)

# Apply to model
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b")
peft_model = get_peft_model(model, peft_config)

# Check trainable parameters
peft_model.print_trainable_parameters()
# Output: trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.06%
```

---

## When to Use Each Method

| Method | Best For | Trade-off |
|--------|----------|-----------|
| **LoRA** | General fine-tuning | Good balance |
| **QLoRA** | Limited GPU memory | Slightly slower |
| **Adapters** | Multi-task serving | More parameters |
| **Prefix Tuning** | Generation tasks | Context length cost |
| **Prompt Tuning** | Classification | Limited flexibility |
| **IA³** | Minimal parameters | Less expressive |

---

## Multi-Adapter Serving

```
Serve multiple fine-tuned models efficiently:

Base Model (Llama-7B): 14GB VRAM
+ LoRA Adapter 1 (Code): 10MB
+ LoRA Adapter 2 (Chat): 10MB
+ LoRA Adapter 3 (Medical): 10MB

Hot-swap adapters per request!
Same base model, different behaviors
```

---

## Interview Tips

When discussing PEFT:
1. Explain why full fine-tuning is expensive
2. Compare major methods (LoRA, Adapters, Prefix)
3. Know parameter counts (0.1% typical)
4. Discuss multi-adapter serving
5. Mention QLoRA for memory efficiency

