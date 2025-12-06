# Vector Databases

## What It Is

A **Vector Database** is a specialized database designed to store, index, and query high-dimensional vectors (embeddings), enabling fast similarity search for AI applications like RAG, recommendations, and semantic search.

---

## The Analogy 📚

Think of a **smart librarian**:
- Books stored by meaning, not just title
- Ask for "stories about friendship" → finds relevant books
- Understands similarity, not just exact matches
- Retrieves based on semantic closeness

---

## Why It Exists

### The Embedding Revolution
```
Traditional search: Keyword matching
"How to train a dog" matches "dog training tips"
Misses: "puppy obedience lessons"

Vector search: Semantic similarity
"How to train a dog" → embedding → [0.2, 0.8, -0.3, ...]
"puppy obedience lessons" → embedding → [0.21, 0.79, -0.31, ...]

Cosine similarity: 0.98 → Match!
```

### Key Use Cases
```
1. Retrieval-Augmented Generation (RAG)
   - Query: "What's our refund policy?"
   - Search company docs by meaning
   - Feed relevant docs to LLM

2. Semantic Search
   - Beyond keyword matching
   - Find conceptually similar content

3. Recommendations
   - Find similar products/content
   - User preference matching
```

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                   Vector Database Pipeline                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Indexing:                                                     │
│   Document → Embedding Model → Vector → Index                   │
│   "The cat sat" → [0.1, 0.8, 0.3, ...] → HNSW/IVF              │
│                                                                  │
│   Querying:                                                     │
│   Query → Embedding Model → Vector → ANN Search → Top-K        │
│   "feline resting" → [0.15, 0.75, 0.35] → Similar vectors      │
│                                                                  │
│   Return documents with most similar embeddings                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Indexing Algorithms

### 1. HNSW (Hierarchical Navigable Small World)
```
Multi-layer graph structure:
Layer 2:  A ──────────── D
          │              │
Layer 1:  A ─── C ────── D
          │    │         │
Layer 0:  A─B──C─E───F───D

Search: Start at top layer, descend to find nearest
Time: O(log n)
Space: O(n × M) where M = edges per node
Best for: High recall, fast query
```

### 2. IVF (Inverted File Index)
```
Cluster vectors, search only relevant clusters:

Clusters: [C1: vectors 1-1000] [C2: vectors 1001-2000] ...

Query:
1. Find nearest cluster centroids
2. Search only those clusters

Faster but lower recall than HNSW
Good for: Large datasets, acceptable recall trade-off
```

### 3. Product Quantization (PQ)
```
Compress vectors for memory efficiency:

Original: 768 floats × 4 bytes = 3KB per vector
PQ: 768 → 96 codes × 1 byte = 96 bytes

10x memory reduction
Trade-off: Some accuracy loss
```

---

## Popular Vector Databases

| Database | Type | Best For |
|----------|------|----------|
| **Pinecone** | Managed | Production, ease of use |
| **Weaviate** | Open-source | Hybrid search, GraphQL |
| **Milvus** | Open-source | Large scale, on-prem |
| **Qdrant** | Open-source | Rust performance |
| **Chroma** | Open-source | Prototyping, LangChain |
| **pgvector** | Extension | Postgres users |
| **FAISS** | Library | Research, custom solutions |

---

## Implementation Example

### Using Chroma
```python
import chromadb
from chromadb.utils import embedding_functions

# Initialize client
client = chromadb.Client()

# Create collection with embedding function
ef = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="all-MiniLM-L6-v2"
)
collection = client.create_collection(
    name="documents",
    embedding_function=ef
)

# Add documents
collection.add(
    documents=["The cat sat on the mat", "Dogs love to play fetch"],
    metadatas=[{"source": "doc1"}, {"source": "doc2"}],
    ids=["id1", "id2"]
)

# Query
results = collection.query(
    query_texts=["feline behavior"],
    n_results=2
)
```

### Using FAISS
```python
import faiss
import numpy as np

# Create index
dimension = 768
index = faiss.IndexFlatL2(dimension)  # Exact search

# For large scale, use IVF
nlist = 100  # Number of clusters
quantizer = faiss.IndexFlatL2(dimension)
index = faiss.IndexIVFFlat(quantizer, dimension, nlist)
index.train(training_vectors)

# Add vectors
index.add(vectors)

# Search
k = 5  # Top-k results
distances, indices = index.search(query_vector, k)
```

---

## RAG Integration

```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.chains import RetrievalQA
from langchain.llms import OpenAI

# Create vector store
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=OpenAIEmbeddings()
)

# Create RAG chain
qa_chain = RetrievalQA.from_chain_type(
    llm=OpenAI(),
    retriever=vectorstore.as_retriever(
        search_kwargs={"k": 3}
    )
)

# Query
answer = qa_chain.run("What is our vacation policy?")
```

---

## Best Practices

```
1. Choose right embedding model
   - Domain-specific if available
   - Balance quality vs speed

2. Chunk documents appropriately
   - 256-512 tokens typical
   - Overlap for context

3. Add metadata for filtering
   - Source, date, category
   - Hybrid search (vector + filter)

4. Monitor and tune
   - Measure recall
   - Adjust index parameters

5. Consider hybrid search
   - Combine with keyword (BM25)
   - Rerank results
```

---

## Performance Comparison

```
╔════════════════════════════════════════════════╗
║  Index Type     │  Query Time │  Recall       ║
╠════════════════════════════════════════════════╣
║  Flat (exact)   │  O(n)       │  100%         ║
║  IVF            │  O(√n)      │  95-99%       ║
║  HNSW           │  O(log n)   │  99%+         ║
║  PQ + IVF       │  O(√n)      │  90-95%       ║
╚════════════════════════════════════════════════╝
```

---

## Interview Tips

When discussing Vector Databases:
1. Explain embeddings and similarity search
2. Describe indexing algorithms (HNSW, IVF)
3. Discuss RAG use case
4. Compare popular solutions
5. Know trade-offs (recall vs speed vs memory)

