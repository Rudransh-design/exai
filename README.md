# Exai

Exai is a small app that turns an old WhatsApp chat into someone you can talk to again. You export a conversation, upload the ZIP, and the model answers in that person’s voice — using their actual messages as memory, not a made-up persona.

The idea is simple: chats already hold how someone jokes, what they cared about, and how they replied. Exai reads that transcript, indexes it, and lets you send new messages. Each reply is pulled from retrieved lines in the export, so the model stays close to what was really said instead of inventing a generic friend.

Nothing is written to a long-term database. The transcript lives in memory for that browser session. When you leave, or the last socket disconnects, the session is deleted. That is on purpose — the upload is treated as a temporary memory, not an archive.

## How a session runs

WhatsApp’s export is a ZIP with a `_chat.txt` file (export **without media**). The frontend opens the ZIP in the browser, reads that text file, and guesses the other person’s name from the ZIP filename. It posts the transcript and that name to the Express backend, which creates a session and opens a Socket.IO room.

On the server, a LangGraph pipeline runs once to warm up the memory, then again on every message you send:

1. **Load the chat.** The transcript is split into lines and parsed into `{ timestamp, sender, message }`. Messages from the other person are kept so the model is answering *as them*, not as you.
2. **Index.** Those lines are embedded with Mistral and stored in an in-memory vector store. Indexing happens in small batches; the UI can show progress while that finishes.
3. **Retrieve.** When you send a message, the graph decides whether it needs more context and searches the index for the closest old lines (jokes, plans, repeated phrases, how they answered similar things).
4. **Generate.** Groq (DeepSeek) writes a reply in the same chat style, grounded in those retrieved lines. The reply is pushed back over the socket so it shows up like a normal chat bubble.

```
WhatsApp ZIP
    → browser extracts _chat.txt
    → POST /upload  (session id)
    → LangGraph: load → index → retrieve → generate
    → Socket.IO replies in the chat UI
```

After the first index, later messages skip re-parsing and only retrieve + generate. If you refresh or close the tab, that memory is gone and you upload again.

## Features

- WhatsApp ZIP upload (`_chat.txt`, no media)
- Name inferred from the export filename
- Embeddings over the old chat, not a blank prompt
- Live replies over Socket.IO
- In-memory only — cleared when the session ends

## Stack

| Layer | Tech |
| --- | --- |
| Frontend | Next.js 15, React, Tailwind |
| Backend | Express, Socket.IO |
| Graph | LangGraph + LangChain |
| Models | Groq / DeepSeek (chat), Mistral (embeddings) |

## Project layout

```
server.mjs      Express + Socket.IO
graph.mjs       LangGraph pipeline
nodes.mjs       load, index, retrieve, generate
frontend/       Next.js app (port 3000)
```

## Setup

```bash
git clone https://github.com/Rudransh-design/exai.git
cd exai
cp .env.example .env
# add MISTRAL_API_KEY and GROQ_API_KEY

npm install
npm run dev
```

In a second terminal:

```bash
cd frontend
cp .env.example .env.local
npm install
npm run dev
```

- Frontend: http://localhost:3000
- Backend: http://localhost:3001

## Environment

**Backend (`.env`)**

| Variable | What it is |
| --- | --- |
| `MISTRAL_API_KEY` | Embeddings |
| `GROQ_API_KEY` | Chat model |
| `PORT` | Backend port (default `3001`) |

**Frontend (`frontend/.env.local`)**

| Variable | What it is |
| --- | --- |
| `NEXT_PUBLIC_API_URL` | Backend URL (default `http://localhost:3001`) |

## WhatsApp export

On your phone: open a chat → menu → **Export chat** → **Without media**. Upload that ZIP.

## License

ISC
