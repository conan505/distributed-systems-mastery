# Episodic Memory

## What It Is

**Episodic Memory** in LLMs refers to the ability to store, retrieve, and reason about specific past experiences or interactions, enabling the model to remember and reference previous conversations, events, or learned information.

---

## The Analogy 📔

Think of a **personal diary**:
- Record specific events with context
- "On Tuesday, user asked about Python"
- Can flip back to find relevant entries
- Builds understanding over time
- Different from general knowledge (semantic memory)

---

## Why It Exists

### LLM Memory Limitations
```
Standard LLM:
- No memory between sessions
- Context window is only "memory"
- Can't learn from past interactions
- Treats each conversation as new

With Episodic Memory:
- Remember past conversations
- Learn user preferences
- Build on previous interactions
- Personalized responses
```

---

## Episodic vs Semantic Memory

```
┌─────────────────────────────────────────────────────────────────┐
│           Episodic vs Semantic Memory                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Semantic Memory (Model Weights):                              │
│   - "Python is a programming language"                          │
│   - General facts and knowledge                                 │
│   - Learned during training                                     │
│   - Static during inference                                     │
│                                                                  │
│   Episodic Memory (External Store):                             │
│   - "User prefers Python over JavaScript"                       │
│   - Specific experiences and events                             │
│   - Accumulated during interactions                             │
│   - Dynamic and updatable                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation

### Basic Episodic Memory Store
```python
from dataclasses import dataclass
from datetime import datetime
import numpy as np

@dataclass
class Episode:
    content: str
    timestamp: datetime
    embedding: np.ndarray
    metadata: dict
    importance: float = 0.5

class EpisodicMemory:
    def __init__(self, embedding_model, max_episodes: int = 1000):
        self.embedding_model = embedding_model
        self.episodes = []
        self.max_episodes = max_episodes
    
    def add_episode(self, content: str, metadata: dict = None):
        """Store a new episode."""
        embedding = self.embedding_model.encode(content)
        importance = self._compute_importance(content)
        
        episode = Episode(
            content=content,
            timestamp=datetime.now(),
            embedding=embedding,
            metadata=metadata or {},
            importance=importance
        )
        
        self.episodes.append(episode)
        self._consolidate_if_needed()
    
    def retrieve(self, query: str, k: int = 5) -> list[Episode]:
        """Retrieve relevant episodes."""
        query_embedding = self.embedding_model.encode(query)
        
        # Score by similarity and recency
        scored = []
        for ep in self.episodes:
            similarity = np.dot(query_embedding, ep.embedding)
            recency = self._recency_score(ep.timestamp)
            score = 0.7 * similarity + 0.2 * recency + 0.1 * ep.importance
            scored.append((ep, score))
        
        scored.sort(key=lambda x: x[1], reverse=True)
        return [ep for ep, _ in scored[:k]]
    
    def _consolidate_if_needed(self):
        """Remove old, low-importance episodes."""
        if len(self.episodes) > self.max_episodes:
            # Keep high importance and recent
            self.episodes.sort(
                key=lambda e: e.importance + self._recency_score(e.timestamp),
                reverse=True
            )
            self.episodes = self.episodes[:self.max_episodes]
```

### Integration with LLM
```python
class EpisodicLLM:
    def __init__(self, llm, memory: EpisodicMemory):
        self.llm = llm
        self.memory = memory
    
    def chat(self, user_message: str) -> str:
        # Retrieve relevant past episodes
        relevant_episodes = self.memory.retrieve(user_message, k=3)
        
        # Build context with episodes
        context = "Relevant past interactions:\n"
        for ep in relevant_episodes:
            context += f"- [{ep.timestamp.strftime('%Y-%m-%d')}]: {ep.content}\n"
        
        # Generate response
        prompt = f"{context}\n\nUser: {user_message}\nAssistant:"
        response = self.llm.generate(prompt)
        
        # Store this interaction as new episode
        self.memory.add_episode(
            f"User asked: {user_message}. I responded: {response}",
            metadata={"type": "conversation"}
        )
        
        return response
```

---

## Memory Operations

### 1. Encoding
```python
def encode_episode(interaction: dict) -> Episode:
    """Convert interaction to storable episode."""
    # Extract key information
    summary = summarize(interaction)
    entities = extract_entities(interaction)
    
    return Episode(
        content=summary,
        embedding=embed(summary),
        metadata={
            "entities": entities,
            "topic": classify_topic(interaction),
            "sentiment": analyze_sentiment(interaction)
        }
    )
```

### 2. Retrieval
```python
def retrieve_episodes(query: str, memory: list[Episode]) -> list[Episode]:
    """Multi-factor retrieval."""
    scores = []
    for ep in memory:
        # Semantic similarity
        semantic = cosine_similarity(embed(query), ep.embedding)
        
        # Temporal relevance (recent = higher)
        temporal = 1 / (1 + days_since(ep.timestamp))
        
        # Importance weighting
        importance = ep.importance
        
        # Combined score
        score = 0.5 * semantic + 0.3 * temporal + 0.2 * importance
        scores.append((ep, score))
    
    return sorted(scores, key=lambda x: x[1], reverse=True)
```

### 3. Consolidation
```python
def consolidate_memories(episodes: list[Episode]) -> list[Episode]:
    """Merge similar episodes, summarize old ones."""
    # Group similar episodes
    clusters = cluster_by_similarity(episodes)
    
    consolidated = []
    for cluster in clusters:
        if len(cluster) > 3:
            # Merge into summary episode
            merged = Episode(
                content=summarize([e.content for e in cluster]),
                timestamp=max(e.timestamp for e in cluster),
                importance=max(e.importance for e in cluster)
            )
            consolidated.append(merged)
        else:
            consolidated.extend(cluster)
    
    return consolidated
```

---

## Forgetting Mechanisms

```python
def apply_forgetting(memory: EpisodicMemory, decay_rate: float = 0.01):
    """Gradually reduce importance of old memories."""
    for episode in memory.episodes:
        age_days = (datetime.now() - episode.timestamp).days
        
        # Exponential decay
        decay = np.exp(-decay_rate * age_days)
        episode.importance *= decay
    
    # Remove very low importance
    memory.episodes = [
        e for e in memory.episodes 
        if e.importance > 0.1
    ]
```

---

## Real-World Applications

| System | Episodic Memory Use |
|--------|---------------------|
| **ChatGPT Memory** | Stores user preferences |
| **Character.AI** | Remembers conversation history |
| **Personal Assistants** | Tracks user habits |
| **Customer Support** | Recalls past tickets |
| **Tutoring Systems** | Tracks learning progress |

---

## Best Practices

```
1. Store summaries, not raw text
   - Compress for efficiency
   - Extract key information

2. Use multi-factor retrieval
   - Semantic similarity
   - Temporal recency
   - Importance weighting

3. Implement consolidation
   - Merge similar memories
   - Summarize old episodes

4. Enable forgetting
   - Decay old memories
   - Remove irrelevant ones

5. Respect privacy
   - Allow memory deletion
   - Encrypt sensitive data
```

---

## Interview Tips

When discussing Episodic Memory:
1. Contrast with semantic memory (weights)
2. Explain storage and retrieval mechanisms
3. Discuss importance scoring
4. Mention consolidation and forgetting
5. Give real-world examples (ChatGPT Memory)

