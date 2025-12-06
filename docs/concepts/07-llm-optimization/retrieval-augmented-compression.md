# Retrieval-Augmented Compression

## What It Is

**Retrieval-Augmented Compression** combines retrieval techniques with model compression to reduce LLM size while maintaining knowledge access through external retrieval, effectively offloading knowledge from parameters to searchable databases.

---

## The Analogy 📚

Think of a **student with a textbook**:
- Don't memorize every fact (smaller brain/model)
- Know how to find information (retrieval)
- Remember core skills and reasoning (compressed model)
- Look up specifics when needed (knowledge base)
- Effective with less memorization

---

## Why It Exists

### The Knowledge-Size Trade-off
```
Large models store knowledge in parameters:
- GPT-3: 175B params → ~700GB
- Much of this is factual knowledge
- Facts change, models don't update easily
- Redundant storage across instances

RAC Solution:
- Smaller model for reasoning (7B params)
- External database for facts
- Update knowledge by updating DB
- Share DB across model instances
```

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│           Retrieval-Augmented Compression                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Standard LLM (175B params):                                   │
│   ┌─────────────────────────────────────────────┐              │
│   │  Reasoning  │  World Knowledge  │  Language  │              │
│   │    (20%)    │      (60%)        │   (20%)    │              │
│   └─────────────────────────────────────────────┘              │
│                                                                  │
│   RAC Approach:                                                 │
│   ┌───────────────────────┐   ┌─────────────────────┐          │
│   │ Compressed Model (7B) │ + │  Knowledge Base     │          │
│   │ Reasoning + Language  │   │  (Vector DB)        │          │
│   └───────────────────────┘   └─────────────────────┘          │
│                                                                  │
│   Query → Retrieve relevant knowledge → Generate response       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Techniques

### 1. Knowledge Distillation with Retrieval
```python
class RetrievalDistillation:
    """Train smaller model to use retrieval effectively."""
    
    def __init__(self, teacher_model, student_model, retriever):
        self.teacher = teacher_model  # Large, no retrieval
        self.student = student_model  # Small, with retrieval
        self.retriever = retriever
    
    def train_step(self, query):
        # Teacher generates answer (using internal knowledge)
        teacher_answer = self.teacher.generate(query)
        
        # Student retrieves context, then generates
        retrieved_docs = self.retriever.search(query)
        context = format_docs(retrieved_docs)
        student_answer = self.student.generate(query, context=context)
        
        # Train student to match teacher
        loss = compute_distillation_loss(teacher_answer, student_answer)
        return loss
```

### 2. RETRO-fitting Existing Models
```python
def retrofit_with_retrieval(base_model, knowledge_corpus):
    """Add retrieval capability to existing model."""
    
    # Index knowledge corpus
    embeddings = embed_corpus(knowledge_corpus)
    index = build_faiss_index(embeddings)
    
    # Add cross-attention layers for retrieved context
    retrieval_layers = nn.ModuleList([
        CrossAttentionLayer(base_model.d_model)
        for _ in range(base_model.num_layers)
    ])
    
    # Fine-tune to use retrieved context
    return RetrievalAugmentedModel(base_model, index, retrieval_layers)
```

### 3. Selective Knowledge Externalization
```python
def externalize_factual_knowledge(model, knowledge_db):
    """Move factual knowledge from params to retrieval."""
    
    # Identify factual vs procedural knowledge
    factual_queries = identify_factual_knowledge(model)
    
    # Extract and store in knowledge base
    for query in factual_queries:
        answer = model.generate(query)
        knowledge_db.add(query, answer)
    
    # Prune model's factual parameters (if possible)
    # Fine-tune to rely on retrieval for facts
```

---

## RETRO Architecture

```
Retrieval-Enhanced Transformer (RETRO):

1. Chunk input into 64-token segments
2. For each chunk, retrieve similar chunks from corpus
3. Encode retrieved chunks with separate encoder
4. Cross-attend to retrieved context in decoder
5. Generate output

Result:
- 25x smaller than GPT-3
- Comparable performance on knowledge tasks
- Easy to update knowledge (change corpus)
```

```python
class RETROBlock(nn.Module):
    def __init__(self, d_model, n_heads):
        super().__init__()
        self.self_attention = SelfAttention(d_model, n_heads)
        self.cross_attention = CrossAttention(d_model, n_heads)  # For retrieval
        self.ffn = FeedForward(d_model)
    
    def forward(self, x, retrieved_context):
        # Standard self-attention
        x = x + self.self_attention(x)
        
        # Cross-attend to retrieved context
        if retrieved_context is not None:
            x = x + self.cross_attention(x, retrieved_context)
        
        # Feed-forward
        x = x + self.ffn(x)
        return x
```

---

## Compression Strategies with Retrieval

### Aggressive Quantization + Retrieval
```
Without retrieval: 4-bit quantization loses factual accuracy
With retrieval: 4-bit model for reasoning, facts from DB

Result: 4-bit model + retrieval ≈ 16-bit model quality
```

### Pruning Knowledge Heads
```python
def prune_knowledge_heads(model, importance_scores):
    """Prune attention heads focused on factual recall."""
    
    # Identify heads that encode factual knowledge
    factual_heads = identify_factual_heads(model, importance_scores)
    
    # Prune these heads (rely on retrieval instead)
    for layer_idx, head_idx in factual_heads:
        prune_head(model, layer_idx, head_idx)
    
    # Fine-tune to use retrieval for factual queries
    return model
```

---

## Benefits

```
1. Smaller model size
   - Core model handles reasoning
   - Facts offloaded to DB

2. Updatable knowledge
   - Change DB, not model
   - No retraining needed

3. Verifiable sources
   - Can cite retrieved documents
   - Reduces hallucination

4. Domain adaptation
   - Swap knowledge bases for different domains
   - Same model, different expertise
```

---

## Trade-offs

```
Pros:
+ Much smaller model
+ Updateable knowledge
+ Reduced hallucination
+ Shareable knowledge base

Cons:
- Retrieval latency
- Retrieval failures
- Complex system
- Quality depends on corpus
```

---

## Use Cases

```
1. Enterprise knowledge bases
   - Company docs + small model

2. Frequently updated domains
   - News, regulations, pricing

3. Multi-domain deployment
   - One model, many knowledge bases

4. Resource-constrained environments
   - Edge deployment with remote KB
```

---

## Interview Tips

When discussing RAC:
1. Explain knowledge-in-params vs retrieval trade-off
2. Describe RETRO architecture
3. Know benefits (updateable, smaller, verifiable)
4. Discuss retrieval latency concerns
5. Compare with pure compression methods

