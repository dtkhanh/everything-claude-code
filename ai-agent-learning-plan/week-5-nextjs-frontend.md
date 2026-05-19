# Week 5 — Next.js Frontend + Full-Stack Integration

> **Connection to final project:** The user-facing interface of your learning agent. Leverages your existing Next.js skills, but adds AI-specific patterns like streaming chat and document management.

**⏱ 10 hours total**

---

## Core Topics

- Next.js App Router: Server Components vs Client Components — when each applies for AI apps
- Streaming UI with `ReadableStream` / `EventSource` — consuming SSE from FastAPI
- `useRef` for auto-scroll in chat windows, `useOptimistic` for instant message display
- File upload with `<input type="file">` + drag-and-drop
- `shadcn/ui` component library for rapid UI (`Card`, `ScrollArea`, `Input`, `Button`)
- `react-markdown` + `rehype-highlight` for rendering formatted LLM responses
- Environment variable: `NEXT_PUBLIC_API_URL` pointing to FastAPI

---

## Best Resources (Free)

| Resource | URL | Time |
|----------|-----|------|
| **Vercel AI SDK docs (streaming patterns)** | https://sdk.vercel.ai/docs/getting-started | 1.5 hr |
| **shadcn/ui docs + CLI setup** | https://ui.shadcn.com/docs/installation/next | 1 hr |
| **Next.js App Router official docs** | https://nextjs.org/docs/app | 1 hr |
| **"Build a ChatGPT clone" — Fireship (YouTube)** | https://www.youtube.com/watch?v=O2mBNMm-tDQ | 1 hr |
| **react-markdown GitHub** | https://github.com/remarkjs/react-markdown | 0.5 hr |
| **MDN: EventSource / SSE reference** | https://developer.mozilla.org/en-US/docs/Web/API/EventSource | 0.5 hr |

---

## Project Setup

```bash
npx create-next-app@latest frontend --typescript --tailwind --app
cd frontend

# shadcn/ui
npx shadcn@latest init
npx shadcn@latest add card scroll-area input button badge separator

# Markdown rendering
npm install react-markdown rehype-highlight highlight.js

# Set API URL
echo "NEXT_PUBLIC_API_URL=http://localhost:8000" >> .env.local
```

---

## Page Structure

```
app/
├── page.tsx              ← redirect to /chat
├── chat/
│   └── page.tsx          ← main chat interface
├── documents/
│   └── page.tsx          ← document management
└── components/
    ├── ChatWindow.tsx    ← streaming message display
    ├── MessageBubble.tsx ← renders markdown response
    ├── SourcesAccordion.tsx ← expandable source chunks
    └── DocumentUpload.tsx ← drag-and-drop file upload
```

---

## 🛠 Mini Project: Learning Agent Chat UI

### `app/chat/page.tsx` — Main Chat Interface
```tsx
"use client";
import { useState, useRef, useEffect } from "react";
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";
import { ScrollArea } from "@/components/ui/scroll-area";
import MessageBubble from "@/components/MessageBubble";

interface Message {
  role: "user" | "assistant";
  content: string;
  sources?: Array<{ filename: string; chunk: string }>;
  isStreaming?: boolean;
}

export default function ChatPage() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState("");
  const [isLoading, setIsLoading] = useState(false);
  const bottomRef = useRef<HTMLDivElement>(null);

  // Auto-scroll on new messages
  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  async function sendMessage() {
    if (!input.trim() || isLoading) return;

    const userMessage = input.trim();
    setInput("");
    setIsLoading(true);

    // Add user message immediately (optimistic)
    setMessages((prev) => [...prev, { role: "user", content: userMessage }]);

    // Add empty assistant message placeholder
    setMessages((prev) => [
      ...prev,
      { role: "assistant", content: "", isStreaming: true },
    ]);

    try {
      const response = await fetch(
        `${process.env.NEXT_PUBLIC_API_URL}/chat`,
        {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            message: userMessage,
            history: messages.map((m) => ({ role: m.role, content: m.content })),
          }),
        }
      );

      const reader = response.body!.getReader();
      const decoder = new TextDecoder();
      let accumulatedText = "";
      let sources: Message["sources"] = [];

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const lines = decoder.decode(value).split("\n");
        for (const line of lines) {
          if (!line.startsWith("data: ")) continue;
          const data = line.slice(6);
          if (data === "[DONE]") break;

          try {
            const parsed = JSON.parse(data);
            if (parsed.token) {
              accumulatedText += parsed.token;
              // Update last message in real-time
              setMessages((prev) => {
                const updated = [...prev];
                updated[updated.length - 1] = {
                  role: "assistant",
                  content: accumulatedText,
                  isStreaming: true,
                };
                return updated;
              });
            }
            if (parsed.sources?.length) {
              sources = parsed.sources;
            }
          } catch {}
        }
      }

      // Finalize: remove streaming flag, add sources
      setMessages((prev) => {
        const updated = [...prev];
        updated[updated.length - 1] = {
          role: "assistant",
          content: accumulatedText,
          sources,
          isStreaming: false,
        };
        return updated;
      });
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <div className="flex flex-col h-screen max-w-3xl mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">🧠 AI Learning Agent</h1>

      <ScrollArea className="flex-1 border rounded-lg p-4 mb-4">
        {messages.length === 0 && (
          <p className="text-muted-foreground text-center mt-8">
            Ask me anything about your study documents...
          </p>
        )}
        {messages.map((msg, i) => (
          <MessageBubble key={i} message={msg} />
        ))}
        <div ref={bottomRef} />
      </ScrollArea>

      <div className="flex gap-2">
        <Input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && sendMessage()}
          placeholder="Ask a question about your documents..."
          disabled={isLoading}
        />
        <Button onClick={sendMessage} disabled={isLoading}>
          {isLoading ? "..." : "Send"}
        </Button>
      </div>
    </div>
  );
}
```

