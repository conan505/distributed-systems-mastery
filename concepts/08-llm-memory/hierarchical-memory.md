# Hierarchical Memory Systems

## What It Is

**Hierarchical Memory** in LLMs refers to multi-tiered memory architectures that organize information at different levels of granularity and accessibility, enabling efficient handling of both short-term context and long-term knowledge.

---

## The Analogy 🏛️

Think of **human memory**:
- **Working memory**: What you're thinking about now (7±2 items)
- **Short-term**: Recent events (hours/days)
- **Long-term**: Learned knowledge (years)
- **Procedural**: Skills and habits (automatic)

Each level has different capacity, speed, and persistence.

---

## Why It Exists

### Single-Level Limitations
```
Standard transformer:
- Only one memory level: Context window
- Fixed size (4K-128K tokens)
- No persistence across sessions
- Can't learn from interactions

What's needed:
- Fast access for recent context
- Large storage for knowledge
- Persistence for learning
- Efficient retrieval
```

---

## Memory Hierarchy Levels

```
┌─────────────────────────────────────────────────────────────────┐
│                   LLM Memory Hierarchy                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Level 0: Immediate (Attention)                                │
│   ├── Current token context                                     │
│   ├── Full attention within context                             │
│   └── Fastest access, smallest capacity                         │
│                                                                  │
│   Level 1: Working Memory (KV Cache)                            │
│   ├── Recent tokens and their representations                   │
│   ├── Fast access, limited by GPU memory                        │
│   └── Session-persistent                                        │
│                                                                  │
│   Level 2: Episodic Memory (Conversation Buffer)                │
│   ├── Summarized conversation history                           │
│   ├── Compressed older context                                  │
│   └── Sliding window with summarization                         │
│                                                                  │
│   Level 3: Semantic Memory (Vector DB)                          │
│   ├── Embedded knowledge chunks                                 │
│   ├── Retrieved on demand                                       │
│   └── Large capacity, slower access                             │
│                                                                  │
│   Level 4: Parametric Memory (Model Weights)                    │
│   ├── Learned during training                                   │
│   ├── Static during inference                                   │
│   └── Largest capacity, most persistent                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Patterns

### Memory Manager
```python
class HierarchicalMemory:
    def __init__(self, config):
        # Level 1: Working memory (recent context)
        self.working_memory = deque(maxlen=config.working_size)
        
        # Level 2: Episodic memory (summarized history)
        self.episodic_memory = []
        self.summarizer = Summarizer()
        
        # Level 3: Semantic memory (vector store)
        self.semantic_memory = VectorStore()
        
    def add(self, content: str, importance: float = 0.5):
        """Add content to appropriate memory level."""
        # Always add to working memory
        self.working_memory.append({
            "content": content,
            "timestamp": time.time(),
            "importance": importance
        })
        
        # High importance → also store in semantic memory
        if importance > 0.7:
            embedding = self.embed(content)
            self.semantic_memory.add(content, embedding)
    
    def consolidate(self):
        """Move old working memory to episodic memory."""
        if len(self.working_memory) >= self.working_memory.maxlen:
            # Summarize oldest entries
            old_entries = list(self.working_memory)[:len(self.working_memory)//2]
            summary = self.summarizer.summarize(old_entries)
            self.episodic_memory.append(summary)
    
    def retrieve(self, query: str, k: int = 5) -> list:
        """Retrieve relevant memories from all levels."""
        results = []
        
        # Level 1: Recent working memory
        results.extend(list(self.working_memory)[-3:])
        
        # Level 2: Relevant episodic memories
        for episode in self.episodic_memory[-5:]:
            if self.is_relevant(query, episode):
                results.append(episode)
        
        # Level 3: Semantic search
        semantic_results = self.semantic_memory.search(query, k=k)
        results.extend(semantic_results)
        
        return results
```

---

## MemGPT Architecture

```
MemGPT: OS-inspired memory management for LLMs

┌─────────────────────────────────────────────────────────────────┐
│                      MemGPT System                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Main Context (Limited, ~8K tokens)                            │
│   ├── System prompt                                             │
│   ├── Core memory (persona, user info)                          │
│   ├── Recent messages (working memory)                          │
│   └── Retrieved context                                         │
│                                                                  │
│   External Storage (Unlimited)                                  │
│   ├── Archival memory (vector DB)                               │
│   ├── Recall memory (conversation logs)                         │
│   └── Files and documents                                       │
│                                                                  │
│   LLM has tools to:                                             │
│   - core_memory_append/replace                                  │
│   - archival_memory_insert/search                               │
│   - conversation_search                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Memory Compression

### Recursive Summarization
```python
def compress_memory(memories: list, target_tokens: int) -> str:
    """Recursively summarize until within token budget."""
    
    if count_tokens(memories) <= target_tokens:
        return "\n".join(memories)
    
    # Group and summarize
    chunks = chunk_memories(memories, chunk_size=5)
    summaries = [summarize(chunk) for chunk in chunks]
    
    # Recursively compress if still too large
    return compress_memory(summaries, target_tokens)
```

### Importance-Based Pruning
```python
def prune_memory(memories: list, keep_ratio: float = 0.5) -> list:
    """Keep most important memories."""
    
    # Score each memory
    scored = [(m, compute_importance(m)) for m in memories]
    
    # Sort by importance
    scored.sort(key=lambda x: x[1], reverse=True)
    
    # Keep top percentage
    keep_count = int(len(scored) * keep_ratio)
    return [m for m, _ in scored[:keep_count]]
```

---

## Access Patterns

```
Memory Access by Task:

Question Answering:
1. Parse question → Extract entities
2. Search semantic memory → Find relevant docs
3. Load into working memory
4. Generate answer

Conversation:
1. Check working memory → Recent context
2. Retrieve user preferences → Episodic
3. Search relevant knowledge → Semantic
4. Compose response

Learning:
1. Process new information
2. Store in working memory
3. Consolidate to episodic (summaries)
4. Index important content (semantic)
```

---

## Real-World Systems

| System | Architecture |
|--------|--------------|
| **MemGPT** | OS-inspired paging |
| **LangChain Memory** | Conversation buffer + summary |
| **ChatGPT** | Context + memory tool |
| **AutoGPT** | Short + long term stores |
| **Voyager** | Skill library + curriculum |

---

## Best Practices

```
1. Define clear memory tiers
   - What goes where
   - When to promote/demote

2. Implement consolidation
   - Summarize older memories
   - Prune low-importance items

3. Balance retrieval sources
   - Recent context (high weight)
   - Semantic search (relevant)
   - Core knowledge (always present)

4. Handle memory conflicts
   - Newer info updates older
   - Track confidence/recency

5. Enable forgetting
   - Remove outdated information
   - Limit memory growth
```

---

## Interview Tips

When discussing Hierarchical Memory:
1. Explain the analogy to human memory
2. Describe different memory levels
3. Discuss consolidation and compression
4. Mention MemGPT or similar systems
5. Know trade-offs (capacity vs speed vs persistence)

