# Week 3 — RAG Pipeline with LangChain

> **Connection to final project:** This is the heart of your learning agent. By end of week you'll have a working RAG Q&A system — the core of the final project's backend.

**⏱ 10 hours total**

---

## Core Topics

- LangChain architecture: chains, runnables, LCEL (LangChain Expression Language)
- Document loaders: `PyPDFLoader`, `WebBaseLoader`, `DirectoryLoader`
- Text splitters: `RecursiveCharacterTextSplitter` — the most important one to know
- `Chroma` vector store integration in LangChain
- Retrieval chain: `create_retrieval_chain` + `create_stuff_documents_chain`
- LangSmith tracing for debugging RAG pipelines (free tier)
- Advanced retrieval: MMR (Max Marginal Relevance), self-query retriever

---

## Best Resources (Free)

| Resource | URL | Time |
|----------|-----|------|
| **LangChain RAG tutorial (official)** | https://python.langchain.com/docs/tutorials/rag/ | 2 hr |
| **LangChain LCEL conceptual guide** | https://python.langchain.com/docs/concepts/lcel/ | 1 hr |
| **James Briggs — RAG from scratch (YouTube playlist)** | https://www.youtube.com/playlist?list=PLfaoidHpzDYjBUMIIQs5b0F1R-OKP3k4M | 2 hr |
| **LangSmith quickstart (free tracing)** | https://docs.smith.langchain.com/tutorials/Administrators/quick_start | 0.5 hr |
| **LangChain GitHub — RAG from scratch** | https://github.com/langchain-ai/rag-from-scratch | 1 hr |
| **"Advanced RAG techniques" — Pinecone blog** | https://www.pinecone.io/learn/advanced-rag/ | 1 hr |

---

## RAG Pipeline Architecture

```
User Query: "What is N+1 problem in JPA?"
          │
          ▼
   [Embed query]  ←── sentence-transformers / Voyage AI
          │
          ▼
 [ChromaDB vector search]  ─── returns top 5 similar chunks
          │
          ▼
 [Build prompt]
   System: "Answer from context only. Cite sources."
   Context: [chunk1] [chunk2] [chunk3]
   Question: "What is N+1 problem in JPA?"
          │
          ▼
   [Claude API call]
          │
          ▼
   Answer + source citations
```

---

##  Mini Project: Document Q&A System

Build `rag_qa.py` — the learning agent's core engine:

```python
# rag_qa.py
import os
from pathlib import Path
from dotenv import load_dotenv
from langchain_anthropic import ChatAnthropic
from langchain_community.document_loaders import PyPDFLoader, WebBaseLoader, DirectoryLoader
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate

load_dotenv()

# Setup
llm = ChatAnthropic(model="claude-sonnet-4-6", streaming=True)
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)

CHROMA_PATH = "./chroma_db"

def get_vectorstore():
    return Chroma(persist_directory=CHROMA_PATH, embedding_function=embeddings)

def ingest_pdf(pdf_path: str):
    loader = PyPDFLoader(pdf_path)
    docs = loader.load()
    chunks = splitter.split_documents(docs)
    vectorstore = Chroma.from_documents(
        chunks,
        embeddings,
        persist_directory=CHROMA_PATH,
    )
    print(f"✅ Indexed {len(chunks)} chunks from {pdf_path}")
    return vectorstore

def ingest_url(url: str):
    loader = WebBaseLoader(url)
    docs = loader.load()
    chunks = splitter.split_documents(docs)
    vectorstore = get_vectorstore()
    vectorstore.add_documents(chunks)
    print(f"✅ Indexed {len(chunks)} chunks from {url}")

def build_rag_chain():
    vectorstore = get_vectorstore()
    retriever = vectorstore.as_retriever(
        search_type="mmr",       # Max Marginal Relevance — diverse results
        search_kwargs={"k": 5},
    )

    system_prompt = """You are a knowledgeable study assistant.
Answer the question using ONLY the provided context.
If the context doesn't contain enough information, say "I don't have information on that in my documents."
Always cite the source document(s) at the end of your answer.

Context:
{context}"""

    prompt = ChatPromptTemplate.from_messages([
        ("system", system_prompt),
        ("human", "{input}"),
    ])

    question_answer_chain = create_stuff_documents_chain(llm, prompt)
    return create_retrieval_chain(retriever, question_answer_chain)

def ask(question: str):
    chain = build_rag_chain()
    result = chain.invoke({"input": question})

    print(f"\n Question: {question}")
    print(f"\n Answer:\n{result['answer']}")
    print(f"\n Sources used:")
    for doc in result["context"]:
        src = doc.metadata.get("source", "unknown")
        page = doc.metadata.get("page", "")
        print(f"  - {src}" + (f" (page {page})" if page else ""))

# Usage
if __name__ == "__main__":
    import sys
    if len(sys.argv) < 2:
        print("Usage:")
        print("  python rag_qa.py ingest-pdf path/to/file.pdf")
        print("  python rag_qa.py ingest-url https://example.com/article")
        print('  python rag_qa.py ask "Your question here"')
        sys.exit(1)

    cmd = sys.argv[1]
    if cmd == "ingest-pdf":
        ingest_pdf(sys.argv[2])
    elif cmd == "ingest-url":
        ingest_url(sys.argv[2])
    elif cmd == "ask":
        ask(" ".join(sys.argv[2:]))
```

```bash
# Setup
pip install langchain langchain-anthropic langchain-community \
            chromadb sentence-transformers pypdf python-dotenv \
            langsmith bs4

# Enable LangSmith tracing (free account at smith.langchain.com)
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY=your-langsmith-key

# Ingest documents
python rag_qa.py ingest-pdf ./docs/spring-boot-reference.pdf
python rag_qa.py ingest-url https://docs.spring.io/spring-boot/docs/current/reference/html/

# Ask questions
python rag_qa.py ask "What is the N+1 problem in JPA and how to fix it?"
python rag_qa.py ask "How does Spring Boot auto-configuration work?"
python rag_qa.py ask "What is the capital of France?"  # Should trigger "I don't know"
```

---

## LangSmith Trace — What to Look For

1. Go to `smith.langchain.com` → your project
2. Click on a trace → you'll see:
   - Input query
   - Retrieved documents (with similarity scores)
   - Final prompt sent to Claude
   - Claude's response + token count
3. If answers are bad → check what was retrieved. Bad retrieval = bad answers.

---

## Advanced: MMR vs Similarity Search

```python
# Basic similarity — top 5 most similar (may be redundant)
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# MMR — diverse top 5 (avoids 5 near-identical chunks)
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 5, "fetch_k": 20, "lambda_mult": 0.5}
)
# lambda_mult: 0 = max diversity, 1 = max relevance
```

---

## ✅ Success Criteria

- [ ] Can load a 50-page PDF and answer specific questions about its content accurately
- [ ] Source citations in answers reference the correct documents
- [ ] "I don't know" responses trigger when query is outside ingested documents
- [ ] LangSmith trace shows the full retrieval → prompt → LLM chain visually
- [ ] Can add a new document and query it without re-indexing existing documents

---

[← Week 2](./week-2-embeddings-vectordb.md) | [Week 4 →](./week-4-fastapi-backend.md)
