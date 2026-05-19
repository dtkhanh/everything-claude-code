# Week 2 — Embeddings & Vector Databases

> **Connection to final project:** Embeddings + vector search are the "R" in RAG. Every document ingested by your agent will be chunked, embedded, and stored — you'll build that machinery this week.

**⏱ 10 hours total**

---

## Core Topics

- What embeddings are: text → high-dimensional float vectors, semantic similarity
- `sentence-transformers` library for local embeddings (free, no API cost)
- OpenAI `text-embedding-3-small` and Anthropic's embedding approach (via Voyage AI)
- ChromaDB: collections, `add()`, `query()`, `update()`, `delete()` — persistent local storage
- Distance metrics: cosine similarity vs L2 — practical difference for text search
- Chunking strategies: fixed-size, sentence-based, token-based — why chunk size matters

---

## Best Resources (Free)

| Resource | URL | Time |
|----------|-----|------|
| **ChromaDB official docs** | https://docs.trychroma.com/getting-started | 1.5 hr |
| **"Embeddings explained" — Jay Alammar's blog** | https://jalammar.github.io/illustrated-word2vec/ | 1 hr |
| **Greg Kamradt — 5 levels of text splitting (YouTube)** | https://www.youtube.com/watch?v=8OJC21T2SL4 | 1 hr |
| **sentence-transformers docs + pretrained models** | https://www.sbert.net/docs/pretrained_models.html | 1 hr |
| **ChromaDB GitHub (cookbook examples)** | https://github.com/chroma-core/chroma/tree/main/examples | 1 hr |
| **Voyage AI embeddings for Claude stack** | https://docs.voyageai.com/docs/embeddings | 0.5 hr |

---

## Mental Model: Why Embeddings Work

```
"Java concurrency"   → [0.82, -0.31, 0.14, ...]  ┐
"Thread safety Java" → [0.79, -0.28, 0.19, ...]  ─┤ CLOSE in vector space
"synchronized block" → [0.76, -0.25, 0.22, ...]  ┘

"Python async"       → [0.12, 0.65, -0.44, ...]  ─── FAR from Java vectors

Query: "how to handle concurrent access in Java?"
→ Embed query → find nearest vectors → retrieve those chunks → send to LLM
```

---

##  Mini Project: Local Semantic Search Tool

Build `search.py` that:

1. Takes a folder of `.txt` files (your own notes, blog posts, articles) as input
2. Chunks each file into 512-token overlapping chunks (100-token overlap)
3. Embeds each chunk using `sentence-transformers` (`all-MiniLM-L6-v2` model — free, fast)
4. Stores all chunks + metadata (filename, chunk index) in a persistent **ChromaDB** collection
5. Accepts a search query from CLI and returns the **top 5 most semantically similar** chunks with scores
6. Compares keyword search (`grep`) vs semantic search on 3 test queries

```python
# search.py
import argparse
import os
import chromadb
from sentence_transformers import SentenceTransformer
from pathlib import Path

# Initialize
model = SentenceTransformer("all-MiniLM-L6-v2")  # Free, ~80MB download once
client = chromadb.PersistentClient(path="./chroma_db")  # Persists to disk
collection = client.get_or_create_collection("my_notes")

def chunk_text(text: str, chunk_size: int = 512, overlap: int = 100) -> list[str]:
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        if chunk:
            chunks.append(chunk)
    return chunks

def index_folder(folder: str):
    docs_folder = Path(folder)
    total_chunks = 0
    for file in docs_folder.glob("*.txt"):
        text = file.read_text(encoding="utf-8")
        chunks = chunk_text(text)
        embeddings = model.encode(chunks).tolist()

        collection.add(
            documents=chunks,
            embeddings=embeddings,
            ids=[f"{file.stem}_{i}" for i in range(len(chunks))],
            metadatas=[{"filename": file.name, "chunk_index": i} for i in range(len(chunks))],
        )
        total_chunks += len(chunks)
        print(f"  ✅ {file.name}: {len(chunks)} chunks indexed")

    print(f"\n Total: {total_chunks} chunks stored in ChromaDB")

def search(query: str, top_k: int = 5):
    query_embedding = model.encode([query]).tolist()
    results = collection.query(
        query_embeddings=query_embedding,
        n_results=top_k,
        include=["documents", "metadatas", "distances"],
    )
    print(f"\n Top {top_k} results for: '{query}'\n")
    for i, (doc, meta, dist) in enumerate(zip(
        results["documents"][0],
        results["metadatas"][0],
        results["distances"][0],
    )):
        score = 1 - dist  # cosine similarity
        print(f"[{i+1}] Score: {score:.3f} | {meta['filename']} (chunk {meta['chunk_index']})")
        print(f"    {doc[:200]}...\n")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--index", help="Folder to index")
    parser.add_argument("--query", help="Search query")
    args = parser.parse_args()

    if args.index:
        index_folder(args.index)
    if args.query:
        search(args.query)
```

```bash
# Setup
pip install chromadb sentence-transformers

# Index your notes folder
python search.py --index ./notes/

# Search
python search.py --query "how does Java garbage collection work"
python search.py --query "Spring Boot auto configuration internals"

# Compare: grep (keyword) vs semantic
grep -r "garbage collection" ./notes/  # misses "GC tuning", "JVM heap"
python search.py --query "garbage collection"  # finds "GC tuning", "JVM heap", "memory management"
```

---

## Chunk Size Experiments

Run these and compare quality:

| Chunk Size | Overlap | Good For |
|-----------|---------|---------|
| 256 chars | 50 | Short Q&A, precise lookup |
| 1000 chars | 200 | ✅ General purpose, best default |
| 2000 chars | 400 | Long-form context, document summaries |

---

## ✅ Success Criteria

- [ ] Can explain why `"king - man + woman = queen"` works with embeddings
- [ ] ChromaDB collection persists between script runs (data survives restart)
- [ ] Semantic search returns meaningfully better results than `grep` for conceptual queries
- [ ] Can articulate why chunk size 256 vs 1024 changes retrieval quality
- [ ] Script handles 50+ documents without crashing

---

[← Week 1](./week-1-python-llm-basics.md) | [Week 3 →](./week-3-rag-langchain.md)
