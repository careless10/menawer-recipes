# Build My Business Chatbot — Claude prompt (v2)

> **You are reading this:** copy the *entire* contents of this file (Ctrl/Cmd+A → Ctrl/Cmd+C) and paste it into Claude. Hit enter. Claude takes it from there.

This recipe is from the [@eng_menawer](https://instagram.com/eng_menawer) channel's chatbot reel. It builds a **Next.js chatbot for your business** with **RAG** (it reads your docs and answers from them) and **tool calling** (it can DO things, not just talk). Local first, optional one-click deploy at the end.

---

## Instructions for Claude — everything below this line is for you, Claude

You are helping a **non-technical business owner** build a working AI chatbot for their business. They have no engineering background, and they almost certainly haven't installed Node.js before. Do not assume any prior knowledge. Verify everything. Be patient. Use friendly language. Never lecture.

### Step 0 — Detect your environment and ANNOUNCE it

Before you do anything else, figure out which environment you're running in. The whole rest of this recipe depends on this.

There are three possibilities:

- **A. Claude Code** (the CLI on the user's actual machine) — you have `Bash`, `Write`, `Edit`, `Read`, `AskUserQuestion`, `WebFetch`, and full filesystem access. **You can do the work for the user.** Default to this branch if you have these tools.
- **B. Codex / sandbox** (Python or shell sandbox) — you have a sandboxed filesystem and shell, but you can't install software on the user's machine. **You build the project in the sandbox and have the user download it.**
- **C. claude.ai web** (the web chat, no tools) — you can only output text. **You walk the user through every step manually.**

Quick check: try to read your own tool list. If you see tools named `Bash`, `Write`, and `AskUserQuestion`, you're in Claude Code (A). If you see Python execution but no `Write` tool, you're Codex (B). If you have nothing but text output, you're web (C).

**State clearly to the user which environment you're in and what that means for them.** Example:

> *"I'm running as Claude Code on your Mac, which means I can install things, write files, and run commands for you. You'll just answer a few questions and watch. Ready?"*

Then proceed to Step 1.

---

### Step 1 — Set the right tone

Throughout this session:

- **No jargon without translation.** First time you say "API key," explain it in one sentence ("a password your code uses to talk to a service").
- **Verify visually after each step.** Don't say "done" without showing what changed: `✅ Node v22.21.1 detected`, `✅ folder created at ~/Desktop/myco-chatbot`, etc.
- **Never blame the user.** If something fails, calmly say "that's a known one, let me fix it" — even if it wasn't.
- **Use AskUserQuestion** (Branch A only) for every choice — it gives the user a clean picker instead of asking them to type. Almost every question in this recipe is a multi-choice.

---

### Step 2 — Detect the operating system

**Branch A (Claude Code):** Run this and announce what you see:
```bash
uname -s && uname -m
```
- Output starting with `Darwin` → macOS. You'll use Homebrew if available.
- Output starting with `Linux` → Linux. Use the user's package manager (`apt`, `dnf`, etc.).
- Output starting with `MINGW`/`CYGWIN`/`MSYS` → Windows running a Unix shell (rare). Treat as Linux-ish.
- If `uname` fails entirely, it's likely Windows PowerShell. Switch your approach.

**Branch B/C:** Ask the user with AskUserQuestion (if available) or plain text: macOS / Windows / Linux?

Tell the user what you detected so they know you're paying attention.

---

### Step 3 — Interview the user (the personalization)

Ask these **four questions, one at a time**. In Branch A, **use AskUserQuestion** with the suggested options. In B/C, ask conversationally.

**Q1: What's your business?**
- Free text. Get name + one sentence about what they do.
- Use this for the bot's system prompt and the folder name.

**Q2: What are two example questions customers ask you that the bot should handle?**
- Free text. Don't accept vague answers — push for *real* examples ("when's the next available slot?", "do you ship to Riyadh?", etc.). These go in the bot's prompt as guidance.

**Q3: What docs do you have for the bot to learn from?** (AskUserQuestion, multiSelect)
- PDFs (price lists, brochures, policies)
- Word docs
- Markdown / plain text notes
- Web pages (a list of URLs you'll paste)
- I don't have any yet — I'll add them later

**Q4: What's ONE action the bot should do besides answering?** (AskUserQuestion, single-select)
- Book an appointment
- Look up an order or customer
- Send an email or SMS
- Process a payment
- Check inventory / stock
- Other (you'll describe it)

**After all four, repeat back what you heard and confirm**:

> *"Got it — for **<business>**, the bot will answer questions like '<example1>' and '<example2>', read from your **<doc types>**, and can **<action>**. I'll build that now. This will take about 10 minutes."*

Don't proceed until the user confirms.

---

### Step 4 — Check prerequisites and install what's missing

**Branch A (Claude Code):** run checks and install via Bash. Be verbose so the user can follow.

1. Check Node.js (need v22+):
   ```bash
   node --version
   ```
   - If output is `v22.x.x` or higher → ✅ `Node X.Y.Z detected, good to go.`
   - If "command not found" or version <22 → install:
     - macOS with Homebrew: `brew install node@22`
     - macOS without Homebrew: install Homebrew first (`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`), then `brew install node@22`
     - Linux: `curl -fsSL https://fnm.vercel.app/install | bash && fnm install 22 && fnm use 22`
     - Windows: ask the user to download the LTS installer from https://nodejs.org and run it, then restart their terminal.

2. Check npm:
   ```bash
   npm --version
   ```
   Comes with Node. If missing, the Node install failed — debug.

3. Check git (for later, if deploying):
   ```bash
   git --version
   ```
   - Missing on Mac: triggers Xcode Command Line Tools install ("xcode-select --install"). Have user click "Install" in the popup, wait, retry.

After each: tell the user what you got and what it means in plain words.

**Branch C (web):** walk the user through the same — give them links, screenshots descriptions, and the exact commands to type in Terminal/PowerShell. Wait for them to confirm each step before moving on. Don't dump everything at once.

---

### Step 5 — Get the OpenRouter API key

OpenRouter is the bot's brain. It's one signup that gives you access to Claude, GPT, Gemini, Llama — every major AI model. Free credits to start; pay-as-you-go after.

**Walk the user through:**

1. Open https://openrouter.ai in their browser → click "Sign in" (Google or email both work, 30 seconds)
2. After signing in, open https://openrouter.ai/keys → click "Create Key" → name it "my-chatbot" → copy the key (it starts with `sk-or-v1-...`)
3. **Important:** tell the user to keep this tab open or paste the key into a notes app — they'll need it in a minute, and OpenRouter doesn't show it again after you close the modal.

**Branch A:** use AskUserQuestion to ask "Do you have your OpenRouter key copied?" with options [Yes I have it / I'm signing up now / I need help]. Wait until they say yes.

**Branch C:** wait for the user to say they have the key.

You will use this key in Step 7 to populate `.env.local`. **Never write it into committed code** — only into `.env.local` which is git-ignored.

---

### Step 6 — Scaffold the Next.js project

**Branch A (Claude Code):** run these commands via Bash, in order. Stop on failure.

```bash
# Pick a folder name from the business name (e.g., "mycafe" or "alabbar-trading")
SLUG="<business-slug>"   # replace with a kebab-case version of the business name
mkdir -p "$HOME/Desktop/${SLUG}-chatbot"
cd "$HOME/Desktop/${SLUG}-chatbot"

# Scaffold Next.js with App Router + TypeScript + Tailwind
npx --yes create-next-app@latest . \
  --typescript --tailwind --app \
  --no-eslint --src-dir \
  --import-alias '@/*' --yes
```

`create-next-app` will produce a fresh Next.js 15 project. Wait for it to finish.

Then create the chatbot files using your `Write` tool. Each file in full — no placeholders, no `// ...rest of code`:

**File 1: `src/lib/openrouter.ts`** — the LLM client
```ts
import OpenAI from "openai";

export const openrouter = new OpenAI({
  apiKey: process.env.OPENROUTER_API_KEY,
  baseURL: "https://openrouter.ai/api/v1",
  defaultHeaders: {
    "HTTP-Referer": "https://github.com/careless10/chatbot-recipe",
    "X-Title": "<BUSINESS_NAME>",
  },
});

// Default model — swap with any OpenRouter id (e.g., "openai/gpt-4o-mini")
export const MODEL = "anthropic/claude-haiku-4-5";
```

**File 2: `src/lib/prompts.ts`** — the system prompt baked with the user's answers
```ts
export const SYSTEM_PROMPT = `You are the customer assistant for <BUSINESS_NAME>.

About the business:
<ONE_SENTENCE_DESCRIPTION>

You answer customer questions using the documents the business owner uploaded.
Always answer based on those documents. If the answer isn't there, say:
"That's a great question — let me connect you to a human." Don't make things up.

Example questions you'll get:
- "<EXAMPLE_QUESTION_1>"
- "<EXAMPLE_QUESTION_2>"

Keep responses short and friendly. Use the same language the customer wrote in.`;
```

**File 3: `src/lib/rag.ts`** — read docs, chunk, embed, retrieve
```ts
import fs from "node:fs/promises";
import path from "node:path";
import { openrouter } from "./openrouter";

const INDEX_PATH = path.join(process.cwd(), "docs.index.json");
const DOCS_DIR = path.join(process.cwd(), "docs");

type Chunk = { source: string; text: string; embedding: number[] };

export async function buildIndex() {
  const files = await fs.readdir(DOCS_DIR).catch(() => []);
  const chunks: Chunk[] = [];
  for (const file of files) {
    if (file === "README.md") continue;
    const full = path.join(DOCS_DIR, file);
    let text = "";
    if (file.endsWith(".pdf")) {
      const pdfParse = (await import("pdf-parse")).default;
      const buf = await fs.readFile(full);
      text = (await pdfParse(buf)).text;
    } else {
      text = await fs.readFile(full, "utf8");
    }
    // chunk to ~800 chars with 100 char overlap
    for (let i = 0; i < text.length; i += 700) {
      const slice = text.slice(i, i + 800).trim();
      if (slice.length < 50) continue;
      const embedding = await embed(slice);
      chunks.push({ source: file, text: slice, embedding });
    }
  }
  await fs.writeFile(INDEX_PATH, JSON.stringify(chunks));
  return chunks.length;
}

async function embed(text: string) {
  const r = await openrouter.embeddings.create({
    model: "openai/text-embedding-3-small",
    input: text,
  });
  return r.data[0].embedding;
}

export async function retrieve(query: string, topK = 5) {
  const raw = await fs.readFile(INDEX_PATH, "utf8").catch(() => "[]");
  const chunks: Chunk[] = JSON.parse(raw);
  if (!chunks.length) return [];
  const q = await embed(query);
  return chunks
    .map((c) => ({ ...c, score: cos(c.embedding, q) }))
    .sort((a, b) => b.score - a.score)
    .slice(0, topK);
}

function cos(a: number[], b: number[]) {
  let dot = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    na += a[i] * a[i];
    nb += b[i] * b[i];
  }
  return dot / (Math.sqrt(na) * Math.sqrt(nb));
}
```

**File 4: `src/lib/tools.ts`** — the user's chosen action as a real OpenAI-format tool
```ts
import type { ChatCompletionTool } from "openai/resources/chat/completions";

export const TOOLS: ChatCompletionTool[] = [
  {
    type: "function",
    function: {
      name: "<USER_CHOSEN_TOOL_NAME>",  // e.g., "book_appointment"
      description: "<DESCRIPTION>",      // e.g., "Books an appointment in the calendar"
      parameters: {
        type: "object",
        properties: {
          // example for book_appointment
          customer_name: { type: "string" },
          date: { type: "string", description: "ISO date YYYY-MM-DD" },
          time: { type: "string", description: "HH:MM in 24h" },
        },
        required: ["customer_name", "date", "time"],
      },
    },
  },
];

export async function handleToolCall(name: string, args: any) {
  // TODO: wire this to your real system (Calendly, your CRM, an email service, etc.)
  console.log(`[tool] ${name}`, args);
  return { ok: true, message: "Booked (stub)." };
}
```

(Customize the tool definition based on the user's Q4 answer.)

**File 5: `src/app/api/chat/route.ts`** — the API the chat UI calls
```ts
import { NextRequest, NextResponse } from "next/server";
import { openrouter, MODEL } from "@/lib/openrouter";
import { SYSTEM_PROMPT } from "@/lib/prompts";
import { retrieve } from "@/lib/rag";
import { TOOLS, handleToolCall } from "@/lib/tools";

export async function POST(req: NextRequest) {
  const { messages } = await req.json();
  const userMsg = messages[messages.length - 1].content as string;
  const context = await retrieve(userMsg);
  const systemWithContext =
    SYSTEM_PROMPT +
    "\n\nRelevant docs:\n" +
    context.map((c) => `# ${c.source}\n${c.text}`).join("\n\n---\n\n");

  let resp = await openrouter.chat.completions.create({
    model: MODEL,
    messages: [{ role: "system", content: systemWithContext }, ...messages],
    tools: TOOLS,
  });

  while (resp.choices[0].finish_reason === "tool_calls") {
    const call = resp.choices[0].message.tool_calls![0];
    const result = await handleToolCall(call.function.name, JSON.parse(call.function.arguments));
    messages.push(resp.choices[0].message);
    messages.push({ role: "tool", tool_call_id: call.id, content: JSON.stringify(result) });
    resp = await openrouter.chat.completions.create({
      model: MODEL,
      messages: [{ role: "system", content: systemWithContext }, ...messages],
      tools: TOOLS,
    });
  }

  return NextResponse.json({ message: resp.choices[0].message });
}
```

**File 6: `src/app/page.tsx`** — chat UI (Tailwind, mobile-friendly)
```tsx
"use client";
import { useState } from "react";

type Msg = { role: "user" | "assistant"; content: string };

export default function Home() {
  const [messages, setMessages] = useState<Msg[]>([]);
  const [input, setInput] = useState("");
  const [busy, setBusy] = useState(false);

  async function send() {
    if (!input.trim() || busy) return;
    const next = [...messages, { role: "user" as const, content: input }];
    setMessages(next);
    setInput("");
    setBusy(true);
    const r = await fetch("/api/chat", {
      method: "POST",
      body: JSON.stringify({ messages: next }),
    });
    const data = await r.json();
    setMessages([...next, data.message]);
    setBusy(false);
  }

  return (
    <main className="mx-auto max-w-2xl min-h-screen p-6 flex flex-col">
      <h1 className="text-2xl font-bold mb-6">Customer Assistant</h1>
      <div className="flex-1 space-y-4 mb-4 overflow-y-auto">
        {messages.map((m, i) => (
          <div
            key={i}
            className={`p-4 rounded-2xl max-w-[85%] ${
              m.role === "user"
                ? "bg-orange-500 text-white self-end ml-auto"
                : "bg-gray-100 self-start"
            }`}
          >
            {m.content}
          </div>
        ))}
        {busy && <div className="text-gray-400 text-sm">...</div>}
      </div>
      <div className="flex gap-2">
        <input
          className="flex-1 border rounded-xl px-4 py-3"
          placeholder="Ask something..."
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && send()}
        />
        <button
          className="bg-orange-500 text-white px-5 py-3 rounded-xl font-semibold disabled:opacity-50"
          onClick={send}
          disabled={busy}
        >
          Send
        </button>
      </div>
    </main>
  );
}
```

**File 7: `scripts/index-docs.ts`** — one-time embedding script
```ts
import { buildIndex } from "../src/lib/rag";

buildIndex()
  .then((n) => {
    console.log(`✅ Indexed ${n} chunks from /docs`);
    process.exit(0);
  })
  .catch((err) => {
    console.error("❌ Indexing failed:", err);
    process.exit(1);
  });
```

**File 8: `docs/README.md`** — instructions for the user
```md
# Drop your business docs here

Supported: `.pdf`, `.md`, `.txt`

After adding or changing docs, run:

    npm run index

Then restart the dev server: `npm run dev`
```

**File 9: `.env.local.example`** — env template
```
OPENROUTER_API_KEY=
```

**File 10:** add to `package.json` scripts (use Edit tool):
```json
"index": "tsx scripts/index-docs.ts"
```

**File 11:** install the extra deps in one shot:
```bash
npm install openai pdf-parse
npm install -D tsx @types/pdf-parse
```

**Branch B/C:** print each file as a code block; tell the user to save them in the same paths inside the project folder, then run the install commands.

---

### Step 7 — Wire the key and run it

1. Copy `.env.local.example` → `.env.local`:
   ```bash
   cp .env.local.example .env.local
   ```
2. Open `.env.local` and paste the OpenRouter key after `OPENROUTER_API_KEY=`. **Branch A:** use Write to update `.env.local` directly with the key the user gave you. Confirm the line shows `OPENROUTER_API_KEY=sk-or-v1-...` (don't print the actual key back to the user; just confirm it's set).
3. Start the dev server:
   ```bash
   npm run dev
   ```
4. **Branch A:** watch the Bash output until you see `Ready in <Xms>` and `Local: http://localhost:3000`. Tell the user: *"✅ Your chatbot is running. Open http://localhost:3000 in your browser."*
5. **Sanity test:** ask the user to send the message "hello" in the chat. They should see a friendly reply. If they get an error, check `.env.local` first.

If the user has docs ready, skip to Step 8. If not, they can stop here, add docs later, re-run `npm run index`, and restart.

---

### Step 8 — Add the docs and index them

1. Tell the user to drop their docs into the `docs/` folder (PDFs, .md, .txt — whatever they have).
2. Run the indexer:
   ```bash
   npm run index
   ```
3. Watch for `✅ Indexed N chunks from /docs`. If N is 0, the docs folder is empty.
4. Restart `npm run dev` so the server picks up the new `docs.index.json`.
5. Have the user test a question that's actually answered in their docs — verify the bot uses the doc content.

---

### Step 9 — (Optional) Put it online

Ask the user (AskUserQuestion in Branch A):

> *"Local is working. Do you want to put this online so other people can use it?"*
> - Yes → continue
> - Not yet → wrap up at Step 10

If yes, ask **where**:

> - **Cloudflare Pages** (recommended — free + fast + works great with Next.js)
> - **Vercel** (also free, made by the Next.js team)
> - **Skip — I'll figure it out later**

**If Cloudflare + Branch A:**

1. **Read the current Cloudflare MCP install docs** with WebFetch — don't guess, the install method changes. Try `https://developers.cloudflare.com/agents/model-context-protocol/` first, then fall back to `https://github.com/cloudflare/mcp-server-cloudflare/blob/main/README.md`.
2. Help the user install the official Cloudflare MCP server. The standard flow is something like `claude mcp add cloudflare --transport http https://mcp.cloudflare.com/mcp`, then the user authenticates in a browser when prompted.
3. **Tell the user to restart Claude Code** so the new MCP attaches. (Or `/mcp` if available.)
4. Once it's connected, walk them through:
   - Creating a Cloudflare account at cloudflare.com/sign-up if they don't have one
   - Pushing the project to GitHub first (`git init && git add . && git commit -m 'init' && gh repo create --public --source=. --push`)
   - Using the Cloudflare MCP tools to create a Pages project from the repo, set the build command to `npm run build`, set output to `.next`, and add the `OPENROUTER_API_KEY` env var
   - Triggering the first deploy and reporting back the `*.pages.dev` URL
5. Verify the deployed URL responds the same as local.

**If Cloudflare + Branch C** (no MCP available):
- Walk them through the Cloudflare dashboard manually: Workers & Pages → Create → Connect Git → pick the repo → Framework preset: Next.js → set env var → Deploy.

**If Vercel:** simpler — `npx vercel` and follow the prompts. They'll need to sign up at vercel.com first.

---

### Step 10 — Wrap up

Tell the user:

- **Local URL:** http://localhost:3000
- **Live URL:** (if deployed) the `*.pages.dev` or `*.vercel.app`
- **To add tools:** "Come back and ask me — 'add a tool that sends a confirmation email to the customer' or 'add a tool that looks up an order by phone number.'"
- **To update docs:** Drop new files in `docs/`, run `npm run index`, restart.
- **To swap the LLM model:** Open `src/lib/openrouter.ts` and change the `MODEL` constant. Recommended swaps: `openai/gpt-4o-mini` (cheap GPT), `google/gemini-2.5-flash` (cheap Gemini), or `anthropic/claude-sonnet-4-7` (smarter Claude).
- **If anything breaks later:** paste the error in a new chat with this same recipe attached, and Claude will debug.

---

## Branch C (web claude.ai) fallback — short summary

If you're web Claude with no tools, the flow above doesn't change conceptually — you just walk the user through every command yourself. Be EXTRA patient. Wait for them to confirm each step. Print every file in full as a code block. Don't promise to "run" anything — you can't.

---

**End of instructions. Begin Step 0 now.**
