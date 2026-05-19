# Week 6 — Production Deployment & Polish

> **Connection to final project:** The difference between a demo and a real tool you'll actually use. This week makes it reliable, shareable, and maintainable.

**⏱ 10 hours total**

---

## Core Topics

- Docker: `Dockerfile` for FastAPI, `docker-compose.yml` for full-stack local dev
- Deploying FastAPI to **Railway** (free tier, easiest)
- Deploying Next.js to **Vercel** (free tier, optimal)
- Persisting ChromaDB via Railway volumes OR migrating to **Qdrant Cloud** (free tier)
- Environment variable management in production (Railway/Render secrets)
- Basic observability: LangSmith traces + Railway logs
- Rate limiting with `slowapi` — protect your Claude API budget
- `README.md` — document the setup for future re-deploys

---

## Best Resources (Free)

| Resource | URL | Time |
|----------|-----|------|
| **Railway docs — Python deployments** | https://docs.railway.app/guides/python | 1 hr |
| **Vercel deployment docs (Next.js)** | https://vercel.com/docs/frameworks/nextjs | 0.5 hr |
| **Qdrant Cloud quickstart (free tier)** | https://qdrant.tech/documentation/cloud/quickstart-cloud/ | 1 hr |
| **Docker official "Getting Started" (Part 1–3)** | https://docs.docker.com/get-started/ | 1.5 hr |
| **`slowapi` rate limiting for FastAPI** | https://github.com/laurentS/slowapi | 0.5 hr |
| **LangSmith production monitoring guide** | https://docs.smith.langchain.com/how_to_guides/monitoring | 1 hr |

---

## Step 1: Dockerize the Backend

### `Dockerfile` (FastAPI)
```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app
COPY . .

# ChromaDB persistence directory
RUN mkdir -p /app/chroma_db

EXPOSE 8000

CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### `docker-compose.yml` (local full-stack dev)
```yaml
version: "3.9"

services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    volumes:
      - chroma_data:/app/chroma_db  # Persist ChromaDB
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - LANGCHAIN_API_KEY=${LANGCHAIN_API_KEY}
      - LANGCHAIN_TRACING_V2=true
    env_file:
      - .env

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8000
    depends_on:
      - backend

volumes:
  chroma_data:   # Named volume — persists between container restarts
```

```bash
# Test locally with Docker
docker-compose up --build

# Verify data persists
docker-compose down
docker-compose up  # ChromaDB data should still be there
```

---

## Step 2: Deploy Backend to Railway

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login and deploy
railway login
railway init
railway up

# Set environment variables (Railway dashboard or CLI)
railway variables set ANTHROPIC_API_KEY=your-key
railway variables set LANGCHAIN_API_KEY=your-langsmith-key
railway variables set LANGCHAIN_TRACING_V2=true
railway variables set CHROMA_PERSIST_DIR=/app/chroma_db
```

**Critical: Attach a persistent volume in Railway dashboard:**
1. Go to your service → Storage → Add Volume
2. Mount path: `/app/chroma_db`
3. Without this, ChromaDB resets on every deploy/restart

### `railway.json` (deploy config)
```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "DOCKERFILE",
    "dockerfilePath": "Dockerfile"
  },
  "deploy": {
    "healthcheckPath": "/health",
    "healthcheckTimeout": 30,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 3
  }
}
```

---

## Step 3: Deploy Frontend to Vercel

```bash
# Install Vercel CLI
npm install -g vercel

cd frontend
vercel

# Set production env var
vercel env add NEXT_PUBLIC_API_URL production
# Enter your Railway URL: https://your-app.railway.app
```

---

## Step 4: Rate Limiting (Protect Claude API Budget)

```python
# pip install slowapi

from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import Request

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# In routers/chat.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@router.post("/chat")
@limiter.limit("10/minute")  # 10 requests per minute per IP
async def chat(request: Request, body: ChatRequest):
    ...
```

```bash
# Test rate limiting
for i in {1..15}; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST https://your-app.railway.app/chat \
    -H "Content-Type: application/json" \
    -d '{"message": "test"}'
done
# Should see 200×10, then 429×5
```

---

## Optional Step 5: Migrate to Qdrant Cloud (Recommended)

