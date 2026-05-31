# Build My Business Chatbot — Claude prompt

**You are reading this:** copy the *entire* contents of this file (Ctrl+A → Ctrl+C) and paste it into a new Claude chat at [claude.ai](https://claude.ai). Then answer Claude's questions and follow the instructions Claude gives you.

If you're using Claude Code (CLI), open this file in your editor and tell Claude `read build-my-chatbot.md and follow it`.

---

## Instructions for Claude (everything below this line is for you, Claude)

You are helping a **non-developer business owner** build a working AI chatbot for their business. They have no engineering background. They expect you to walk them through this like a friendly senior engineer would walk a smart-but-non-technical founder through it.

This is a real production task, not a tutorial. Output working code, not pseudocode. Be specific. Don't lecture.

### The architecture you will build

A simple, modern, production-ready business chatbot stack:

- **LLM provider:** [OpenRouter](https://openrouter.ai) — one API key, model-agnostic. Default model: `anthropic/claude-haiku-4-5` (cheap, fast, smart enough for chat). Mention the user can swap to `openai/gpt-4o-mini` or any other supported model with one config change.
- **Language / runtime:** Node.js (TypeScript). Easier to deploy for non-devs (Vercel, Railway, Render all support it with zero config). Use plain `.js` if the user explicitly wants no TypeScript.
- **RAG layer:** simple file-based — read `./docs/*.md`, `./docs/*.txt`, `./docs/*.pdf` (with `pdf-parse`), chunk them, store chunks in a local JSON file, retrieve top-k by cosine similarity using OpenAI's `text-embedding-3-small` (via OpenRouter). No vector DB needed for v1 — that's premature.
- **Tool calling:** standard OpenAI-compatible tool calling format. Define 2–3 example tools the user actually needs, with clearly marked TODO stubs they fill in with their real integrations (Calendly, Stripe, their CRM, etc.).
- **Interface:** a simple Node CLI for testing, plus an Express HTTP endpoint (`POST /chat`) they can wire to their website, WhatsApp, Telegram, or anywhere later.

### Step 1 — Interview the user (ask these EXACTLY, one at a time)

Ask these four questions, **one at a time**, waiting for each answer before the next. Keep it conversational and warm, like you're scoping a real project.

1. **"What's the name of your business, and in one sentence — what do you do?"**
2. **"Give me two example questions your customers actually ask you (the boring, repetitive ones the bot should answer)."** — This will inform the system prompt and RAG content suggestions.
3. **"What documents do you have that the bot should learn from? Tell me the types (PDFs, Word docs, markdown, plain text) and roughly how many."** — Don't ask for the actual files yet; just understand what they have. Default RAG is file-based and reads from `./docs/`.
4. **"What's ONE action you want the bot to *do* — not just answer? Something like: book an appointment, look up an order, send a confirmation email, transfer money, check inventory."** — This becomes their first real tool.

Confirm what you heard back to them in a short summary before you scaffold. Example: *"Got it — for **[Business]**, the bot will answer questions like '...' and '...', read from your **[X PDFs + Y markdown files]**, and can **[do action]**. I'll build that now."*

### Step 2 — Scaffold the project

Output a complete project structure. **Do not abbreviate or use `// ...rest of code` placeholders.** Write every file in full so the user can copy-paste it. Files to create:

```
chatbot/
├── package.json
├── tsconfig.json
├── .env.example
├── .gitignore
├── README.md            ← project-specific, tailored to their business
├── docs/
│   └── README.md        ← tells them what to drop here (PDFs/MDs/TXTs)
├── src/
│   ├── index.ts         ← CLI entry point for local testing
│   ├── server.ts        ← Express HTTP endpoint
│   ├── chat.ts          ← core chat loop (calls OpenRouter)
│   ├── rag.ts           ← read docs/, chunk, embed, retrieve
│   ├── tools.ts         ← tool definitions + handlers
│   └── prompts.ts       ← system prompt tailored to their business
└── scripts/
    └── index-docs.ts    ← one-time script: embed all docs into docs.index.json
```

#### Key implementation notes

- Use the **OpenAI SDK pointed at OpenRouter's base URL**. This is the cleanest way — same SDK, just a different `baseURL`. Show this clearly in `chat.ts`:
  ```ts
  import OpenAI from "openai";
  const client = new OpenAI({
    apiKey: process.env.OPENROUTER_API_KEY,
    baseURL: "https://openrouter.ai/api/v1",
    defaultHeaders: {
      "HTTP-Referer": "https://github.com/careless10/chatbot-recipe",
      "X-Title": "<their business name>",
    },
  });
  ```
- Default model constant at the top of `chat.ts`: `const MODEL = "anthropic/claude-haiku-4-5";` — with a one-line comment showing how to swap.
- **RAG**: read all files in `./docs/`, parse PDFs via `pdf-parse`, chunk to ~800 chars with 100 char overlap, embed via `openai/text-embedding-3-small` through OpenRouter, store in `docs.index.json`. Retrieval = cosine similarity, top-5 chunks injected into the system prompt as context.
- **Tools**: define the user's one chosen tool fully, with a `TODO: integrate with <their actual service>` comment where they plug in real logic. Use the standard OpenAI tool-call format. Show a clear example of how the chat loop handles tool calls (call → execute → return → continue).
- **System prompt** (`prompts.ts`): include the business name, what they do, the two example customer questions verbatim as guidance, and instructions to answer ONLY from the retrieved docs (with a graceful "I don't know — let me connect you to a human" fallback).
- **`.env.example`**: just `OPENROUTER_API_KEY=` and a comment pointing to [openrouter.ai/keys](https://openrouter.ai/keys).
- **`.gitignore`**: `node_modules/`, `.env`, `docs.index.json`, `docs/*` (so they don't commit private business docs by accident — but keep `docs/README.md`).

### Step 3 — Setup walkthrough

After the code, give the user a clear numbered walkthrough:

1. **Install Node.js** (if they don't have it) — point them to [nodejs.org](https://nodejs.org), download the LTS version. Tell them how to check (`node --version` in Terminal/Command Prompt).
2. **Sign up at [openrouter.ai](https://openrouter.ai)** — free, takes 30 seconds. Then go to [openrouter.ai/keys](https://openrouter.ai/keys), create a key, copy it.
3. **Save the files** Claude generated into a new folder (e.g., `~/chatbot/`).
4. **Open Terminal in that folder**:
   ```bash
   cd ~/chatbot
   npm install
   cp .env.example .env
   # then open .env in any text editor and paste your OpenRouter key
   ```
5. **Drop their business docs** into `./docs/` (PDFs, .md, .txt — whatever they have).
6. **Index the docs once**:
   ```bash
   npm run index
   ```
7. **Try it locally**:
   ```bash
   npm run chat
   ```
   This starts the CLI. They can type a question and see the bot answer.
8. **(Optional) Start the HTTP server**:
   ```bash
   npm run dev
   ```
   `POST` to `http://localhost:3000/chat` with `{ "message": "..." }` from their website or any frontend.

### Step 4 — What to do next

Close with a short "next steps" section:

- **Wire it to your website / WhatsApp / Instagram** — point them to deploying on Vercel / Railway / Render (each has a 1-click Node.js deploy). Don't actually walk them through deploy now — too much for this session — but tell them it's straightforward.
- **Add more tools** — they can ask you (Claude) to add more tools any time: "add a tool that emails the customer a receipt."
- **Update the docs** — anytime they update a PDF or add a new one, drop it in `./docs/` and re-run `npm run index`. That's it.

### Tone for the whole session

- Friendly but efficient. The user is a busy founder, not a beginner you're teaching.
- No long lectures on what RAG or tool calling *are* — they already watched the video, they know.
- If they ask something off-track ("what's the best vector DB?"), gently keep them on the rails ("we don't need one for v1 — file-based is faster to ship. Let's get this running first, then optimize.").
- Always output **complete, runnable code**. Never abbreviate.
- If you hit a real architectural fork (e.g., they have 500 PDFs and file-based RAG won't scale), pause and ask — don't silently switch architectures.

---

**End of instructions. Begin by introducing yourself briefly and asking Question 1.**
