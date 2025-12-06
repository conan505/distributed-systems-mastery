# Working Memory Buffers

## What It Is

**Working Memory Buffers** are short-term memory components in LLM systems that maintain active context during interactions, managing what information is immediately accessible to the model for processing.

---

## The Analogy 📋

Think of a **whiteboard during a meeting**:
- Limited space (context window)
- Write current discussion points
- Erase old content to make room
- Most recent and important info visible
- Reference it while speaking

---

## Why It Exists

### The Context Window Problem
```
LLM context window: Fixed size (4K-128K tokens)

Long conversation:
Message 1: 500 tokens
Message 2: 300 tokens
Message 3: 800 tokens
...
Message 50: 400 tokens
Total: 25,000 tokens

If window = 8K tokens, what do we keep?
Working memory buffers manage this!
```

---

## Buffer Types

### 1. Conversation Buffer
```python
from collections import deque

class ConversationBuffer:
    """Simple sliding window of messages."""
    
    def __init__(self, max_tokens: int = 4000):
        self.messages = deque()
        self.max_tokens = max_tokens
        self.current_tokens = 0
    
    def add_message(self, role: str, content: str):
        tokens = count_tokens(content)
        
        # Remove old messages if needed
        while self.current_tokens + tokens > self.max_tokens:
            if not self.messages:
                break
            old_msg = self.messages.popleft()
            self.current_tokens -= count_tokens(old_msg["content"])
        
        # Add new message
        self.messages.append({"role": role, "content": content})
        self.current_tokens += tokens
    
    def get_messages(self) -> list:
        return list(self.messages)
```

### 2. Summary Buffer
```python
class ConversationSummaryBuffer:
    """Keep recent messages + summary of older ones."""
    
    def __init__(self, llm, max_tokens: int = 4000, summary_threshold: int = 2000):
        self.llm = llm
        self.max_tokens = max_tokens
        self.summary_threshold = summary_threshold
        self.summary = ""
        self.recent_messages = []
    
    def add_message(self, role: str, content: str):
        self.recent_messages.append({"role": role, "content": content})
        
        # Check if we need to summarize
        recent_tokens = sum(count_tokens(m["content"]) for m in self.recent_messages)
        
        if recent_tokens > self.summary_threshold:
            self._consolidate()
    
    def _consolidate(self):
        """Summarize older messages."""
        # Keep last few messages
        to_summarize = self.recent_messages[:-3]
        self.recent_messages = self.recent_messages[-3:]
        
        # Generate summary
        messages_text = "\n".join(
            f"{m['role']}: {m['content']}" for m in to_summarize
        )
        
        new_summary = self.llm.generate(
            f"Summarize this conversation:\n{messages_text}"
        )
        
        # Append to existing summary
        self.summary = f"{self.summary}\n{new_summary}".strip()
    
    def get_context(self) -> str:
        context = ""
        if self.summary:
            context += f"Previous conversation summary:\n{self.summary}\n\n"
        context += "Recent messages:\n"
        for m in self.recent_messages:
            context += f"{m['role']}: {m['content']}\n"
        return context
```

### 3. Entity Buffer
```python
class EntityMemory:
    """Track and update entity information."""
    
    def __init__(self, llm):
        self.llm = llm
        self.entities = {}  # entity_name -> description
    
    def extract_and_update(self, message: str):
        """Extract entities from message and update memory."""
        prompt = f"""
        Extract entities and their information from this message:
        "{message}"
        
        Current known entities:
        {self.entities}
        
        Return updated entity information as JSON.
        """
        
        result = self.llm.generate(prompt)
        new_entities = parse_json(result)
        
        # Merge with existing
        for entity, info in new_entities.items():
            if entity in self.entities:
                self.entities[entity] = f"{self.entities[entity]} {info}"
            else:
                self.entities[entity] = info
    
    def get_context(self) -> str:
        if not self.entities:
            return ""
        
        lines = ["Known entities:"]
        for entity, info in self.entities.items():
            lines.append(f"- {entity}: {info}")
        return "\n".join(lines)
```

---

## LangChain Memory Types

```python
from langchain.memory import (
    ConversationBufferMemory,
    ConversationBufferWindowMemory,
    ConversationSummaryMemory,
    ConversationSummaryBufferMemory,
    ConversationKGMemory,
)

# Simple buffer
buffer_memory = ConversationBufferMemory()

# Last k messages
window_memory = ConversationBufferWindowMemory(k=5)

# Summarize old messages
summary_memory = ConversationSummaryMemory(llm=llm)

# Hybrid: recent + summary
hybrid_memory = ConversationSummaryBufferMemory(
    llm=llm,
    max_token_limit=1000
)

# Knowledge graph
kg_memory = ConversationKGMemory(llm=llm)
```

---

## Buffer Management Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│                 Buffer Management Strategies                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. FIFO (First In, First Out)                                 │
│      Remove oldest messages first                               │
│      Simple but may lose important early context               │
│                                                                  │
│   2. Importance-Based                                           │
│      Score messages by relevance/importance                     │
│      Keep high-scoring messages longer                          │
│                                                                  │
│   3. Summarization                                              │
│      Compress old messages into summaries                       │
│      Preserves information, loses detail                        │
│                                                                  │
│   4. Hybrid (Most Common)                                       │
│      Keep recent messages verbatim                              │
│      Summarize older messages                                   │
│      Store key entities separately                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Token Budget Allocation

```python
def allocate_buffer(total_budget: int) -> dict:
    """Allocate token budget across buffer sections."""
    return {
        "system_prompt": int(total_budget * 0.10),  # 10%
        "summary": int(total_budget * 0.15),         # 15%
        "entities": int(total_budget * 0.10),        # 10%
        "recent_messages": int(total_budget * 0.50), # 50%
        "retrieved_context": int(total_budget * 0.15), # 15%
    }

# For 8K context:
# system: 800, summary: 1200, entities: 800
# recent: 4000, retrieved: 1200
```

---

## Best Practices

```
1. Prioritize recent context
   - Most relevant for current turn
   - Keep verbatim when possible

2. Summarize strategically
   - Trigger based on token count
   - Preserve key decisions/facts

3. Track important entities
   - Names, preferences, states
   - Update continuously

4. Handle multi-turn coherence
   - Maintain conversation flow
   - Reference earlier context

5. Graceful degradation
   - Inform when context is lost
   - Offer to recap if needed
```

---

## Performance Considerations

```
╔════════════════════════════════════════════════╗
║  Strategy       │  Memory │  Latency │ Quality ║
╠════════════════════════════════════════════════╣
║  Full buffer    │  High   │  High    │ Best    ║
║  Window only    │  Low    │  Low     │ OK      ║
║  Summary only   │  Low    │  Medium  │ Good    ║
║  Hybrid         │  Medium │  Medium  │ Best    ║
╚════════════════════════════════════════════════╝
```

---

## Interview Tips

When discussing Working Memory Buffers:
1. Explain the context window limitation
2. Describe different buffer strategies
3. Discuss summarization trade-offs
4. Know LangChain memory types
5. Mention token budget allocation

