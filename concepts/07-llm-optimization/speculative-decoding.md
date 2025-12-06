# Speculative Decoding

## What It Is

**Speculative Decoding** accelerates LLM inference by using a smaller, faster model to draft tokens, then having the larger model verify multiple tokens in parallel, reducing the number of forward passes needed.

---

## The Analogy 📝

Think of **hiring an assistant for a CEO**:
- **Without speculation**: CEO writes every word themselves (slow)
- **With speculation**: Assistant drafts document, CEO reviews quickly
- CEO accepts most of it, rewrites some parts
- Much faster overall, same quality output

---

## Why It Exists

### The LLM Inference Problem
```
Autoregressive generation is slow:

Token 1: Forward pass → Generate
Token 2: Forward pass → Generate
Token 3: Forward pass → Generate
...
Token N: Forward pass → Generate

Each token requires full forward pass
70B model: ~50ms per token
100 tokens: 5 seconds total

Problem: GPU utilization is low during autoregressive decoding
```

### The Solution
```
Use draft model to speculate multiple tokens:

Draft (small model): Generate 5 tokens quickly (~10ms)
Verify (large model): Check all 5 in ONE forward pass (~50ms)

If 4/5 accepted: 4 tokens in 60ms instead of 200ms
Speedup: 3-4x!
```

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                  Speculative Decoding Flow                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Step 1: Draft Model generates K tokens                        │
│   ┌──────────┐                                                  │
│   │ Draft    │ → "The quick brown fox jumps"                   │
│   │ (1B)     │   [t1, t2, t3, t4, t5]                          │
│   └──────────┘                                                  │
│                                                                  │
│   Step 2: Target Model verifies in parallel                     │
│   ┌──────────┐                                                  │
│   │ Target   │ → Accepts: "The quick brown"                    │
│   │ (70B)    │   Rejects: "fox" (should be "red")             │
│   └──────────┘                                                  │
│                                                                  │
│   Step 3: Accept valid prefix + resample rejected              │
│   Output: "The quick brown red" (4 tokens in 1 pass!)          │
│                                                                  │
│   Repeat from last accepted token                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Algorithm

```python
def speculative_decode(
    target_model,    # Large model (70B)
    draft_model,     # Small model (1B)
    prompt: str,
    num_speculative: int = 5,
    max_tokens: int = 100
) -> str:
    
    tokens = tokenize(prompt)
    
    while len(tokens) < max_tokens:
        # Step 1: Draft model generates K speculative tokens
        draft_tokens = []
        draft_probs = []
        
        for _ in range(num_speculative):
            prob = draft_model(tokens + draft_tokens)
            next_token = sample(prob)
            draft_tokens.append(next_token)
            draft_probs.append(prob[next_token])
        
        # Step 2: Target model evaluates all at once
        target_probs = target_model.batch_forward(
            tokens, draft_tokens
        )
        
        # Step 3: Accept/reject each token
        accepted = 0
        for i, (t, d_prob, t_prob) in enumerate(
            zip(draft_tokens, draft_probs, target_probs)
        ):
            # Rejection sampling
            if random() < min(1, t_prob[t] / d_prob):
                tokens.append(t)
                accepted += 1
            else:
                # Resample from adjusted distribution
                tokens.append(sample_adjusted(t_prob, d_prob))
                break
        
        # Always generate at least one token
        if accepted == len(draft_tokens):
            tokens.append(sample(target_probs[-1]))
    
    return detokenize(tokens)
```

---

## Acceptance Rate

```
Key insight: Acceptance probability depends on how well
draft model matches target model

If P_draft ≈ P_target:
- Most tokens accepted
- High speedup

If P_draft ≠ P_target:
- Many rejections
- Low speedup

Typical acceptance rates:
- Same architecture family: 70-90%
- Different architectures: 50-70%
```

---

## Draft Model Selection

| Strategy | Draft Model | Acceptance Rate |
|----------|-------------|-----------------|
| **Same family** | Llama-7B for Llama-70B | ~80% |
| **Quantized** | 4-bit version | ~75% |
| **Distilled** | Trained on target outputs | ~85% |
| **n-gram** | Statistical model | ~40% |

---

## Speedup Analysis

```python
def expected_speedup(
    acceptance_rate: float,  # α
    draft_speed: float,      # t_d
    target_speed: float,     # t_t
    num_speculative: int     # K
) -> float:
    """
    Calculate expected speedup from speculative decoding.
    """
    # Expected tokens per iteration
    expected_tokens = sum(
        acceptance_rate ** i 
        for i in range(num_speculative + 1)
    )
    
    # Time per iteration
    time_per_iter = (num_speculative * draft_speed) + target_speed
    
    # Baseline time for same tokens
    baseline_time = expected_tokens * target_speed
    
    return baseline_time / time_per_iter

# Example:
# α=0.8, draft=5ms, target=50ms, K=5
# Expected tokens: 1 + 0.8 + 0.64 + 0.51 + 0.41 + 0.33 = 3.69
# Time: 5*5 + 50 = 75ms
# Baseline: 3.69 * 50 = 184ms
# Speedup: 184/75 = 2.45x
```

---

## Variants

### 1. Medusa (Multi-Head)
```
Instead of draft model, add prediction heads to target model

Target model with multiple heads:
- Head 1: predicts token t+1
- Head 2: predicts token t+2
- Head 3: predicts token t+3

Verify all predictions in parallel
No separate draft model needed!
```

### 2. SpecInfer (Tree-Based)
```
Draft multiple possible continuations as tree:
        "The"
       /    \
    "quick" "slow"
     /   \
  "brown" "red"

Verify entire tree in one pass
Higher parallelism
```

### 3. Lookahead Decoding
```
Use Jacobi iteration to solve for multiple tokens:
- Initialize random tokens
- Iteratively refine until convergence
- Verify all at once
```

---

## Implementation (Hugging Face)

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

# Load models
target = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-70b")
draft = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b")

# Generate with speculative decoding
output = target.generate(
    input_ids,
    assistant_model=draft,  # Draft model
    max_new_tokens=100,
    do_sample=False,
)
```

---

## Best Practices

```
1. Choose draft model from same family
2. Tune num_speculative (5-10 typical)
3. Use greedy decoding for best acceptance
4. Profile to find optimal draft model size
5. Consider quantized draft model
6. Monitor acceptance rate in production
```

---

## Interview Tips

When discussing Speculative Decoding:
1. Explain the autoregressive bottleneck
2. Describe draft-then-verify process
3. Explain rejection sampling for correctness
4. Know speedup factors (2-4x typical)
5. Mention Medusa as alternative

