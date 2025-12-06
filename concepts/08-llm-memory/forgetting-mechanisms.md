# Forgetting Mechanisms

## What It Is

**Forgetting Mechanisms** in LLM memory systems are strategies for selectively removing or deprioritizing information to manage memory capacity, improve relevance, and prevent outdated knowledge from affecting responses.

---

## The Analogy 🧹

Think of **cleaning out a closet**:
- Limited space (memory capacity)
- Keep what you use often
- Donate rarely worn items
- Throw away worn-out clothes
- Make room for new purchases

---

## Why It Exists

### Memory Without Forgetting
```
Problems of infinite memory:
1. Storage costs grow unbounded
2. Retrieval becomes slower
3. Outdated info conflicts with current
4. Noise drowns out signal
5. Privacy concerns (can't delete)

Forgetting is a feature, not a bug!
- Keeps memory relevant
- Improves retrieval quality
- Reduces resource usage
- Enables updates
```

---

## Forgetting Strategies

### 1. Time-Based Decay
```python
import math
from datetime import datetime

def exponential_decay(memory_item, current_time, half_life_days=30):
    """Exponential decay based on age."""
    age_days = (current_time - memory_item.created_at).days
    decay_factor = math.exp(-0.693 * age_days / half_life_days)
    return memory_item.importance * decay_factor

def power_law_decay(memory_item, current_time, exponent=0.5):
    """Power law decay (slower than exponential)."""
    age_days = (current_time - memory_item.created_at).days + 1
    decay_factor = 1.0 / (age_days ** exponent)
    return memory_item.importance * decay_factor
```

### 2. Access-Based (LRU-like)
```python
class LRUMemory:
    """Forget least recently used memories."""
    
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.memories = OrderedDict()
    
    def access(self, key: str) -> any:
        if key in self.memories:
            # Move to end (most recent)
            self.memories.move_to_end(key)
            return self.memories[key]
        return None
    
    def add(self, key: str, value: any):
        if key in self.memories:
            self.memories.move_to_end(key)
        else:
            if len(self.memories) >= self.capacity:
                # Remove oldest (least recently used)
                self.memories.popitem(last=False)
            self.memories[key] = value
```

### 3. Importance-Based
```python
class ImportanceBasedMemory:
    """Keep most important memories."""
    
    def compute_importance(self, memory) -> float:
        """Multi-factor importance scoring."""
        factors = {
            'explicit_importance': memory.importance,
            'access_frequency': memory.access_count / max_access,
            'recency': self.recency_score(memory),
            'relevance': memory.avg_retrieval_score,
            'uniqueness': memory.uniqueness_score,
        }
        
        weights = {
            'explicit_importance': 0.3,
            'access_frequency': 0.25,
            'recency': 0.2,
            'relevance': 0.15,
            'uniqueness': 0.1,
        }
        
        return sum(factors[k] * weights[k] for k in factors)
    
    def forget_if_needed(self, keep_ratio=0.8):
        if len(self.memories) > self.capacity:
            # Score all memories
            scored = [(m, self.compute_importance(m)) for m in self.memories]
            scored.sort(key=lambda x: x[1], reverse=True)
            
            # Keep top percentage
            keep_count = int(len(scored) * keep_ratio)
            self.memories = [m for m, _ in scored[:keep_count]]
```

### 4. Contradiction-Based
```python
def resolve_contradictions(memories: List[Memory]) -> List[Memory]:
    """Remove memories that contradict newer information."""
    # Group by topic
    by_topic = defaultdict(list)
    for m in memories:
        by_topic[m.topic].append(m)
    
    resolved = []
    for topic, topic_memories in by_topic.items():
        # Sort by timestamp
        topic_memories.sort(key=lambda m: m.timestamp)
        
        # Check for contradictions
        for i, m in enumerate(topic_memories):
            is_contradicted = False
            for later_m in topic_memories[i+1:]:
                if contradicts(m.content, later_m.content):
                    is_contradicted = True
                    break
            
            if not is_contradicted:
                resolved.append(m)
    
    return resolved
```

---

## Soft vs Hard Forgetting

```
┌─────────────────────────────────────────────────────────────────┐
│              Soft vs Hard Forgetting                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   HARD FORGETTING:                                              │
│   - Delete memory permanently                                   │
│   - Free storage immediately                                    │
│   - Cannot be recovered                                         │
│   Use: Privacy, clear errors                                    │
│                                                                  │
│   SOFT FORGETTING:                                              │
│   - Reduce retrieval priority                                   │
│   - Move to cold storage                                        │
│   - Can be recovered if needed                                  │
│   Use: Deprioritize, archive                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Consolidation-Based Forgetting

```python
def consolidate_and_forget(memories: List[Memory], llm) -> List[Memory]:
    """Summarize similar memories, then forget originals."""
    
    # Cluster similar memories
    clusters = cluster_memories(memories, threshold=0.85)
    
    consolidated = []
    for cluster in clusters:
        if len(cluster) == 1:
            consolidated.append(cluster[0])
        else:
            # Summarize cluster
            contents = [m.content for m in cluster]
            summary = llm.generate(
                f"Summarize these related memories:\n{contents}"
            )
            
            # Create consolidated memory
            consolidated.append(Memory(
                content=summary,
                importance=max(m.importance for m in cluster),
                timestamp=max(m.timestamp for m in cluster),
                source_count=len(cluster)
            ))
    
    return consolidated
```

---

## Spaced Repetition

```
Inspired by human memory:

New memory → Review soon (1 day)
Remembered → Review later (3 days)
Remembered → Review even later (7 days)
...
Forgotten → Reset review schedule

Memories that survive reviews become permanent
Others naturally fade away
```

---

## Privacy-Aware Forgetting

```python
class PrivacyAwareMemory:
    def forget_user_data(self, user_id: str):
        """GDPR-compliant forgetting."""
        # Hard delete all user memories
        self.memories = [m for m in self.memories 
                        if m.user_id != user_id]
        
        # Also remove from indices
        self.vector_index.delete_by_user(user_id)
        
        # Log deletion for compliance
        self.audit_log.record_deletion(user_id, datetime.now())
    
    def set_retention_policy(self, max_age_days: int):
        """Auto-delete memories older than retention period."""
        cutoff = datetime.now() - timedelta(days=max_age_days)
        self.memories = [m for m in self.memories 
                        if m.created_at > cutoff]
```

---

## Best Practices

```
1. Combine multiple signals
   - Age, importance, access frequency
   - No single factor is sufficient

2. Prefer soft over hard initially
   - Deprioritize before deleting
   - Allow recovery of mistakes

3. Consolidate before forgetting
   - Summarize to preserve insights
   - Reduce volume, keep value

4. Respect user preferences
   - Allow explicit deletion
   - Retention policies

5. Monitor forgetting impact
   - Track quality metrics
   - Adjust thresholds
```

---

## Interview Tips

When discussing Forgetting:
1. Explain why forgetting is necessary
2. Describe decay strategies (time, access, importance)
3. Discuss soft vs hard forgetting
4. Mention consolidation approach
5. Know privacy considerations (GDPR)

