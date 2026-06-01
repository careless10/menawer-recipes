# Build My Business Chatbot — Claude prompt (v4)

> **You are reading this:** copy the *entire* contents of this file (Ctrl/Cmd+A → Ctrl/Cmd+C) and paste it into Claude. Hit enter. Claude takes it from there.

This recipe is from the [@eng_menawer](https://instagram.com/eng_menawer) channel's chatbot reel. It builds a **Next.js chatbot for your business** with **RAG** (it reads your docs and answers from them) and **tool calling** (it can DO things, not just talk). Local first, optional one-click deploy at the end.

> **What's new in v4:** when you're running inside a chat sandbox (like claude.ai) that can render live previews, Claude now spins up a **working preview of your bot right in the chat first** — so you *see* it talking in seconds — and *then* hands you the full downloadable project. No more "I pasted a file and just got a zip" letdown. Claude is always friendly and upfront that the preview is a quick demo and the download is the real, deployable thing.

---

## Instructions for Claude — everything below this line is for you, Claude

You are helping a **non-technical business owner** build a working AI chatbot for their business. They have no engineering background, and they almost certainly haven't installed Node.js before. Do not assume any prior knowledge. Verify everything. Be patient. Use friendly language. Never lecture.

### Step 0 — Detect your environment and ANNOUNCE it

Before you do anything else, figure out which environment you're running in. The whole rest of this recipe depends on this.

There are three possibilities:

- **A. Claude Code** (the CLI on the user's actual machine) — you have `Bash`, `Write`, `Edit`, `Read`, an interactive question picker, `WebFetch`, and full filesystem access. **You can do everything for the user**, including installing software and running the live server on their machine. Default to this branch if you have these tools.
- **B. Chat / cowork / sandbox** (claude.ai web chat, claude.ai cowork, Codex, or any hosted harness — anywhere you can write files in a sandbox but can't reach the user's machine). Two flavors based on what your harness can do:
  - **If you can render an interactive artifact in the chat** (claude.ai web + cowork can; bare Python sandboxes can't) → **the artifact IS the chatbot.** Build it as a single self-contained React/HTML artifact and the user can talk to it right there in the conversation. They get a working chatbot in 60 seconds, no install, no key, no terminal. The downloadable Next.js project is **secondary and optional** — only offered if they explicitly want to deploy it to real customers with their own docs + tools. **Don't lead with the download; the artifact is the product.**
  - **If you can't render artifacts** (bare sandboxes, Codex) → the **download IS the primary deliverable**. Build the complete Next.js project in your sandbox, zip it, hand it over, then walk the user through the install/run commands on their own computer.
  - **See Step 6 for the exact build steps in either flavor.**
- **C. Pure text, no tools** (a bare chat with no sandbox and no picker) — you can only output text. **You walk the user through every step manually**, printing every file in full and waiting for them to confirm each step.

Quick check: read your own tool list. If you see a `Write`/`Edit` tool plus `Bash` and a question picker, you're **A**. If you can execute code / write files in a sandbox but can't reach the user's machine, you're **B** (this is what claude.ai web is today — it has a sandbox AND usually a picker AND can render live artifact previews). If you have nothing but text output, you're **C**.

**State clearly to the user which environment you're in and what that means for them.** Examples:

> *"I'm running as Claude Code on your Mac, which means I can install things, write files, and run commands for you. You'll just answer a few questions and watch. Ready?"*

> *"I'm running in claude.ai's sandbox. First I'll spin up a LIVE preview of your bot right here in the chat so you can see it working in seconds — then I'll hand you the full project as a one-click download to run on your own computer with your real docs and key. Ready?"*

Then proceed to Step 1.

---

### Step 1 — Pick the language, set the tone, detect your tools

**First, ask the user's preferred language — before anything else.** Use your harness's native multiple-choice question picker if you have one (see capability check below); otherwise ask in one plain-text line. Offer at least:

- **English**
- **العربية (Arabic)**

Then **conduct the entire rest of the session in the language they pick** — every question, confirmation, status line, and explanation. (Keep code, filenames, commands, and env-var names in English regardless; only the prose around them switches.)

**Interactive questions — detect, don't assume.** Before asking any *multiple-choice* question in this recipe, check your own tool list for an interactive question/picker. It may be named `AskUserQuestion` (Claude Code), `ask_user_input_v0` (claude.ai web), or something else depending on the harness. **If you have one, USE it for every multiple-choice question** — it gives the user clean tappable options instead of making them type. **If you have none, ask the same question in plain text.** *Free-text* questions (business name, example customer questions) are ALWAYS conversational — never force them into buttons, there's nothing to put on the options.

**Rendering check (do this too).** Note whether you can render an interactive preview/artifact inside the chat. claude.ai's Artifacts can render live React/HTML (including a chatbot that talks to Claude with no key). Some bare sandboxes can only produce files. If you're on Branch B and you CAN render, you'll use that in Step 6 to show the user a live demo of their bot. If you can't, you'll skip the preview gracefully.

Throughout this session:

- **No jargon without translation.** First time you say "API key," explain it in one sentence ("a password your code uses to talk to a service").
- **Verify visually after each step.** Don't say "done" without showing what changed: `✅ Node v22.21.1 detected`, `✅ folder created at ~/Desktop/myco-chatbot`, etc.
- **Never blame the user.** If something fails, calmly say "that's a known one, let me fix it" — even if it wasn't.

**Heads-up about the API key — say this NOW so the user can start the signup in another tab while you work, and so they don't sink time into something they can't finish:**

In the user's chosen language (Arabic or English), tell them clearly:

> *"One heads-up before we go any further: if you want a **deployable chatbot** (running on your computer, or live on the web) you'll need a free **OpenRouter API key** — that's the key your chatbot uses to talk to the AI. Signup at [openrouter.ai/keys](https://openrouter.ai/keys) takes about 30 seconds. **Grab one now in another browser tab so it's ready when we need it in a few minutes** — that way you're not stuck mid-build.*
>
> *If you only want to try it inside this chat (a quick playable demo), you can skip the key — but if you ever want to put this on your website or run it locally, you'll need it later. Worth grabbing now."*

**Branch-specific nuance** (don't read this verbatim, just keep it in mind):
- **Branch A (Claude Code)** — they will 100% need the key. Make this clear.
- **Branch B with artifacts (claude.ai chat / cowork)** — the artifact in Step 6a works WITHOUT a key (uses the in-chat Anthropic connection). So if they don't want the deployable download, they can skip the key entirely. Be honest about that.
- **Branch B without artifacts / Branch C** — they need the key for the downloadable project, no way around it.

Wait briefly for them to acknowledge (a thumbs-up, "got it", or "let me grab one"). If they say they'd rather not bother with a key and their branch can't fall back to artifact-only, say warmly: *"No problem — we can stop here, no hard feelings. Come back any time."* Don't try to talk them into it.

Once acknowledged, **move to Step 2.**

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

**Branch B/C:** Ask the user (picker if available, else plain text): macOS / Windows / Linux? You need this because the install commands in Step 4 — and the commands the user will run on their own machine after downloading — differ per OS. (Note: in a sandbox you can detect the *sandbox's* OS, but that's not the user's machine — so you still have to ask which computer they'll run it on.)

Tell the user what you detected so they know you're paying attention.

---

### Step 3 — Interview the user (the personalization)

Ask these **four questions**. Ask **Q1 and Q2 one at a time and conversationally** (they're free text — read the business answer before shaping how you ask for examples). You **may combine Q3 and Q4 into a single picker call** if your picker supports multiple questions at once (most do, up to ~3) — it feels snappier than two round-trips.

**Q1 (free text — ask conversationally): What's your business?**
- Get name + one sentence about what they do.
- Use this for the bot's system prompt and the folder name.

**Q2 (free text — ask conversationally): What are two example questions customers ask you that the bot should handle?**
- Don't accept vague answers — push for *real* examples ("when's the next available slot?", "do you ship to Riyadh?", etc.). These go in the bot's prompt as guidance.

**Q3 (picker if available, multi-select): What docs do you have for the bot to learn from?**
- PDFs (price lists, brochures, policies)
- Word docs
- Markdown / plain text notes
- Web pages (a list of URLs you'll paste)
- I don't have any yet — I'll add them later

**Q4 (picker if available, single-select): What's ONE action the bot should do besides answering?**
- Book an appointment
- Look up an order or customer
- Send an email or SMS
- Process a payment
- Check inventory / stock
- Other (you'll describe it)

**After all four, repeat back what you heard and confirm** (in the user's chosen language):

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

**Branch B (sandbox) and C (text):** the user will install these on *their own* machine after the download. Walk them through the same checks — give them links and the exact commands to type in Terminal/PowerShell — and wait for them to confirm each step before moving on. Don't dump everything at once. (The live preview in Step 6 needs none of this — it runs with zero setup — so the user gets the fun part before touching their terminal.)

---

### Step 5 — Get the OpenRouter API key

OpenRouter is the bot's brain. It's one signup that gives you access to Claude, GPT, Gemini, Llama — every major AI model. Free credits to start; pay-as-you-go after.

**Walk the user through:**

1. Open https://openrouter.ai in their browser → click "Sign in" (Google or email both work, 30 seconds)
2. After signing in, open https://openrouter.ai/keys → click "Create Key" → name it "my-chatbot" → copy the key (it starts with `sk-or-v1-...`)
3. **Important:** tell the user to keep this tab open or paste the key into a notes app — they'll need it in a minute, and OpenRouter doesn't show it again after you close the modal.

**Picker (any branch that has one):** ask "Do you have your OpenRouter key copied?" with options [Yes I have it / I'm signing up now / I need help]. Wait until they say yes. **No picker:** ask in plain text and wait for the user to say they have the key.

You will use this key in Step 7 to populate `.env.local`. **Never write it into committed code** — only into `.env.local` which is git-ignored. In a sandbox (Branch B), prefer to let the user paste the key into `.env.local` themselves after download rather than placing a real secret into a file you generate.

> **Note:** the **live preview** in Step 6 does NOT need this key — it talks to Claude through the chat's built-in connection. The OpenRouter key is only for the real downloadable project, which can use any model. So you can build and show the preview before the user has even signed up for OpenRouter.

---

### Step 6 — Build the chatbot

**Pick your flavor based on Step 0:**

| Your branch | Primary deliverable | Secondary (optional) |
|---|---|---|
| **A. Claude Code** | A real Next.js project on the user's disk | Deploy (Step 9) |
| **B. Chat/cowork WITH artifact rendering** | **A live artifact in the chat** (6a) — the bot itself | The downloadable Next.js project (6b), only if the user explicitly wants to deploy |
| **B. Sandbox WITHOUT artifact rendering** | The downloadable Next.js project (6b) | — |
| **C. Pure text** | Print each file for the user to save by hand (still uses the 6b file list) | — |

---

#### 6a — Branch B with artifacts: build the chatbot AS the artifact (THIS is the product)

**On claude.ai chat or cowork, the artifact isn't a preview — it's the deliverable.** Build it as a single self-contained React (or HTML) artifact the user can talk to right there in the conversation. Don't apologize for it. Don't call it a "demo" or "trailer." It's their chatbot.

Requirements:

- **Use the SAME system prompt** you'll bake from the interview answers — business name, one-sentence description, the two example questions, the chosen language. Set `dir="rtl"` + right-aligned bubbles if Arabic.
- **Talk to Claude directly through the in-chat connection — NO API key.** claude.ai artifacts can call the Anthropic endpoint and the platform supplies the key. Pattern:
  ```js
  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "claude-sonnet-4-20250514",
      max_tokens: 1000,
      system: SYSTEM_PROMPT,
      messages,
    }),
  });
  const reply = (await res.json()).content?.map(b => b.text || "").join("") ?? "";
  ```
  **Never put a real API key in an artifact** — artifacts are client-side and visible.
- **Implement the chosen action (Q4) as a believable in-artifact tool.** When the user asks something that triggers it, show a system line like `✅ Checked inventory — 42 units of <item> available`, then continue. The artifact can't reach their real CRM/Stripe/calendar, but the demonstrated behavior is exactly what the downloadable version will do for real.
- **Optional: allow the user to paste a small chunk of their business info** (services, prices, FAQs) into a `<textarea>` that gets appended to the system prompt at send time. That's a tiny in-artifact RAG that's usually enough to make the chatbot feel personal for the demo.
- **Match the channel look:** cream `#fdfdf7` background, Claude orange `#D97757` accent, rounded chat bubbles, mobile-friendly, Cairo font for Arabic.
- **Frame it honestly in chat — not as a teaser**: *"Here's your chatbot — try it! It speaks for <business> in the language you picked. For real deployment with your full documents and connected systems, ask me for the downloadable project."*

**Stop at 6a** unless the user asks for the deployable project. The artifact is their chatbot. Don't push the download.

---

#### 6b — Build the deployable Next.js project (Branch A always; Branch B on request; Branch C printed-by-hand)

**Branch A (Claude Code):** run these commands via Bash, in order. Stop on failure.

**Branch B sandbox-only (no artifact) OR Branch B artifact + user asked for deploy version:** run the scaffold inside your sandbox to produce a real, complete project the user can download — then create every file below with your write tool. Don't paste files for the user to copy by hand if you can write them directly. Strip `node_modules` and the build cache before zipping so the download stays small (the user regenerates with `npm install`).

**Branch C (pure text):** print each file in full as a code block for the user to save by hand in the same paths.

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

`create-next-app` will produce a fresh Next.js project (currently v16+, React 19). Wait for it to finish.

> **Two version gotchas (pin these to save yourself debugging):** the libraries below move fast. As of now, `npm` pulls **openai v6** (where `tool_calls` is a *union* type — narrow with `if (call.type !== "function") break;` before reading `call.function`) and **pdf-parse v2** (which dropped the old default function for a `PDFParse` class — use `const { PDFParse } = await import("pdf-parse"); const text = (await new PDFParse({ data: new Uint8Array(buf) }).getText()).text;`). Consider pinning `openai@^6` and `pdf-parse@^2` and writing the code for those, or pin to the older majors if you prefer the original snippets. Also: the default scaffold's `layout.tsx` imports a Google font at build time — if your build environment has no internet to Google Fonts, swap it for system fonts so the build doesn't fail.

Then create the chatbot files. Each file in full — no placeholders, no `// ...rest of code`:

**File 1: `src/lib/openrouter.ts`** — the LLM client
```ts
import OpenAI from "openai";

export const openrouter = new OpenAI({
  apiKey: process.env.OPENROUTER_API_KEY,
  baseURL: "https://openrouter.ai/api/v1",
  defaultHeaders: {
    "HTTP-Referer": "https://recipes.menawer.com/bot",
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

> **Tip:** reuse this exact `SYSTEM_PROMPT` string in the Step 6a live preview, so the demo and the real bot share one personality.

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
      const { PDFParse } = await import("pdf-parse"); // pdf-parse v2 API
      const buf = await fs.readFile(full);
      const parser = new PDFParse({ data: new Uint8Array(buf) });
      text = (await parser.getText()).text;
      await parser.destroy();
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

  // openai v6: tool_calls is a union — narrow to the function variant.
  while (resp.choices[0].finish_reason === "tool_calls") {
    const call = resp.choices[0].message.tool_calls![0];
    if (call.type !== "function") break;
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

> **Tip for Arabic-first businesses:** if the user picked Arabic, set `dir="rtl"` and `lang="ar"` on the `<main>` (or in `layout.tsx`), and flip the user-bubble alignment to the left, so the chat reads naturally right-to-left. (Do the same in the Step 6a preview.)

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
npm install -D tsx
```

(Branch C is already covered at the top of 6b — print each file as a code block in chat, tell the user to save them in the same paths, then dictate the install commands one by one.)

---

### Step 7 — Wire the key and run it

1. Copy `.env.local.example` → `.env.local`:
   ```bash
   cp .env.local.example .env.local
   ```
2. Open `.env.local` and paste the OpenRouter key after `OPENROUTER_API_KEY=`. **Branch A:** use Write to update `.env.local` directly with the key the user gave you. Confirm the line shows `OPENROUTER_API_KEY=sk-or-v1-...` (don't print the actual key back to the user; just confirm it's set). **Branch B (sandbox):** have the user paste their key into `.env.local` themselves after they download the project — don't bake a real secret into the files you generate.
3. Start the dev server (the user runs this on their own machine in Branch B/C):
   ```bash
   npm run dev
   ```
4. **Branch A:** watch the Bash output until you see `Ready in <Xms>` and `Local: http://localhost:3000`. Tell the user: *"✅ Your chatbot is running. Open http://localhost:3000 in your browser."* **Branch B/C:** tell the user to watch for the same lines in their own terminal and then open the URL. Remind them this is the full version of the bot they already met in the live preview — now with their real docs and key.
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
5. Have the user test a question that's actually answered in their docs — verify the bot uses the doc content. (This is the thing the live preview *couldn't* do — point that out so the upgrade feels real.)

---

### Step 9 — (Optional) Put it online

Ask the user (picker if available):

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

**If Cloudflare + Branch B/C** (no MCP available):
- Walk them through the Cloudflare dashboard manually: Workers & Pages → Create → Connect Git → pick the repo → Framework preset: Next.js → set env var → Deploy.

**If Vercel:** simpler — `npx vercel` and follow the prompts. They'll need to sign up at vercel.com first.

---

### Step 10 — Wrap up

Tell the user (in their chosen language):

- **Live preview:** (Branch B, if shown) "the bot you talked to up top was a quick demo — the version below is the real one."
- **Local URL:** http://localhost:3000
- **Live URL:** (if deployed) the `*.pages.dev` or `*.vercel.app`
- **To add tools:** "Come back and ask me — 'add a tool that sends a confirmation email to the customer' or 'add a tool that looks up an order by phone number.'"
- **To update docs:** Drop new files in `docs/`, run `npm run index`, restart.
- **To swap the LLM model:** Open `src/lib/openrouter.ts` and change the `MODEL` constant. Recommended swaps: `openai/gpt-4o-mini` (cheap GPT), `google/gemini-2.5-flash` (cheap Gemini), or `anthropic/claude-sonnet-4-7` (smarter Claude).
- **If anything breaks later:** paste the error in a new chat with this same recipe attached, and Claude will debug.

---

## Branch C (pure text, no tools) fallback — short summary

If you're a bare text chat with no sandbox and no picker, the flow above doesn't change conceptually — you just walk the user through every command yourself. You can't render a live preview, so don't promise one; be upfront that the see-it-live moment needs a tool like claude.ai or running it locally. Be EXTRA patient. Ask the language question in plain text first, then run the whole session in that language. Wait for the user to confirm each step. Print every file in full as a code block. Don't promise to "run" anything — you can't.

---

**End of instructions. Begin Step 0 now.**