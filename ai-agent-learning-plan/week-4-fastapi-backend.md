# Week 4 — FastAPI Backend

> **Connection to final project:** Wrap the RAG engine in a production-quality API that your Next.js frontend (Week 5) and future integrations can call.

**⏱ 10 hours total**

---

## Core Topics

- FastAPI: routers, dependency injection, request/response models with Pydantic
- `async` request handlers — critical for non-blocking LLM calls
- Server-Sent Events (SSE) with `StreamingResponse` — for streaming LLM tokens to the frontend
- File upload handling: `UploadFile`, saving and ingesting PDFs via API endpoint
- Background tasks: `BackgroundTasks` for async document indexing
- CORS setup for Next.js dev server
- Basic error handling: `HTTPException`, global exception handlers

---

## Best Resources (Free)

| Resource | URL | Time |
|----------|-----|------|
| **FastAPI official tutorial (full)** | https://fastapi.tiangolo.com/tutorial/ | 2 hr |
| **FastAPI streaming responses guide** | https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse | 1 hr |
| **"Build a production FastAPI app" — TestDriven.io** | https://testdriven.io/blog/fastapi-crud/ | 1 hr |
| **FastAPI + LangChain streaming example** | https://github.com/langchain-ai/langchain/blob/master/docs/docs/how_to/streaming_llm.ipynb | 1 hr |
| **Pydantic settings management (for env vars)** | https://docs.pydantic.dev/latest/concepts/pydantic_settings/ | 0.5 hr |

---

## Project Structure

```
api/
├── main.py              ← FastAPI app entry point
├── routers/
│   ├── chat.py          ← POST /chat (streaming SSE)
│   ├── documents.py     ← POST /ingest, GET /documents, DELETE /documents/{id}
│   └── health.py        ← GET /health
├── services/
│   └── rag_service.py   ← Wraps week-3 RAG logic
├── models/
│   └── schemas.py       ← Pydantic request/response models
└── config.py            ← pydantic-settings for env vars
```

---

## 🛠 Mini Project: RAG API Server

### `config.py`
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    anthropic_api_key: str
    chroma_persist_dir: str = "./chroma_db"
    langchain_api_key: str = ""
    langchain_tracing_v2: bool = False

    class Config:
        env_file = ".env"

settings = Settings()
```

### `models/schemas.py`
```python
from pydantic import BaseModel
from typing import Optional

class ChatRequest(BaseModel):
    message: str
    history: list[dict] = []

class IngestURLRequest(BaseModel):
    url: str

class DocumentMeta(BaseModel):
    id: str
    filename: str
    chunk_count: int
    indexed_at: str
```

### `routers/chat.py` — Streaming SSE
```python
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from ..models.schemas import ChatRequest
from ..services.rag_service import RAGService
import json

router = APIRouter()
rag = RAGService()

@router.post("/chat")
async def chat(request: ChatRequest):
    async def generate():
        async for chunk in rag.stream_answer(request.message, request.history):
            # SSE format: "data: {json}\n\n"
            data = json.dumps({"token": chunk["token"], "sources": chunk.get("sources", [])})
            yield f"data: {data}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        }
    )
```

### `routers/documents.py` — File Upload
```python
from fastapi import APIRouter, UploadFile, File, BackgroundTasks, HTTPException
from ..services.rag_service import RAGService
import shutil, os
from datetime import datetime

router = APIRouter()
rag = RAGService()

@router.post("/ingest")
async def ingest_file(
    background_tasks: BackgroundTasks,
    file: UploadFile = File(...),
):
    if not file.filename.endswith((".pdf", ".txt")):
        raise HTTPException(status_code=400, detail="Only PDF and TXT files supported")

    # Save temporarily
    tmp_path = f"/tmp/{file.filename}"
    with open(tmp_path, "wb") as f:
        shutil.copyfileobj(file.file, f)

    # Index in background (non-blocking)
    background_tasks.add_task(rag.ingest_file, tmp_path, file.filename)

    return {"status": "indexing", "filename": file.filename, "message": "Document is being indexed"}

@router.get("/documents")
async def list_documents():
    return rag.list_documents()

@router.delete("/documents/{doc_id}")
async def delete_document(doc_id: str):
    deleted = rag.delete_document(doc_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Document not found")
    return {"status": "deleted", "doc_id": doc_id}
```

### `main.py`
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from .routers import chat, documents, health

app = FastAPI(title="AI Learning Agent API", version="1.0.0")

# CORS for Next.js dev + prod
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
        "https://your-app.vercel.app",  # Add your Vercel URL
    ],
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(health.router, tags=["health"])
app.include_router(documents.router, tags=["documents"])
app.include_router(chat.router, tags=["chat"])
```

```bash
# Setup
pip install fastapi uvicorn python-multipart pydantic-settings

# Run
uvicorn api.main:app --reload --port 8000

# Test streaming with curl
curl -N -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What is the N+1 problem?"}'

# Auto-generated docs
open http://localhost:8000/docs
```

---

## Testing the 5 Endpoints

```bash
# Health check
curl http://localhost:8000/health

# Upload PDF
curl -X POST http://localhost:8000/ingest \
  -F "file=@./docs/spring-boot.pdf"

# List documents
curl http://localhost:8000/documents

# Chat (streaming)
curl -N -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Explain transaction propagation"}'

# Delete document
curl -X DELETE http://localhost:8000/documents/spring-boot_0
```

---

## ✅ Success Criteria

- [ ] `POST /chat` streams visible tokens progressively (test with `curl --no-buffer` / `-N`)
- [ ] Upload a new PDF via `/ingest` and query it via `/chat` immediately after
- [ ] FastAPI auto-generated docs at `/docs` correctly documents all 5 endpoints
- [ ] API returns proper 4xx errors (not 500s) for missing files or bad inputs
- [ ] ChromaDB state survives API server restart (documents persist)

---

[← Week 3](./week-3-rag-langchain.md) | [Week 5 →](./week-5-nextjs-frontend.md)

