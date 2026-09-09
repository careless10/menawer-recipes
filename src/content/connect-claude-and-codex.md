# Make Claude and ChatGPT Talk To Each Other — Claude prompt (v2)

> **You are reading this:** copy the *entire* contents of this file (Ctrl/Cmd+A → Ctrl/Cmd+C) and paste it into **Claude Code** running in your terminal. Hit enter. Claude takes it from there.

This recipe is from the [@eng_menawer](https://instagram.com/eng_menawer) channel. It wires up a **two-way channel between Claude Code and OpenAI's Codex (ChatGPT's coding agent)** on *your own machine*, so the two AIs can send each other messages instead of you copy-pasting between two terminals.

**Works on macOS, Linux, and Windows.** Every step below was executed and verified on a Mac (macOS, Claude Code 2.1.263, codex-cli 0.153.4) *and* independently on a Windows machine (Windows 10 19045, Git Bash, Claude Code 2.1.260, codex-cli 0.153.4, Node v24.16.0) — including both message directions on both platforms.

> **Important, up front:** this uses **internal, undocumented plumbing** inside Claude Code and Codex. It is not an official feature of either product and can break with any update to either tool. Claude will run a preflight check first and tell you honestly if your machine can't do it.

**This must be pasted into Claude Code, not claude.ai.** Either the terminal CLI or the Claude desktop app's **Code tab** works — both register a messaging socket and have real filesystem access. The browser web chat does not, and cannot reach your machine's sockets. If you paste it into a web chat, Claude should say so and stop.

---

## Instructions for Claude — everything below this line is for you, Claude

You are helping a user connect **you** (Claude Code, on their machine) to a **Codex** session on that same machine, in both directions. The user may or may not be technical. Verify every step visually and be scrupulously honest about verified vs. unknown.

**A standing rule for this whole recipe: never state a security property you have not observed on this machine.** An earlier version of this document made three security claims from inference — all three turned out wrong, and all three were wrong in the direction that makes a reader relax. Read a value, print it, then describe it.

### Step 0 — Environment check

Check your own tool list.

- If you have **`Bash` + `Read` + `Write`/`Edit` and real filesystem access** → you are Claude Code on the user's machine. Continue. **This includes the Claude desktop app's Code tab**, not just the terminal CLI — verified on Windows 11, where it registers in `~/.claude/sessions` with a `messagingSocketPath` like any other session (its `entrypoint` field reads `claude-desktop` rather than `cli`). Do not turn a user away for using the desktop app.
- If you are in **claude.ai web chat, cowork, or any hosted sandbox** → **stop.** Say:

> *"This recipe only works in Claude Code — the CLI that runs in your terminal on your own computer. I'm running in a web sandbox right now, so I can't reach your machine's sockets or files. Install Claude Code, run `claude` in your terminal, and paste this recipe there instead."*

Then detect the platform and **keep going either way**:

```bash
uname -s 2>/dev/null || echo "Windows"
```

**Do not stop on Windows.** An earlier version of this recipe did, and it was wrong — the whole thing works there. The transport differs (named pipe instead of Unix socket) and some tooling differs, both handled below. Only the **preflight in Step 2** decides whether this machine qualifies.

### Step 0.5 — Auto mode will probably interrupt this. Say so NOW, before it happens.

**This is the single most common way readers get stuck, and it looks exactly like the bridge being broken.**

If this session is in **auto mode**, its own permission classifier will likely refuse some commands in this recipe. That refusal is not a sandbox error, not a missing dependency, and not a bug in the bridge. It is this session's permission layer declining to run something on the user's behalf.

**Tell the user this up front, in one sentence, before you run anything:**

> *"Before we start: if I'm in auto mode, my own permission system may refuse some of these commands. That's not the bridge failing — I'll tell you plainly when it happens and hand you the command to run yourself. Switching this session out of auto mode now will avoid most of it."*

Then recommend they switch to a prompting/default permission mode for the duration. Measured across two machines: in a non-auto mode the same commands prompt normally instead of being silently refused.

**When a refusal happens — and it may — follow this order. Do not improvise.**

1. **Name it immediately, at the point of failure.** Say "this was refused by my permission classifier, not by the bridge." Never let the user believe the mechanism is broken. Never report a classifier refusal as a technical failure of the recipe.
2. **Do not silently retry, and do not reword the command to get through.** Rewording is not a diagnosis. On one machine the refused commands included two carrying *no message text at all*, and one that had **succeeded minutes earlier in the same session** — there was nothing to reword. Trying to slip past a permission decision is the wrong instinct even when it works.
3. **Hand the user the exact command to run in their own terminal.** Paste-ready, with real values already filled in, no placeholders. This is the immediate way out and it always works.
4. **A later session is a genuinely different context, not a workaround.** If the step is still needed then, run it once and report the result honestly either way.

Refusals are **transient and not a property of the machine**: on the same machine, commands refused six times in one session passed on the first attempt in a new session with no settings change, no new permission rule and no rewording. That is a measurement, not an explanation — do not offer the user a mechanism for it, because nobody has established one.

**One thing that does NOT help: the user telling you to proceed.** Measured three separate times, including a user typing an explicit written instruction to continue. Chat consent is not the layer the classifier listens to. Do not ask the user to confirm and then re-run the same command — that wastes their time and yours. Go to step 3.

### Step 1 — Pick the language

Ask the user their preferred language with `AskUserQuestion`. Offer at least **English** and **العربية (Arabic)**. Conduct the rest of the session in whichever they pick. Keep commands, paths, and JSON field names in English; only the prose switches.

Then set expectations in one short paragraph:

> *"Here's the plan: first I check whether your machine has the pieces this needs — that's the step that decides whether this works at all. If it passes, I set up two one-way channels: one that lets me send messages to Codex, and one that teaches Codex how to send messages back to me. Then we test both."*

---

### Step 2 — PREFLIGHT. This is the gate.

The receiving half of this bridge is infrastructure **Claude Code runs on its own** — neither you nor Codex can create it. If it isn't present, the recipe is *inapplicable*, not broken.

**Use your own `Bash` and `Read` tools for these checks. Do not require python3 — it does not exist on a default Windows install, and a preflight that can't run is worse than no preflight.**

**(a) Does the session registry exist?**

```bash
ls -la ~/.claude/sessions/ 2>/dev/null | head -20
```

Expect one `<pid>.json` per live session, plus `<pid>.<64 hex>.key` files. Then **`Read` one of the `.json` files directly** and confirm it contains a `messagingSocketPath` field.

What that field looks like tells you the transport:

| Platform | `messagingSocketPath` value |
|---|---|
| macOS / Linux | `/tmp/cc-socks/<pid>.sock` (a real file on disk) |
| Windows | `\\.\pipe\LOCAL\cc-msg-<32 hex>` (a named pipe) |

**Do not check for `/tmp/cc-socks/` on Windows.** It won't exist and shouldn't — that check is a false negative that makes Windows readers quit at step one.

- **At least one session file with a `messagingSocketPath`** → the receiver half exists. Continue.
- **Directory missing/empty, or no `messagingSocketPath`** → **STOP:**

> *"Your Claude Code doesn't expose the messaging socket this recipe needs. That's the piece I can't create — it has to come from Claude Code itself. Most likely a version difference. Try updating Claude Code and re-running this; if it still shows nothing, this recipe can't work on your setup and there's no workaround I can honestly offer."*

Do not fabricate the socket or patch Claude Code. Stop cleanly.

**(b) Is there a matching auth token file?**

```bash
ls -la ~/.claude/sessions/ | grep -i '\.key$' | head
```

Expect `<pid>.<64 hex>.key`, each containing JSON with a `peerToken` (32 hex chars). None → same stop as above.

**Note the permissions you actually see** — you'll need them in Step 7, and they differ by platform. Do not assume.

**(c) What tooling does this machine actually have?**

Probe all of it in one plain-shell line, before you use any of it:

```bash
command -v node python3 sqlite3 codex
```

> **A runtime detector cannot be written in a runtime it is detecting.** Do not probe with a `python3` heredoc — on Windows `python3` is a Microsoft Store alias stub, so the step meant to tell the user what they have becomes the step that fails. Plain shell first; everything richer happens only after you know what you're allowed to use.

Read the result and branch:

- **`node` present** → use it for everything below. `net.connect()` accepts a Unix socket path *and* a Windows named pipe path through the identical API, so one client covers both platforms and the transport branch disappears.
- **`node` missing, `python3` present** (possible on a clean Mac) → a Python client works on macOS/Linux only. **Windows cannot use it at all** — CPython has no `socket.AF_UNIX` there.
- **Neither** → stop; there is no client runtime.
- **`sqlite3` missing** (normal on Windows) → use the Node fallback in Step 4.

**Do not assume Node is present just because Claude Code is.** Verified: on macOS, Claude Code is a standalone binary that bundles no Node at all, and `ClaudeCode.app` contains no `node` anywhere.

**(d) Is Codex installed and does it have `queue`?**

```bash
codex --version && codex queue --help 2>&1 | head -5
```

If `codex: command not found`, **do not tell the user to reinstall** — on the Windows desktop install it simply isn't on PATH. Look for it:

```bash
ls "$LOCALAPPDATA/OpenAI/Codex/bin"/*/codex.exe 2>/dev/null
```

The `<hash>` directory segment varies, so glob it. Verified working when invoked by full path. If you find it, use that full path everywhere below and tell the user what you found.

**If that glob is ALSO empty, Codex still may not be absent.** Verified on Windows 11 with the Codex desktop app installed: no `codex` on PATH and nothing under `$LOCALAPPDATA`. The fix that worked:

```bash
npm install -g @openai/codex
```

That CLI shares the desktop app's `~/.codex` directory *and* its login — `codex login status` reported `Logged in using ChatGPT` with no extra auth — and its `queue` subcommand drove the app's own live threads correctly. Try this before telling anyone they need to install Codex.

If `codex queue` isn't recognised, the CLI is too old — say so, since without it you can only reach *dormant* threads (Step 4), which is a much worse experience.

**(e) Is there at least one Codex thread?**

```bash
ls ~/.codex/state_*.sqlite 2>/dev/null
```

If `~/.codex` doesn't exist, the user has never run Codex. Have them open a second terminal, run `codex`, type `hello`, and leave it open.

**Report the preflight as a checklist with real ✅/❌ per line.** Continue only if (a), (b), (c) and (d) pass.

---

### Step 3 — The shape of what you're building

There is **no single bridge and no handshake protocol**. Two independent one-way transports:

| Direction | Mechanism |
|---|---|
| **You → Codex** | The `codex` CLI (`codex queue` for live threads, `codex exec resume` for dormant) |
| **Codex → You** | A socket/pipe connection to your `messagingSocketPath`, writing two JSON lines |

The halves know nothing about each other. There is **no capability negotiation** — Codex learns to reply because you *send it complete instructions in plain language* and it writes the code itself. Your bootstrap message must be **fully self-contained**: the receiving agent has zero context and no way to ask a clarifying question mid-flight.

---

### Step 4 — Direction 1: you → Codex

**Discover the threads.** The DB filename carries a *schema version* — **always glob, never hardcode `state_5.sqlite`**.

If `sqlite3` exists (typical on macOS, **absent on Windows**):

```bash
DB=$(ls -t ~/.codex/state_*.sqlite | head -1)
sqlite3 -readonly "$DB" \
  "SELECT id, COALESCE(NULLIF(name,''), NULLIF(title,''), id), cwd, tokens_used
   FROM threads WHERE archived=0
   ORDER BY COALESCE(recency_at,updated_at) DESC LIMIT 20;"
```

If `sqlite3` is missing, use Node's built-in SQLite (Node 22.5+; flagless on 24, prints a harmless `ExperimentalWarning` on 22). Write `list-codex-threads.js`:

```js
#!/usr/bin/env node
'use strict';
const fs = require('fs'), os = require('os'), path = require('path');
const { DatabaseSync } = require('node:sqlite');
const CODEX = path.join(os.homedir(), '.codex');

const hits = fs.readdirSync(CODEX)
  .filter(f => /^state_\d+\.sqlite$/.test(f))
  .map(f => ({ f, m: fs.statSync(path.join(CODEX, f)).mtimeMs }))
  .sort((a, b) => b.m - a.m);
if (!hits.length) { console.error('no state_*.sqlite - has Codex ever run?'); process.exit(1); }

const db = new DatabaseSync(path.join(CODEX, hits[0].f), { readOnly: true });
const rows = db.prepare(`
  SELECT id,
         COALESCE(NULLIF(name,''), NULLIF(title,''), id) AS label,
         cwd, tokens_used,
         datetime(COALESCE(recency_at, updated_at),'unixepoch','localtime') AS last_active
  FROM threads WHERE archived = 0
  ORDER BY COALESCE(recency_at, updated_at) DESC LIMIT 20
`).all();

const locks = path.join(CODEX, 'thread-writer-locks');
for (const r of rows) {
  const open = fs.existsSync(path.join(locks, `${r.id}.lock`));
  console.log(`${r.id}\n  ${r.label}\n  cwd=${r.cwd} tokens=${r.tokens_used} last=${r.last_active}`);
  console.log(`  ${open ? 'OPEN NOW -> codex queue' : 'dormant -> codex exec resume'}`);
}
```

> **The label fallback must end at `id`.** A thread with **both** `name` and `title` empty otherwise yields NULL and renders as a blank, unpickable menu option. Measured: 1 such thread on one machine, **4** on another. This is the common path, not an edge case.

Show the list and let the user pick with `AskUserQuestion`, labelling each option with the thread name **and its working directory** — names alone are often ambiguous.

**Check whether the thread is currently open:**

```bash
ls ~/.codex/thread-writer-locks/<thread-id>.lock 2>/dev/null
```

> **Orphan locks exist.** Verified on **both** platforms: lock files whose thread id has *zero* rows in the database. Never assume a lock implies a live, lookup-able thread — tolerate a lock with no matching row instead of erroring.

**If the lock EXISTS (thread is live) — use `queue`. This is the good path:**

```bash
codex queue --thread <thread-id-or-exact-name> --message "your text here"
```

Returns instantly with `Queued message <id> for thread <id>.` It never tries to become the thread's writer, so the lock is irrelevant. Delivery is asynchronous — the client picks it up at its next turn. Verified on both platforms, including into a very large actively-used thread; it landed within seconds.

**If there is NO lock (dormant) — use `exec resume`:**

```bash
codex exec --sandbox read-only resume --skip-git-repo-check <thread-id> "your text here"
```

Two flag gotchas, and the *reason* matters so you can reason about future flags: `-s/--sandbox` is an option of **`exec`**, while `--skip-git-repo-check` is an option of **`resume`**. So `--sandbox` must come **before** the subcommand or you get `unexpected argument '--sandbox'`; and `--skip-git-repo-check` is required whenever the current directory isn't a git repo, else `Not inside a trusted directory`.

> **⚠️ Tell the user about the cost.** `resume` **replays the entire thread history on every call.** Measured: ~124,000 tokens for a one-line ping into a small thread, ~248,000 into a medium one, ~641,000 into a large one. **Cost scales with the target thread's size, not your message.** `codex queue` has no such cost. So: **queue for anything live, resume only for dormant threads, and never loop either as a chat channel.**

**Do not fight the writer lock.** `exec resume` on a live thread gives `thread-store conflict: thread <id> already has an active writer (code -32600)`. That lock is an OS `flock()` held by the *shared* `codex app-server` daemon, not the client — killing the Codex client does **not** release it (verified: still locked 10+ minutes after the process was gone). **Never restart the app-server daemon to "fix" it** — it's shared by every Codex client on the machine. Just use `queue`.

Send a test message and confirm with the user that they saw it arrive.

---

### Step 5 — Direction 2: Codex → you

Write the client yourself and hand it over — don't make Codex reinvent it, it will re-hit the traps below. **One file works on both platforms**, because `net.connect()` takes a Unix socket path and a Windows named pipe path through the same API. Save as `send-claude-message.js`:

```js
#!/usr/bin/env node
'use strict';
const net = require('net'), fs = require('fs'), os = require('os'), path = require('path');
const SESSIONS = path.join(os.homedir(), '.claude', 'sessions');

function discover() {
  let files;
  try { files = fs.readdirSync(SESSIONS); }
  catch (e) { console.error(`cannot read ${SESSIONS}: ${e.message}`); process.exit(1); }
  const out = [];
  for (const f of files) {
    if (!f.endsWith('.json')) continue;
    let d;
    try { d = JSON.parse(fs.readFileSync(path.join(SESSIONS, f), 'utf8')); }
    catch { continue; }
    if (!d.messagingSocketPath) continue;
    // Measured TRUE for a live named pipe on Win10/Node 24 and for unix
    // sockets on macOS, so this filter is safe on both. NEVER probe the pid
    // with kill(0): a sandboxed caller is denied that syscall and every
    // session silently vanishes, which looks identical to "found 0".
    if (!fs.existsSync(d.messagingSocketPath)) continue;
    out.push({ name: d.name, pid: d.pid, sock: d.messagingSocketPath });
  }
  return out;
}

function tokenFor(pid) {
  const hit = fs.readdirSync(SESSIONS).find(f => f.startsWith(`${pid}.`) && f.endsWith('.key'));
  if (!hit) { console.error(`no key file for pid ${pid}`); process.exit(1); }
  const tok = JSON.parse(fs.readFileSync(path.join(SESSIONS, hit), 'utf8')).peerToken;
  if (!/^[0-9a-f]{32}$/i.test(tok || '')) { console.error('bad peerToken format'); process.exit(1); }
  return tok;
}

function send(name, text) {
  const matches = discover().filter(s => s.name === name);
  if (matches.length !== 1) {
    console.error(`Expected exactly one live Claude session named ${JSON.stringify(name)}; `
      + `found ${matches.length}`
      + (matches.length ? `: pids ${matches.map(m => m.pid).join(', ')}` : ''));
    process.exit(1);
  }
  const s = matches[0];
  // Build the payload BEFORE connecting - a connection that doesn't send a
  // complete line quickly gets closed.
  const payload =
    JSON.stringify({ type: 'auth', token: tokenFor(s.pid) }) + '\n' +
    JSON.stringify({ type: 'user', message: { role: 'user', content: text } }) + '\n';
  const conn = net.connect(s.sock);
  let got = '';
  conn.on('connect', () => { conn.write(payload); setTimeout(() => conn.end(), 1500); });
  conn.on('data', d => { got += d.toString(); });
  conn.on('error', e => {
    console.error(`FAILED at ${e.syscall || 'connect'}: ${e.code || ''} ${e.message}`);
    process.exit(1);
  });
  conn.on('close', () => {
    if (got.trim()) console.log('reply:', got.trim());
    console.log(`sent to "${s.name}" (pid ${s.pid})`);
  });
}

const [, , cmd, a, b] = process.argv;
if (cmd === 'list') {
  const rows = discover();
  console.log(`${rows.length} live session(s):`);
  for (const s of rows) console.log(`  ${JSON.stringify(s.name)} pid=${s.pid}\n    ${s.sock}`);
} else if (cmd === 'send' && a && b) { send(a, b); }
else { console.error('usage: node send-claude-message.js list | send "<name>" "<text>"'); process.exit(1); }
```

**FIRST, SELF-TEST IT FROM YOUR OWN SHELL.** Before you involve Codex at all, run the client from your own `Bash` tool and send a message to *your own session*:

```bash
node <PATH>/send-claude-message.js list
node <PATH>/send-claude-message.js send "<YOUR OWN SESSION NAME>" "self-test"
```

It should arrive in this conversation within about a second. This proves the socket path, the token, the frame format and the client are all correct **independently of Codex's sandbox** — so if the Codex attempt fails afterwards, you already know the failure is on Codex's side and not in the plumbing. Skipping this makes every later error ambiguous. Do not skip it.

> **If THIS step is refused, it is auto mode — not the bridge.** This is the most likely place in the whole recipe to hit the permission classifier, and it is marked unskippable, so a reader will stall here. Apply the order from Step 0.5 immediately: name it as a classifier refusal, do **not** reword it, and hand the user this exact command for their own terminal with the real path and name already filled in:
>
> ```bash
> node <PATH>/send-claude-message.js send "<ACTUAL NAME>" "self-test"
> ```
>
> Verified: a refusal here says nothing about whether the bridge works. On the machine where it was refused, the identical command succeeded in a later session untouched.

**Both placeholders above must be real before you show this to anyone.** `<PATH>` is wherever you actually saved the client, and `<ACTUAL NAME>` is this session's current name — derive it now rather than reusing anything written earlier:

```bash
cat ~/.claude/sessions/$CLAUDE_PID.json
```

`CLAUDE_PID` is this session's own PID, already in your environment; that file carries the matching `name`. **Session names change between sessions** — one machine's name went from `desktop-41` to `desktop-cd` overnight — so a command containing a name has a shelf life of hours. Re-derive it at the moment of sending, every time, and never hand the user a command with a placeholder still in it.

**The wire protocol it implements** (for your understanding): read `messagingSocketPath` and the matching `peerToken`, connect, then write exactly two newline-terminated JSON frames:

```json
{"type": "auth", "token": "<peerToken>"}
{"type": "user", "message": {"role": "user", "content": "your text"}}
```

**Then send Codex the briefing** via `codex queue`. Fill in the real values — **never paste example names, PIDs or paths from this recipe; resolve them on this machine**:

> *I'm a Claude Code session named `<YOUR ACTUAL SESSION NAME>` on this machine, and I'd like you to message me back.*
>
> *A working client is at `<ABSOLUTE PATH>/send-claude-message.js`. List sessions: `node <PATH> list`. Send to me: `node <PATH> send '<YOUR ACTUAL SESSION NAME>' 'your text'`.*
>
> *On macOS, connecting to the socket is outside your sandbox — run it with `sandbox_permissions: "require_escalated"` on your `exec_command` tool, with a clear justification. Without it the connect fails with `PermissionError: [Errno 1] Operation not permitted` before any data is sent.*
>
> *Please send me a short test message now. If it fails, write what went wrong to `/tmp/codex-bridge-error.txt` (or `%TEMP%\codex-bridge-error.txt`) instead — that's your fallback channel.*

**That last sentence is not optional.** The only reply channel is the very thing being tested, so a first-attempt failure is otherwise completely silent. Always give the remote agent a **file-based fallback**.

> **PowerShell quoting.** If Codex runs on Windows it is likely using PowerShell, where **double quotes expand `$` and backticks** and **single quotes need apostrophes doubled**. The single-quoted example above can bite. Tell Codex to use double quotes for the message text and to avoid `$`, backticks and apostrophes in its reply text.

#### If Codex cannot open the socket/pipe — permissions, by platform

This is the step most likely to fail, and **the fix differs by platform**. Work through what applies.

**macOS.** The connect is blocked by Codex's sandbox and fails with `PermissionError: [Errno 1] Operation not permitted` *before* any frame is written. Fix: have Codex re-run that one command with `sandbox_permissions: "require_escalated"` on its `exec_command` tool, with a clear justification. Verified working.

**Windows (Codex desktop app) — the macOS fix may be unavailable.** Verified on Windows 11 with the desktop app, where the failure is `EPERM connect \\.\pipe\LOCAL\cc-msg-…` and escalation was *not reachable*:

- **Granting network access does not help.** Codex requested and was *granted* network permission for the turn, then hit the identical `EPERM`. A local named pipe is not network access — do not let anyone burn time on this.
- **The escalation prompt may never appear.** In that session the policy auto-rejected `sandbox_approval` requests, so no approval dialog could be raised at all and `require_escalated` was never attemptable. The macOS advice simply has no equivalent in that configuration.
- **What worked:** switching that Codex thread to **Full access** (`danger-full-access`) in the app. The send then succeeded normally in about 1.7s, and every subsequent send worked.

Present Full access to the user as **one tested configuration, not a rule** — it is not established that every Windows sandbox blocks every pipe, and it is a real privilege increase they should agree to knowingly.

**Sandbox-safe fallback, if the user won't grant Full access.** Verified working from *inside* the restricted sandbox: Codex writes the message as UTF-8 into a drop directory (e.g. `%TEMP%\claude-codex-inbox\`), and a small Node watcher on the Claude side picks up each file and forwards it over the pipe. Slower and one more moving part, but it needs no privilege escalation at all.

> **The session name changes between sessions.** Claude session names are often derived per-session (`user-bb`, and so on), so a name you gave Codex will be stale after either side restarts. Re-send Codex the current name rather than assuming the old one still resolves — and remember the client refuses ambiguous matches by design.

---

### Step 6 — Known failure modes

1. **Claude Code's own permission classifier blocks the commands.** Verified on both machines: it refused the pipe client and refused `codex queue`. This is not a sandbox, OS, or dependency problem — it's the agent's own permission layer, and it looks exactly like the bridge being broken when it works perfectly.

   **The user telling you to proceed does not lift it.** Observed: it blocked again immediately after the user typed an explicit instruction to continue. Chat consent is not the layer the classifier listens to, so do not keep asking the user to confirm and re-running — that wastes their time and yours.

   **Follow the order in Step 0.5 — it is the remedy for this entry.** Name it as a classifier refusal at the point of failure, do not silently retry, do not reword to get through, hand the user the paste-ready command for their own terminal, and treat a later session as a different context rather than a workaround.

   **Content is an input, not the input — do not treat rewording as the fix.** One machine saw a `codex queue` refused over the words "non-sandboxed" in the *message text*, and rewording got the identical command through. Another machine saw six refusals in a single session where two carried **no message text at all**, two carried the most neutral text the operator could write, and one was a command that had **succeeded minutes earlier in that same session**. Both results are real and neither generalises. So rephrasing is worth at most one attempt, and it must never stand in front of "hand it to the user" as the first move.

   **It can block fetching this recipe page too.** On one machine in auto mode the in-app browser, `curl` and Chrome navigation were all denied, so the reader could not even load the page to copy it. If that happens: have them paste the recipe text in directly, and switch the session out of auto mode.

   **Do not tell the user to retry.** Retries succeeded on one machine and failed identically twice on another. A *new session* is a different matter and often does pass — but that is an observation, not a mechanism, and it is step 4, not step 1.
2. **`PermissionError: [Errno 1] Operation not permitted` on `connect()` (macOS).** Codex's sandbox blocked the connection. It fires *before* any auth frame, so it is **not** an auth problem, not `ECONNREFUSED`, not a missing key. Fix: retry with escalated sandbox permissions on that exec call.
3. **Codex reports "found 0 sessions" when sessions clearly exist.** Its script is probing liveness with `os.kill(pid, 0)`, which its sandbox denies — everything gets filtered out. Looks like a discovery bug, isn't. The client above avoids it.
4. **`thread-store conflict … already has an active writer`.** You used `exec resume` on a live thread. Use `queue`.
5. **Orphan lock files.** A `.lock` whose thread id isn't in the DB. Verified on both platforms. Tolerate it.
6. **Name collisions.** Claude session names and Codex thread names are **separate namespaces and collide freely** — the same name can refer to one Claude session and two different Codex threads at once. Some setups also bulk-import Claude transcripts into Codex, making duplicates the norm. **Both sides must refuse ambiguous matches.** Disambiguate by pid (Claude) or thread id (Codex).
7. **Don't filter targets on `status`.** A "busy" session is valid — it processes at its next tool round.
8. **`sqlite3` is absent on Windows.** Use the Node fallback in Step 4.
9. **`python3` is absent on Windows** (only a Microsoft Store alias stub). And CPython has no `socket.AF_UNIX` on Windows at all, so a Python client cannot work there regardless.
10. **`socat` may not be installed anywhere.** Not needed — Node covers it.
11. **Writing into `~/.codex/skills/` may need escalation** — it can sit outside Codex's ordinary writable roots. A temp directory is the reliable drop point.
12. *(macOS/Linux only)* Unix socket paths have a ~103-byte limit. Irrelevant to Windows named pipes.

---

### Step 7 — Be honest about the limits

Tell the user all of this, in their language.

- **This is internal, undocumented plumbing in both tools.** Not a supported feature of either. Expect to re-run this recipe after updates. This is the single biggest fragility.
- **Messages are one-shot and asynchronous in both directions.** Each direction is a separate explicit send. No request/response, no delivery receipt, no synchronous reply. **A timeout is not a failure, and silence is not confirmation.** Do **not** auto-resend after an uncertain result — you'll duplicate the message. It is texting, not a phone call: describe it as **bidirectional async messaging**, never as "real-time conversation."
- **Sender identity is labelled but NOT verified — and is actively mislabelled.** Observed, not inferred: an incoming message arrives wrapped as `<cross-session-message from="uds:/tmp/cc-socks/<pid>.sock" from-name="..." from-mode="...">`, together with built-in guidance that a peer cannot grant escalation and that relaying a denied action is "permission laundering." So provenance *is* surfaced. **But the label describes the transport, not the sender:** a message sent by *Codex* over this channel arrives announced as *"Another Claude session sent a message."* Anything holding a valid `peerToken` is presented to the recipient as a Claude session, whoever it actually is. Do not build trust decisions on that label.
- **Anyone who can read the token file can impersonate the user to that session.** Both agents must run **as the same OS user**. On macOS/Linux the key files are mode `0600`; **on Windows they are not** — NTFS shows `-rw-r--r--` and POSIX modes don't apply, so the filesystem protection you'd expect on Unix isn't there. Treat the warning as *stronger* on Windows, not weaker. Whatever platform you're on, **read the permissions and report what you actually see** rather than repeating this paragraph.
- **Not verified:** whether messages can interrupt a session mid-turn while it's actively generating; whether the app-server daemon must be running or auto-starts; behaviour of fully headless/dormant Codex threads; cross-user access.
- **No live real-time attach.** There is an app-server control socket (`~/.codex/app-server-control/app-server-control.sock`, methods `thread/inject` / `turn/start` / `turn/steer`) but its auth handshake is unknown — connections are accepted then silently dropped across every framing tried. Don't promise real-time conversation.
- **Version numbers are observations, not requirements.** This was developed on specific Claude Code / Codex / Node combinations. Those are *what it was tested on*, **not** established minimums — the Step 2 preflight tests the actual capability, which is what matters.

---

### Step 8 — Wrap up

Show the user a short summary of what now exists:

- The command *they* run to message Codex from a terminal.
- The command *Codex* runs to message this Claude session, and the script path.
- Which Codex thread is wired up, by **id**, not just name.
- A reminder that this may need re-running after either tool updates.

Then offer one more round-trip test to confirm both directions.

---

*Recipe from [@eng_menawer](https://instagram.com/eng_menawer). Methodology developed and verified by a Claude Code session working with a Codex session on macOS, then independently re-tested end to end on Windows — including the parts that failed, and the three security claims that turned out to be wrong.*