### `components/MessageBubble.tsx` — Markdown Rendering
```tsx
import ReactMarkdown from "react-markdown";
import rehypeHighlight from "rehype-highlight";
import "highlight.js/styles/github.css";
import SourcesAccordion from "./SourcesAccordion";

export default function MessageBubble({ message }: { message: any }) {
  const isUser = message.role === "user";

  return (
    <div className={`mb-4 ${isUser ? "text-right" : "text-left"}`}>
      <div
        className={`inline-block max-w-[85%] p-3 rounded-lg ${
          isUser
            ? "bg-blue-600 text-white"
            : "bg-gray-100 text-gray-900"
        }`}
      >
        {isUser ? (
          message.content
        ) : (
          <>
            <ReactMarkdown
              rehypePlugins={[rehypeHighlight]}
              components={{
                code: ({ node, inline, className, children, ...props }) => (
                  inline
                    ? <code className="bg-gray-200 px-1 rounded text-sm" {...props}>{children}</code>
                    : <code className={`${className} block overflow-x-auto`} {...props}>{children}</code>
                ),
              }}
            >
              {message.content}
            </ReactMarkdown>
            {message.isStreaming && (
              <span className="inline-block w-2 h-4 bg-gray-500 animate-pulse ml-1" />
            )}
            {message.sources?.length > 0 && (
              <SourcesAccordion sources={message.sources} />
            )}
          </>
        )}
      </div>
    </div>
  );
}
```

### `components/SourcesAccordion.tsx`
```tsx
import { useState } from "react";

export default function SourcesAccordion({ sources }: { sources: any[] }) {
  const [open, setOpen] = useState(false);
  return (
    <div className="mt-2 border-t pt-2">
      <button
        className="text-xs text-gray-500 hover:text-gray-700"
        onClick={() => setOpen(!open)}
      >
        📚 {sources.length} source{sources.length > 1 ? "s" : ""} {open ? "▲" : "▼"}
      </button>
      {open && (
        <div className="mt-1 space-y-1">
          {sources.map((s, i) => (
            <div key={i} className="text-xs bg-gray-50 p-2 rounded border">
              <span className="font-medium">{s.filename}</span>
              <p className="text-gray-500 mt-1 line-clamp-2">{s.chunk}</p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## Documents Page — File Upload

```tsx
// app/documents/page.tsx
"use client";
import { useState, useCallback } from "react";
import { Button } from "@/components/ui/button";
import { Card } from "@/components/ui/card";

export default function DocumentsPage() {
  const [documents, setDocuments] = useState([]);
  const [uploading, setUploading] = useState(false);

  const handleUpload = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;
    setUploading(true);
    const formData = new FormData();
    formData.append("file", file);
    await fetch(`${process.env.NEXT_PUBLIC_API_URL}/ingest`, {
      method: "POST",
      body: formData,
    });
    setUploading(false);
    loadDocuments();
  };

  const loadDocuments = async () => {
    const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/documents`);
    setDocuments(await res.json());
  };

  return (
    <div className="max-w-2xl mx-auto p-4">
      <h1 className="text-2xl font-bold mb-6">📄 Knowledge Base</h1>
      <div className="border-2 border-dashed rounded-lg p-8 text-center mb-6">
        <input type="file" accept=".pdf,.txt" onChange={handleUpload} className="hidden" id="upload" />
        <label htmlFor="upload" className="cursor-pointer">
          <p className="text-muted-foreground">Drop PDF/TXT here or click to upload</p>
          {uploading && <p className="text-blue-500 mt-2">Indexing document...</p>}
        </label>
      </div>
      {/* List documents */}
      {documents.map((doc: any) => (
        <Card key={doc.id} className="p-4 mb-2 flex justify-between items-center">
          <div>
            <p className="font-medium">{doc.filename}</p>
            <p className="text-sm text-muted-foreground">{doc.chunk_count} chunks</p>
          </div>
          <Button variant="destructive" size="sm"
            onClick={async () => {
              await fetch(`${process.env.NEXT_PUBLIC_API_URL}/documents/${doc.id}`, { method: "DELETE" });
              loadDocuments();
            }}>
            Delete
          </Button>
        </Card>
      ))}
    </div>
  );
}
```

---

## ✅ Success Criteria

- [ ] Upload a PDF in Documents page, ask a question in Chat — full round trip works
- [ ] Claude's response streams visibly in real-time (not all at once)
- [ ] Markdown renders correctly — code blocks are syntax-highlighted
- [ ] Sources accordion shows the actual retrieved chunks for every answer
- [ ] App works on mobile viewport (responsive layout)

---

[← Week 4](./week-4-fastapi-backend.md) | [Week 6 →](./week-6-production-deploy.md)

