# Week 1 — Python for AI & LLM Fundamentals

> **Connection to final project:** You'll write all backend logic in Python. This week removes the "Python feels foreign" friction before adding AI complexity.

**⏱ 10 hours total**

---

## Core Topics

- Python virtual environments (`venv`, `uv`), `requirements.txt`, `.env` files
- Type hints, `dataclasses`, `pydantic` v2 models (used heavily in FastAPI + LangChain)
- `async`/`await` in Python — essential for streaming LLM responses
- Anthropic Python SDK: client setup, `messages.create`, `stream` method
- Prompt engineering basics: system prompts, few-shot examples, role instructions

---

## Best Resources (Free)

| Resource | URL | Time |
|----------|-----|------|
| **Official Anthropic Python SDK docs** | https://docs.anthropic.com/en/api/getting-started | 1 hr |
| **Anthropic SDK GitHub (examples folder)** | https://github.com/anthropics/anthropic-sdk-python/tree/main/examples | 1 hr |
| **"Python for JS Developers" crash course** | https://www.youtube.com/watch?v=kqtD5dpn9C8 | 1.5 hr |
| **Pydantic v2 docs — Models section** | https://docs.pydantic.dev/latest/concepts/models/ | 1 hr |
| **Async Python — Real Python guide** | https://realpython.com/async-io-python/ | 1 hr |
| **Prompt engineering guide (Anthropic)** | https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview | 1 hr |

---

##  Mini Project: Personal CLI Chatbot

Build a `chat.py` terminal chatbot that:

1. Loads a system prompt from a `system_prompt.txt` file (make it "you are a tutor for X topic")
2. Maintains a `conversation_history` list across turns (multi-turn memory)
3. Streams Claude's response token-by-token to the terminal using `client.messages.stream()`
4. Accepts a `--topic` CLI flag (using `argparse`) that swaps the system prompt
5. Saves conversation history to a local `chat_log.json` on exit

```python
# chat.py skeleton
import argparse
import json
import os
from pathlib import Path
import anthropic
from dotenv import load_dotenv

load_dotenv()

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from .env

def load_system_prompt(topic: str) -> str:
    prompt_file = Path(f"prompts/{topic}.txt")
    if prompt_file.exists():
        return prompt_file.read_text()
    return f"You are a tutor specializing in {topic}. Be concise and give examples."

def chat(topic: str):
    system_prompt = load_system_prompt(topic)
    history = []

    print(f" Chatbot ready (topic: {topic}). Type 'quit' to exit.\n")

    while True:
        user_input = input("You: ").strip()
        if user_input.lower() in ("quit", "exit"):
            break

        history.append({"role": "user", "content": user_input})

        print("Assistant: ", end="", flush=True)
        full_response = ""

        with client.messages.stream(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            system=system_prompt,
            messages=history,
        ) as stream:
            for text in stream.text_stream:
                print(text, end="", flush=True)
                full_response += text

        print("\n")
        history.append({"role": "assistant", "content": full_response})

    # Save log
    with open("chat_log.json", "w") as f:
        json.dump({"topic": topic, "history": history}, f, indent=2)
    print("✅ Conversation saved to chat_log.json")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--topic", default="python", help="Topic to study")
    args = parser.parse_args()
    chat(args.topic)
```

---

## ✅ Success Criteria

- [ ] Can explain what a Pydantic model is and when to use it vs a plain `dict`
- [ ] Chatbot holds 10+ turn conversation without losing context
- [ ] Streaming output visibly prints token-by-token (not all at once)
- [ ] Can load a `.env` file without hardcoding the API key
- [ ] Conversation log persists between sessions

---

## Quick Setup

```bash
# Create virtual env
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# Install deps
pip install anthropic python-dotenv pydantic

# Create .env
echo "ANTHROPIC_API_KEY=your-key-here" > .env

# Run chatbot
python chat.py --topic java
```

---

[← Back to README](./README.md) | [Week 2 →](./week-2-embeddings-vectordb.md)
