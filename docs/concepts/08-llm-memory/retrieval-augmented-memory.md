# Retrieval-Augmented Memory (RAG)

## What It Is

**Retrieval-Augmented Generation (RAG)** is a technique that enhances LLM responses by retrieving relevant information from external knowledge bases before generating, effectively expanding the model's "memory" beyond its training data.

---

## The Analogy 📖

Think of an **open-book exam**:
- You have knowledge (model weights)
- But you can also look up information (retrieval)
- Find relevant pages before answering
- Combine your understanding with reference material
- Better answers than memory alone

---

## Why It Exists

### LLM Limitations
```
Problems with parametric memory (weights only):
1. Knowledge cutoff (training date)
2. Hallucinations (making things up)
3. No private/proprietary data
4. Can't update without retraining

RAG adds:
1. Current information
2. Grounded, verifiable answers
3. Custom knowledge bases
4. Easy updates (add/remove docs)
```

---

## RAG Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      RAG Pipeline                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Query: "What's our refund policy?"                            │
│            │                                                     │
│            ▼                                                     │
│   ┌─────────────────┐                                           │
│   │ Embedding Model │ → Query embedding                         │
│   └─────────────────┘                                           │
│            │                                                     │
│            ▼                                                     │
│   ┌─────────────────┐     ┌──────────────────┐                 │
│   │  Vector Search  │ ←── │ Document Vectors │                 │
│   └─────────────────┘     └──────────────────┘                 │
│            │                                                     │
│            ▼                                                     │
│   Retrieved: [refund_policy.md, returns_faq.md]                │
│            │                                                     │
│            ▼                                                     │
│   ┌─────────────────┐                                           │
│   │      LLM        │ ← Context + Query                         │
│   └─────────────────┘                                           │
│            │                                                     │
│            ▼                                                     │
│   Answer: "Our refund policy allows returns within 30 days..."  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation

### Basic RAG
```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.chat_models import ChatOpenAI
from langchain.chains import RetrievalQA

# 1. Create vector store from documents
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(documents, embeddings)

# 2. Create retriever
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 3}
)

# 3. Create RAG chain
llm = ChatOpenAI(model="gpt-4")
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",  # Stuff all docs into prompt
    retriever=retriever,
    return_source_documents=True
)

# 4. Query
result = qa_chain({"query": "What's the refund policy?"})
print(result["result"])
print(result["source_documents"])
```

### Advanced RAG with Reranking
```python
from sentence_transformers import CrossEncoder

class AdvancedRAG:
    def __init__(self, vectorstore, llm):
        self.vectorstore = vectorstore
        self.llm = llm
        self.reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
    
    def query(self, question: str, k: int = 3) -> str:
        # 1. Initial retrieval (get more than needed)
        initial_docs = self.vectorstore.similarity_search(question, k=k*3)
        
        # 2. Rerank with cross-encoder
        pairs = [(question, doc.page_content) for doc in initial_docs]
        scores = self.reranker.predict(pairs)
        
        # 3. Take top k after reranking
        ranked = sorted(zip(initial_docs, scores), key=lambda x: x[1], reverse=True)
        top_docs = [doc for doc, _ in ranked[:k]]
        
        # 4. Generate answer
        context = "\n\n".join([doc.page_content for doc in top_docs])
        prompt = f"""Answer based on the following context:

{context}

Question: {question}
Answer:"""
        
        return self.llm.generate(prompt)
```

---

## RAG Patterns

### 1. Naive RAG
```
Retrieve → Stuff into prompt → Generate
Simple but limited by context window
```

### 2. Sentence Window
```
Index sentences, but retrieve surrounding context
Query matches sentence, but LLM sees paragraph
```

### 3. Hierarchical
```
Two-level retrieval:
1. Find relevant documents
2. Find relevant chunks within documents
```

### 4. Hypothetical Document Embeddings (HyDE)
```python
def hyde_retrieval(question: str):
    # Generate hypothetical answer
    hypothetical = llm.generate(
        f"Write a passage that answers: {question}"
    )
    
    # Use hypothetical answer for retrieval
    docs = vectorstore.similarity_search(hypothetical)
    return docs
```

### 5. Self-RAG
```
LLM decides when to retrieve:
1. Generate initial response
2. Self-critique: "Do I need more info?"
3. If yes, retrieve and regenerate
4. Repeat until confident
```

---

## Chunking Strategies

```
Document: 10,000 tokens
Chunk options:

1. Fixed size (simple)
   └── 512 tokens per chunk, overlap 50

2. Semantic (better)
   └── Split at paragraph/section boundaries

3. Recursive (smart)
   └── Split by heading > paragraph > sentence

4. Agentic (advanced)
   └── LLM decides where to split
```

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", ". ", " ", ""]
)

chunks = splitter.split_documents(documents)
```

---

## Evaluation Metrics

```
RAG-specific metrics:

1. Retrieval Quality
   - Recall@k: Were relevant docs retrieved?
   - MRR: Was top result relevant?

2. Generation Quality
   - Faithfulness: Is answer grounded in context?
   - Relevance: Does it answer the question?
   - Coherence: Is it well-formed?

3. End-to-End
   - Answer correctness
   - Hallucination rate
```

---

## Production Considerations

```
1. Embedding model selection
   - Domain-specific vs general
   - Speed vs quality trade-off

2. Indexing strategy
   - HNSW for speed
   - IVF+PQ for scale

3. Metadata filtering
   - Date range
   - Document type
   - Access control

4. Caching
   - Cache embeddings
   - Cache frequent queries

5. Monitoring
   - Track retrieval relevance
   - Log queries with no good matches
```

---

## Best Practices

```
1. Chunk size matters
   - Too small: loses context
   - Too big: retrieves noise
   - Typical: 256-512 tokens

2. Overlap chunks
   - Preserves context at boundaries
   - 10-20% overlap typical

3. Rerank for precision
   - Initial retrieval: high recall
   - Reranking: high precision

4. Include metadata
   - Source, date, section
   - Filter before semantic search

5. Evaluate continuously
   - Track answer quality
   - Identify retrieval failures
```

---

## Interview Tips

When discussing RAG:
1. Explain why RAG beats pure LLM
2. Describe the pipeline (embed → retrieve → generate)
3. Discuss chunking strategies
4. Know reranking and its benefits
5. Mention evaluation metrics

