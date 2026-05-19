#  6-Week AI Learning Agent: Build Plan

> **Goal:** Build and deploy a personal AI learning agent that can ingest documents (PDFs, URLs, notes), store them in a vector database, and answer questions using RAG + an LLM (Claude API).
>
> **Stack:** Python · LangChain · ChromaDB · FastAPI · Next.js · Anthropic Claude API
>
> **Pace:** 10 hrs/week · 60 hrs total · Hands-on first

---

##  Summary Table

| Week | Theme | Key Deliverable | Hours |
|------|-------|----------------|-------|
| [Week 1](./week-1-python-llm-basics.md) | Python for AI & LLM Basics | CLI chatbot using Claude API | 10 |
| [Week 2](./week-2-embeddings-vectordb.md) | Embeddings & Vector Databases | Local semantic search tool | 10 |
| [Week 3](./week-3-rag-langchain.md) | RAG Pipeline with LangChain | Document Q&A script (PDF + URLs) | 10 |
| [Week 4](./week-4-fastapi-backend.md) | FastAPI Backend | REST API serving the RAG pipeline | 10 |
| [Week 5](./week-5-nextjs-frontend.md) | Next.js Frontend + Streaming | Full-stack chat UI with file uploads | 10 |
| [Week 6](./week-6-production-deploy.md) | Production Deployment | Live, deployed AI learning agent | 10 |

---

##  Final Project Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Next.js Frontend (Vercel)              │
│   /chat (streaming UI)   /documents (upload + manage)    │
└────────────────────┬────────────────────────────────────┘
                     │ HTTP / SSE
┌────────────────────▼────────────────────────────────────┐
│                  FastAPI Backend (Railway)                │
│  POST /chat  POST /ingest  GET /documents  DELETE /docs  │
│              ┌─────────────────────┐                     │
│              │  LangChain RAG Chain │                     │
│              │  (LCEL pipeline)     │                     │
│              └──────┬──────────────┘                     │
│         ┌───────────┴──────────────┐                     │
│         ▼                          ▼                      │
│  ChromaDB / Qdrant           Claude API                  │
│  (vector store)              (Anthropic)                  │
└─────────────────────────────────────────────────────────┘
```

---

##  Pro Tips & Pitfalls to Avoid

| Pitfall | Fix |
|---------|-----|
| ChromaDB losing data on Railway restart | Mount a persistent volume at `/app/chroma_db` |
| LLM responses slow to start streaming | Use `StreamingResponse` with a generator, not `await` blocking call |
| Chunk size too large = poor retrieval | Start at 1000 chars / 200 overlap; tune based on your doc types |
| Paying too much for embeddings | Use `sentence-transformers` locally (free) — only call API for Claude completions |
| CORS errors when connecting frontend | Set `allow_origins=["http://localhost:3000", "https://your-app.vercel.app"]` |

---

*Total: 60 hours · 6 weeks · 1 deployed personal AI learning agent*