Qdrant Cloud free tier is more reliable than Railway volumes for vector persistence.

```python
# pip install qdrant-client langchain-qdrant

from langchain_qdrant import Qdrant
from qdrant_client import QdrantClient

client = QdrantClient(
    url=os.getenv("QDRANT_URL"),          # From Qdrant Cloud dashboard
    api_key=os.getenv("QDRANT_API_KEY"),  # From Qdrant Cloud dashboard
)

vectorstore = Qdrant(
    client=client,
    collection_name="learning_agent",
    embeddings=embeddings,
)
```

Just replace ChromaDB with this in `rag_service.py` — ~15 line change. Add `QDRANT_URL` and `QDRANT_API_KEY` to Railway env vars.

---

## Production Smoke Test Checklist

```bash
PROD_URL="https://your-app.railway.app"
FRONT_URL="https://your-app.vercel.app"

# 1. Health check
curl $PROD_URL/health
# Expected: {"status": "ok"}

# 2. Upload a document
curl -X POST $PROD_URL/ingest \
  -F "file=@./test-doc.pdf"
# Expected: {"status": "indexing", ...}

# 3. Wait 5 seconds, then list documents
sleep 5 && curl $PROD_URL/documents
# Expected: doc appears in list

# 4. Query via streaming
curl -N -X POST $PROD_URL/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Summarize the document"}'
# Expected: streaming tokens appear

# 5. Restart backend service in Railway dashboard
# Then re-run steps 3 + 4
# Expected: documents still there, queries still work

# 6. Rate limit test
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "$i: %{http_code}\n" \
    -X POST $PROD_URL/chat \
    -H "Content-Type: application/json" \
    -d '{"message": "test"}'
done
# Expected: 1-10 return 200, 11-15 return 429
```

---

## Final `README.md` Template

```markdown
# AI Learning Agent

Personal document Q&A agent powered by RAG + Claude API.

## Live Demo
- Frontend: https://your-app.vercel.app
- Backend API: https://your-app.railway.app/docs

## Architecture
[paste ASCII diagram from main README]

## Local Development

### Prerequisites
- Python 3.12+
- Node.js 20+
- Docker (optional)

### Setup
```bash
# Backend
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # Add ANTHROPIC_API_KEY
uvicorn api.main:app --reload

# Frontend
cd frontend
npm install
echo "NEXT_PUBLIC_API_URL=http://localhost:8000" > .env.local
npm run dev
```

### Environment Variables
| Variable | Required | Description |
|----------|----------|-------------|
| `ANTHROPIC_API_KEY` | ✅ | Anthropic API key |
| `LANGCHAIN_API_KEY` | optional | LangSmith tracing |
| `QDRANT_URL` | optional | Qdrant Cloud URL (if not using ChromaDB) |
| `QDRANT_API_KEY` | optional | Qdrant Cloud API key |

## Deployment
1. Deploy backend to Railway (attach `/app/chroma_db` volume)
2. Deploy frontend to Vercel (set `NEXT_PUBLIC_API_URL`)
3. Set all env vars in Railway dashboard
```

---

## ✅ Success Criteria

- [ ] App is live at a public URL — someone else can use it without your help
- [ ] Uploading a document and querying it works end-to-end on the deployed version
- [ ] Restarting the Railway service does **not** lose previously indexed documents
- [ ] LangSmith dashboard shows live traces from production queries
- [ ] Rate limiter returns `429` after 10 rapid requests
- [ ] `README.md` complete enough to redeploy from scratch in under 30 minutes

---

## 🎉 You Built It

```
Week 1: CLI chatbot streaming tokens ✅
Week 2: Semantic search over your notes ✅
Week 3: RAG Q&A with source citations ✅
Week 4: FastAPI backend with streaming SSE ✅
Week 5: Next.js chat UI with file uploads ✅
Week 6: Production deployed, persistent, rate-limited ✅
```

**Next steps after this:**
- Add user auth (NextAuth.js) to make it multi-user
- Evaluate RAG quality with `ragas` library
- Add conversation memory with LangChain's `ConversationBufferMemory`
- Build a second agent that generates flashcards from your documents

---

[← Week 5](./week-5-nextjs-frontend.md) | [← Back to README](./README.md)

