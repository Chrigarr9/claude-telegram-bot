# SP1 — Telegram-on-pi Design

| | |
|---|---|
| **Status** | Draft v2 — open questions resolved 2026-05-13; pi API verified |
| **Date** | 2026-05-12 (rev 2026-05-13) |
| **Working name** | `paddelino` |
| **Author** | Chrigarr (with Claude in brainstorming mode) |
| **Supersedes** | n/a |
| **Related** | [`docs/phase2-agent-comparison.md`](../../phase2-agent-comparison.md); pi-research clone at `/mnt/Shared/Code/projects/pi` |

## 1. Context

The existing `claude-telegram-bot` (~13k lines TypeScript on Bun) is a Telegram-native operator console for Claude Code, with polished UX around streaming, voice in/out, multi-project sessions, group routing, table rendering, file fallbacks, and long-running task notifications. Its runtime is tightly coupled to the Anthropic Claude Agent SDK.

The user's goal is broader than a coding bot: a **home assistant** that the household interacts with through Telegram, a local touch display on a 2016 Surface Book (running Linux), and voice; that has per-user memory; that can extend itself at runtime with new skills and new display views; and that ships as **one installable package** runnable on any Linux machine. The home-assistant goal does not eliminate coding tasks — coding becomes a deferred *skill* the assistant can dispatch.

`docs/phase2-agent-comparison.md` already analyses the runtime/transport split problem and recommends a conservative path (keep Telegram-only, make the runtime abstraction real, add policy levels, defer multi-channel). The current document goes one step further than that doc: it commits to **replacing the Claude Agent SDK runtime with [pi-agent-core](https://github.com/earendil-works/pi)** (TypeScript, multi-provider, multi-channel-friendly), and decomposes the larger vision into independently shippable sub-projects.

## 2. Scope decomposition (whole vision)

The full vision decomposes into five sub-projects. **This document specifies SP1 only.** The others are listed so readers can see the trajectory and so seams in SP1 are designed forward-compatibly.

| # | Sub-project | What ships | Depends on |
|---|---|---|---|
| **SP1** | **Telegram-on-pi** *(this document)* | New repo `paddelino`. Single Bun process. pi-agent-core as the runtime. Telegram channel via grammY. Per-user pi `Agent` instance, persisted to disk. Built-in `text` and `photo` input processors. Trivial Router. Two stub tools. Single installable package. | nothing |
| SP2 | Local display channel | Chromium kiosk on the Surface Book pointing at a local HTTP+WSS endpoint served by the same `paddelino` process. Uses `pi-web-ui` (`ChatPanel`, `ArtifactsPanel`, IndexedDB storage). | SP1 |
| SP3 | Memory system | Per-user persistent memory store (file-based, `git`-tracked for traceability), surfaced via a `memory` tool and via `transformContext` for retrieval injection. Replaces SP1's stub `note` tool. | SP1 |
| SP4 | Voice channel | Wake word + STT + TTS. Optional native deps detected at runtime (off by default; core install runs without them). Visual feedback via SP2. | SP1, SP2 |
| SP5 | Self-extension loop | Skill registry, hot reload, sandboxed/policy-gated execution, agent-authored skills with git-tracked versioning and a known-good baseline. Input processors and Router rules also become skill-installable. Sub-agents (delegated task agents) live here. | SP1, SP2, SP3 |

Coding-via-Claude-Code becomes one possible skill in SP5; it is **not** an SP1 deliverable. pi handles coding tasks directly without it.

## 3. Goals and non-goals (SP1)

### Goals

- Replace the Claude Agent SDK runtime with pi-agent-core, end-to-end, on the Telegram channel.
- Multi-provider out of the box (pi-ai supports 20+ providers; user picks via env vars).
- Per-user conversation continuity across restarts (Context persisted as JSON).
- Streaming responses with consolidated status updates (port the current bot's pattern).
- One installable package that runs on any Linux machine: `install.sh` → `paddelino` on `$PATH`.
- Forward-compatible seams (InputProcessor, Router, identity, beforeToolCall policy, afterToolCall audit) so SP2–SP5 plug in without refactoring SP1.

### Non-goals (explicit OUT list)

- Browser display / local HTTP+WSS endpoint (SP2).
- Voice OUTPUT (TTS); will become a "conversational output" skill in SP5 that also adjusts response style.
- Voice INPUT, audio file, PDF, document, video, video-note handling (all SP5 input-processor skills). Polite stub replies in SP1.
- `/raw` file forwarding (SP5; needs a file-access skill).
- URL preprocessing (SP5).
- Real memory system; SP1 ships a stub `note` tool that writes to flat files.
- Skill registry / extensions / self-modification (SP5).
- Sub-agents / delegated task agents (SP5).
- Group chats, multi-project routing, long-running process monitor (artifacts of the bot's coding-tool history; revisit when there's a concrete use case).
- Telemetry, automatic updates, Docker images.
- Multi-host sync.
- Cost limits / per-user spend caps. **Allowlist is the trust boundary in SP1** — a compromised allowlisted account or a runaway tool loop can rack up real provider costs.

## 4. Core architectural decisions

### 4.1 One assistant, per-user pi `Agent`, global skills

"The assistant" is a *shared identity*: a system prompt + a set of skills + a set of tools. **There is only one assistant in the household.**

Inside the process, **each user has their own pi `Agent` instance** for conversational state continuity and privacy. All Agents share the same identity (system prompt, tool registry, skill set). Memory will be per-user (SP3). Sub-agents (delegated task agents) are an SP5 concept and orthogonal to user-Agent.

Rationale:
- Privacy: messages from Alice are not in Bob's context window when Bob talks.
- Context-window scale: each user's history grows independently and slowly.
- pi's continuity primitives (cross-provider handoffs, steering, follow-ups, `transformContext`) assume a single thread per Agent.

### 4.2 Single process, single installable package

One Bun process. pi-agent-core runs in-process. Telegram (and later SP2's HTTP+WSS, SP4's voice channel, SP5's skill loader) are modules in the same process, calling into `AgentManager` directly. **No internal RPC, no event bus in SP1.** The display in SP2 needs an HTTP+WSS endpoint because browsers can't call into a Node module — but that endpoint is *internal*, not a process boundary.

If the user ever wants the agent on a server and the display on a separate machine over LAN, the WSS adapter can become external — refactor, not rewrite.

### 4.3 Pi as a library, not a fork

pi packages are consumed from npm: `@earendil-works/pi-agent-core`, `@earendil-works/pi-ai`, and later `@earendil-works/pi-web-ui` (SP2). When pi releases new versions, `npm update`. Forking pi (e.g. forking pi-coding-agent and replacing its TUI with our channels) was considered and rejected; the maintenance tax of tracking upstream is not worth the inherited machinery.

### 4.4 Forward-compatible seams

SP1 introduces five small seams that have trivial implementations now but are the integration points for SP2–SP5:

| Seam | Where | SP1 behavior | Fills in |
|---|---|---|---|
| `ChannelInterface` | `src/channels/ChannelInterface.ts` | `start()`/`stop()` only. Implemented by Telegram in SP1. | SP2 (display), SP4 (voice). |
| `Identity` | `src/agent/identity.ts` | Single hardcoded identity: a system prompt + a static tool list. | SP5 skill registry contributes tools and possibly identities. |
| `InputProcessorRegistry` | `src/agent/inputs/registry.ts` | Built-ins for `text` and `photo`. All other kinds polite-stub. | SP5 skills register processors for `voice`, `document`, `url`, etc. |
| `Router` | `src/agent/router/router.ts` | Zero rules. `decide()` returns the configured default `{ identityName, model, tools, thinkingLevel }`. | SP5 skills register routing rules (image → vision model, intent → identity, latency → fast model, etc.). |
| `beforeToolCall` policy / `afterToolCall` audit | Built into `agent/factory.ts` | Policy hook is a no-op (allow everything). Audit hook writes one JSON-per-line to `~/.paddelino/audit.log`. | SP5 fills policy with the ZeroClaw-style autonomy levels (readonly / workspace-write / supervised / full). |

### 4.5 Future direction (not SP1, but architecturally relevant)

**Traceability and undo for agent-authored data.** `~/.paddelino` should become a git repo. Writes initiated by the agent (memories in SP3, skills in SP5, notes generally) go through a wrapper: `write → git add → git commit -m "agent: <op> [user=<id>, session=<id>]"`. Conversations stay outside git (too noisy — they're high-frequency, low-value to version). Result: every agent-driven change has a commit, so wrong writes are recoverable with `git revert`, and "why does the assistant believe X about me?" is answerable with `git log -- memory/<userId>`. SP3 will add the wrapper. SP1 keeps the door open by treating `~/.paddelino` as a flat directory and avoiding designs that would make later git tracking awkward.

## 5. Architecture

### 5.1 Process diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                          paddelino (one process)                       │
│                                                                        │
│  ┌──────────────────────┐                  ┌─────────────────────────┐ │
│  │   TelegramChannel    │ ── on msg ────▶ │     AgentManager        │ │
│  │  (grammY runner)     │                  │     userId → Agent      │ │
│  │   - handlers/*       │                  │     (lazy + cached)     │ │
│  │   - StreamingState   │ ◀── events ─── │                          │ │
│  │   - security         │                  └──────────┬──────────────┘ │
│  │   - formatting       │                              │                │
│  │   - input mapping    │                       createAgent()           │
│  └──────────┬───────────┘                              │                │
│             │                                          ▼                │
│             │     ┌──────────────────────────────────────────────────┐ │
│             │     │  InputProcessorRegistry  (text, photo, ...)      │ │
│             │     └──────────────────────────────────────────────────┘ │
│             │                                          │                │
│             │                                          ▼                │
│             │     ┌──────────────────────────────────────────────────┐ │
│             │     │  Router  (zero rules in SP1; returns defaults)   │ │
│             │     └──────────────────────────────────────────────────┘ │
│             │                                          │                │
│             │     ┌──────────────────────────────────────────────────┐ │
│             └────▶│  pi Agent (one per user)                         │ │
│      subscribe()  │   - state.systemPrompt   (from identity)         │ │
│      per prompt   │   - state.tools          (from identity + Router)│ │
│                   │   - state.model          (from Router)           │ │
│                   │   - beforeToolCall       (no-op stub in SP1)     │ │
│                   │   - afterToolCall        (audit log)             │ │
│                   └─────────────────────┬────────────────────────────┘ │
│                                         │                               │
│                                         ▼                               │
│                          ┌──────────────────────────┐                  │
│                          │ persistence (JSON files) │                  │
│                          │ ~/.paddelino/users/<id>/ │                  │
│                          │   conversations/         │                  │
│                          │     default.json         │                  │
│                          └──────────────────────────┘                  │
└────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Repo layout

```
paddelino/
├── src/
│   ├── index.ts                       # entrypoint: load config, wire channels, start
│   ├── config.ts                      # env + .env loading, allowlist, paths
│   ├── agent/
│   │   ├── identity.ts                # the shared assistant identity (one in SP1)
│   │   ├── factory.ts                 # createAgent(userId, config, loadedMessages)
│   │   ├── manager.ts                 # AgentManager: per-user cache + lifecycle
│   │   ├── persistence.ts             # load/save Context JSON (atomic write)
│   │   ├── tools/
│   │   │   ├── current-time.ts
│   │   │   └── note.ts
│   │   ├── inputs/
│   │   │   ├── types.ts               # RawInput, InputProcessor, UserMessage
│   │   │   ├── registry.ts            # InputProcessorRegistry
│   │   │   └── processors/
│   │   │       ├── text.ts            # pass-through
│   │   │       └── photo.ts           # base64 + multimodal content block
│   │   └── router/
│   │       ├── types.ts               # RoutingDecision, RoutingRule, RoutingContext
│   │       └── router.ts              # Router (zero rules in SP1)
│   ├── channels/
│   │   ├── ChannelInterface.ts
│   │   └── telegram/
│   │       ├── bot.ts                 # grammY setup
│   │       ├── handlers/
│   │       │   ├── commands.ts        # /start /reset /stop /retry /status /model
│   │       │   ├── text.ts
│   │       │   ├── photo.ts
│   │       │   └── stub.ts            # polite stubs for voice/document/audio/video
│   │       ├── streaming.ts           # consolidated status callback (ported)
│   │       ├── formatting.ts          # Markdown → HTML (ported)
│   │       └── security.ts            # allowlist + token-bucket rate limiter (ported)
│   └── shared/
│       ├── audit.ts                   # append-only JSON-per-line audit log
│       ├── lock.ts                    # PID lock (ported)
│       └── types.ts
├── scripts/
│   ├── install.sh
│   └── systemd/paddelino.service.template
├── tests/
│   ├── unit/                          # bun test
│   └── integration/                   # using pi-ai's registerFauxProvider
├── .env.example
├── package.json
├── tsconfig.json
└── README.md
```

All sources under `src/` so `bun build --compile src/index.ts --outfile dist/paddelino` produces a single executable.

### 5.3 Module responsibilities

| Module | Responsibility |
|---|---|
| `config.ts` | Parse env + optional `.env` files (process env → `~/.paddelino/.env` → cwd `.env`, later wins). Validate required vars. Expose typed `Config`. Fail fast at startup with readable errors. |
| `agent/identity.ts` | `getSystemPrompt()` reads `~/.paddelino/identity/system-prompt.md` (bootstrapped on first run — see §5.4). `getTools()` returns the SP1 hardcoded tool list. `bootstrapPersona(userInput, model) → string` runs a one-shot pi-ai completion with a hardcoded meta-prompt template that turns a 1–2 sentence persona description into a polished system prompt. The seam where SP5's skill registry plugs in. |
| `agent/factory.ts` | `createAgent(userId, config, loadedMessages) → pi.Agent`. Builds the Agent with identity, configured model, configured thinking level, `beforeToolCall` stub, `afterToolCall` audit, and pre-populated messages from disk. |
| `agent/manager.ts` | `AgentManager` singleton. `getAgent(userId): Promise<Agent>`. Holds Agents for process lifetime (no eviction in SP1 — household scale makes RAM cost trivial). Subscribes to `agent_end` to persist Context. |
| `agent/persistence.ts` | `load(userId)`/`save(userId, context)`. Atomic write (tmp file in same dir → fsync → rename). Robust to crashes. Handles corrupted JSON by archiving and starting fresh with a user-facing warning. |
| `agent/tools/current-time.ts` | TypeBox-defined tool: returns the current date-time in a given timezone. |
| `agent/tools/note.ts` | TypeBox-defined tool: writes a text note to `~/.paddelino/users/<id>/notes/<iso-ts>.md`. SP3 will replace with a real memory tool while keeping the schema stable. |
| `agent/inputs/registry.ts` | Registers `InputProcessor`s by `RawInput.kind`. `process(input)` dispatches to the matching processor and returns a `UserMessage[]` (pi-compatible). |
| `agent/inputs/processors/text.ts` | Pass-through: wraps text into `{ role: "user", content: text, timestamp }`. |
| `agent/inputs/processors/photo.ts` | Downloads the photo, base64-encodes, wraps as `{ role: "user", content: [{ type: "text", text: caption }, { type: "image", data, mimeType }], timestamp }`. Handles Telegram media groups by buffering with a 1-second timeout (port the pattern from the current bot). |
| `agent/router/router.ts` | `Router` class. `register(rule)`, `decide(input, userId, ctx) → RoutingDecision`. In SP1 the rules list is empty; `decide()` always returns the configured default. |
| `channels/ChannelInterface.ts` | `interface Channel { start(): Promise<void>; stop(): Promise<void>; }`. |
| `channels/telegram/bot.ts` | Sets up grammY + grammyjs/runner, registers handlers, wires `StreamingState` factory. |
| `channels/telegram/handlers/*` | One handler per input type. Build `RawInput`, call `InputProcessorRegistry`, call `Router`, get Agent from `AgentManager`. If `agent.state.isStreaming`, call `agent.steer(userMessage)` and reply `"Added — will fold into the current reply."` Otherwise set up `StreamingState` and call `agent.prompt(...)`. Unhandled input types return polite stubs. |
| `channels/telegram/streaming.ts` | Consolidated status callback. Subscribes to a pi Agent for one prompt; consolidates `message_update.text_delta` events into one streamed Telegram message edited every ~500ms; reflects `tool_execution_*` as status lines; finalizes on `agent_end`; unsubscribes. |
| `channels/telegram/formatting.ts` | Markdown → Telegram-safe HTML. Ported from current bot. |
| `channels/telegram/security.ts` | Allowlist check; token-bucket rate limiter per user. Ported from current bot. |
| `shared/audit.ts` | `audit(eventKind, userId, details)` appends one JSON line to `~/.paddelino/audit.log`. |
| `shared/lock.ts` | PID lock file at `~/.paddelino/.lock`. Reclaims if stale (PID not running). Ported from current bot's `process-lock.ts`. |

### 5.4 First-run persona bootstrap

paddelino is built around one shared assistant identity per household (§4.1). Rather than ship with a hardcoded persona or force the user to write their own system prompt, **the user describes the assistant in 1–2 sentences and a meta-prompt generates the polished system prompt**.

**Trigger.** On startup, `agent/identity.ts` checks whether `~/.paddelino/identity/system-prompt.md` exists. If not, the bot enters bootstrap mode: the first message from any allowlisted user is intercepted, regardless of content.

**Bootstrap dialog (Telegram):**

1. Bot: `"Hi! Before we start, describe what kind of assistant you'd like me to be — 1–2 sentences. Tone, focus, anything important. (e.g. 'A friendly home assistant focused on cooking and reminders. Casual tone, brief replies.')"`
2. User replies with a free-form description.
3. Bot runs `bootstrapPersona(userInput, defaultModel)`: a one-shot pi-ai completion using a hardcoded meta-prompt:

   ```
   You are a prompt engineer. Write a complete, polished system prompt for a household assistant that will run as a Telegram bot.

   The user has described the assistant they want as:
   "{USER_INPUT}"

   The prompt you write MUST:
   - Establish the persona (tone, perspective, optional name).
   - Define scope and boundaries (what to help with; what to politely decline).
   - Set style guidelines (verbosity, formality, response length).
   - Note that responses appear in Telegram — keep markdown simple (no tables, no complex nesting).
   - Avoid hardcoding the date or other time-sensitive facts (the assistant is told the current date separately).

   Output ONLY the system prompt itself, with no preamble, no explanation, and no surrounding quotes.
   ```

4. The generated text is written to `~/.paddelino/identity/system-prompt.md`. The raw user input is written to `~/.paddelino/identity/persona-input.md` for reference and re-generation.
5. Bot replies: `"Got it. Here's the persona I'll use:\n\n<generated prompt>\n\nIf you'd like to change it, run /persona. Otherwise — how can I help?"`
6. The user's original first message is **not** forwarded to the LLM as a prompt; bootstrap consumes it. The next message starts normal interaction.

**Re-generation (`/persona`).** Re-runs the bootstrap flow against all allowlisted users:

- `/persona` (no args) → bot asks for a new description.
- `/persona <description>` → bot uses the inline description directly.
- After regeneration, all per-user pi `Agent` instances are evicted from `AgentManager` so they re-create with the new system prompt on the next message. (Existing message history per user is preserved.)

**Failure handling.**
- If the bootstrap LLM call fails (provider error, timeout): bot replies `"Couldn't generate the persona right now — try again, or set ~/.paddelino/identity/system-prompt.md by hand."` Bootstrap state remains; next message retries.
- If `system-prompt.md` is empty or unreadable at startup: bootstrap mode kicks in as if it didn't exist.

**Bootstrap meta-prompt rationale.** The meta-prompt approach means the user doesn't need to know prompt-engineering. Their description ("friendly home assistant, cooking and reminders, brief replies") becomes a structured prompt with persona, scope, style, and channel guidance baked in. SP5 will make the meta-prompt itself skill-installable.

### 5.5 Key types (sketch)

```ts
// agent/inputs/types.ts
type RawInput =
  | { kind: "text";    payload: { text: string };                                          meta: Meta }
  | { kind: "photo";   payload: { fileIds: string[]; caption?: string; mimeTypes: string[] }; meta: Meta }
  // future kinds: "voice" | "audio" | "document" | "video" | "url"
  ;

interface Meta {
  userId: string;
  channel: "telegram";       // SP2 adds "display"; SP4 adds "voice"
  timestamp: number;
}

interface InputProcessor {
  kind: RawInput["kind"];
  process(input: RawInput, ctx: ProcessorContext): Promise<UserMessage[]>;
}

// agent/router/types.ts
type RoutingDecision = {
  identityName: string;       // SP1: always "default"
  model: Model<any>;          // pi-ai Model
  tools: AgentTool<any>[];
  thinkingLevel: ThinkingLevel;
};

interface RoutingRule {
  match(input: RawInput, userId: string, ctx: RoutingContext): Promise<RoutingDecision | null>;
}
```

## 6. Message lifecycle

Trace of a text message, end to end:

```
Telegram update arrives (grammY runner)
        │
        ▼
TelegramChannel handler (text.ts)
        │
        ├── auth check       — non-allowlisted → silent drop, audit
        ├── rate-limit check — exceeded → "slow down" reply
        │
        ▼
Build RawInput { kind: "text", payload: { text }, meta }
        │
        ▼
InputProcessorRegistry.process(rawInput)
        │   text processor → UserMessage[]
        ▼
Router.decide(rawInput, userId, ctx)
        │   SP1: returns configured default
        ▼
AgentManager.getAgent(userId)
        │   cache miss?
        │     ├── persistence.load(userId) → { messages, ... }
        │     ├── createAgent(userId, config, messages)
        │     └── attach `agent_end` listener → persistence.save(...)
        ▼
agent.state.model        = decision.model
agent.state.tools        = decision.tools
agent.state.systemPrompt = identity.getSystemPrompt(decision.identityName)
agent.state.thinkingLevel = decision.thinkingLevel
        │
        ▼
StreamingState (per-prompt)
        │   sends initial "Thinking..." Telegram message
        │   subscribes to agent events
        ▼
agent.prompt(userMessage)
        │
        ▼  Pi loop:
        │    agent_start
        │    turn_start
        │    message_start  (user message)       → no Telegram update
        │    message_end    (user)
        │    message_start  (assistant)
        │    message_update (text_delta) ─┐
        │    message_update (text_delta)  │  StreamingState consolidates,
        │    ...                          │  edits Telegram msg every ~500ms
        │                                 ┘
        │    [if tool calls happen:]
        │    message_end    (assistant with toolCall)
        │    tool_execution_start  → status appended ("Calling X...")
        │      [beforeToolCall: SP1 no-op]
        │      tool.execute(...)
        │      [afterToolCall: audit log entry]
        │    tool_execution_end    → status updated ("Done: X")
        │    message_start/end  (toolResult)
        │    turn_end       (with toolResults)
        │    turn_start                                      ← next turn
        │    ... more text_delta events ...
        │    message_end
        │    turn_end
        │
        │    agent_end ─────► persistence.save(...)
        │                     StreamingState finalizes (HTML formatting),
        │                     unsubscribes
        ▼
```

### Variations by input kind

| Telegram input | SP1 |
|---|---|
| Any message when no persona file exists | First-run bootstrap (§5.4). Bot asks for a 1–2 sentence description, runs `bootstrapPersona()`, stores the result, then resumes normal interaction on the next message. |
| Text | Goes through `text` processor (above). |
| Photo / media group | `photo` processor: downloads, base64-encodes, builds a user message with text + image content blocks (`ImageContent` from `@earendil-works/pi-ai`). Pi handles vision-capable models transparently. |
| Text/photo arriving while `state.isStreaming` | Call `agent.steer(userMessage)`. Reply: `"Added — will fold into the current reply."` Pi injects after the current assistant turn finishes (`steeringMode: "one-at-a-time"`). |
| Voice message | Polite stub: `"Voice isn't supported yet — that's coming in a later version. Send me text and I'll help."` Audited. |
| Audio file | Polite stub: `"Audio files aren't supported yet. Text works."` Audited. |
| Document / PDF | Polite stub: `"Documents aren't supported yet. Paste the text and I'll help."` Audited. |
| Video / video note | Polite stub: `"Video isn't supported yet. Send text or a photo."` Audited. |
| File with `/raw` caption | Polite stub: `"Raw file forwarding isn't supported yet."` Audited. |
| URL in text | No special handling; passed through as plain text. URL preprocessing is SP5. |
| `/start` | Greeting; ensures user dir exists. |
| `/reset` | Archives `default.json` → `default.json.archived-<ts>`; drops the in-memory Agent so the next message lazy-loads a fresh one. |
| `/stop` | `agent.abort()` for that user's current run. Partial content preserved (`stopReason: "aborted"`). |
| `/retry` | `agent.continue()`. Continues from current context without a new user message. |
| `/status` | Reports model, provider, message count, last activity, `state.isStreaming`. |
| `/model` (no args) | Lists currently configured provider's models via `getModels(provider)`. |
| `/model <provider> <id>` | Calls `getModel(provider, id)`; on success sets `agent.state.model`. On miss, runs Levenshtein over `getModels(provider)` and replies `"Unknown model 'X'. Did you mean 'Y'?"`. |
| `/persona` (no args) | Re-enters the bootstrap flow: bot asks for a new 1–2 sentence description; the next user message becomes the new persona input (see §5.4). |
| `/persona <description>` | Skips the dialog. Runs `bootstrapPersona(description, model)` directly, overwrites `system-prompt.md`, evicts all per-user `Agent`s so the next prompt picks up the new persona. |

## 7. Persistence

### Layout

```
~/.paddelino/                              # base; configurable via PADDELINO_HOME
├── config.toml                       # reserved (deferred — env is enough for SP1)
├── audit.log                         # append-only, JSON-per-line
├── .lock                             # PID lock
├── identity/                         # shared assistant persona (§5.4)
│   ├── system-prompt.md              # the generated system prompt (bootstrapped on first run)
│   └── persona-input.md              # the raw 1–2 sentence user input that generated it
├── users/
│   └── <userId>/
│       ├── conversations/
│       │   └── default.json          # pi Context: { systemPrompt, messages, ... }
│       └── notes/
│           └── <iso-ts>.md
└── (reserved for later)
    ├── skills/                       # SP5
    ├── memory/<userId>/              # SP3
    └── runtime/
```

Reserved directories are part of the contract but are not created until their sub-project lands.

### Save semantics

- **Trigger:** `agent_end` event only. Mid-stream state isn't saved; on crash we lose the in-flight turn, and the user can retry.
- **Format:** the entire pi `Context`, plus envelope:
  ```json
  {
    "paddelinoVersion": "0.1.0",
    "piVersion": "x.y.z",
    "savedAt": "<iso>",
    "context": { ... pi Context ... }
  }
  ```
- **Atomic write:** write to `default.json.tmp` in same dir → `fsync` → `rename`. Linux guarantees same-FS rename atomicity.
- **Local-disk recommendation:** keep `~/.paddelino` on local disk. NFS and other network filesystems may not honor `rename` atomicity, which would break crash-safety.
- **Process exclusivity:** one PID per `~/.paddelino`. New starts fail fast.

### Load semantics

- **Lazy.** First message from a user triggers load. Missing file → start fresh.
- **Corrupt file:** archive to `default.json.corrupt-<ts>` and start fresh with a user-facing warning.
- **Schema mismatch after a pi upgrade:** if the envelope's `piVersion` is incompatible with the running pi-agent-core (parse fails, or pi rejects the loaded Context), archive to `default.json.incompatible-<ts>` and start fresh with a user-facing warning.

### Audit log

- Append-only, JSON-per-line. Event kinds: `auth_denied`, `rate_limited`, `prompt_received`, `tool_call`, `tool_result`, `error`, `agent_end`, `restart`. No rotation in SP1.

## 8. Errors and abort

All user-facing replies in SP1 are plain text — no emojis.

| Failure | Source | SP1 response |
|---|---|---|
| Not on allowlist | auth | Silent. Audit logged. |
| Rate limit | local | Reply: `"Slow down — try again in <n>s."` |
| Provider error | pi-ai `error` event | Reply: `"Provider error: <errorMessage>. Try /retry or /reset."` Partial content (if any) appended. Context saved with `stopReason: "error"`. |
| Provider context-length exceeded | same | Reply: `"Context window full. Run /reset to start fresh."` (SP3 replaces with auto-compaction.) |
| User abort | `/stop` → `agent.abort()` | Reply: `"Stopped."` Partial assistant content preserved (`stopReason: "aborted"`). |
| Concurrent message | text/photo arriving while `state.isStreaming` | Call `agent.steer(userMessage)`. Reply: `"Added — will fold into the current reply."` Pi defers injection to the end of the current assistant turn. |
| Tool error | tool throws | pi → `isError: true` tool result. LLM decides. Audited. |
| Persistence error | `persistence.save()` | Reply: `"Couldn't save conversation state. Your next message may repeat some context — I'll retry the save then."` Agent stays alive; next `agent_end` retries the save. |
| Telegram API 400/429 | grammY | Chunk long messages; fall back to sending as document; use grammY auto-retry for 429s. |
| Crash | OOM/SIGKILL | systemd restart, lock reclaim, lazy reload. |
| Multiple instances | startup | PID lock errors out. |

### Abort semantics

`agent.abort()` cancels the in-flight provider stream and emits `error` with `reason: "aborted"`. Partial assistant message preserved with `stopReason: "aborted"`. After abort: cached Agent stays alive; the next prompt is normal; `/retry` calls `agent.continue()` to resume the LLM call without a new user message.

### Concurrency semantics

While `agent.state.isStreaming` is true, additional inbound messages from the same user are queued with `agent.steer(userMessage)`. Pi injects the steered message immediately after the current assistant turn ends (before any new turn starts). The default `steeringMode` is `"one-at-a-time"`, so multiple stacked messages drain in order. `prompt()` is not re-entrant; this is the supported pattern.

### Not handled in SP1

- Pi runtime crashes inside the process — uncaught exception, systemd restarts, lazy reload.
- Mid-tool-call cancellation correctness — depends on each tool honoring its `signal`. SP1's two tools are fast and ignore signals.
- Cost limits — pi-ai reports usage; we don't enforce.

## 9. Build, distribution, testing

### Build

- **Runtime:** Bun (matches current bot).
- **Compile:** `bun build --compile src/index.ts --outfile dist/paddelino` produces a single ~80–100 MB executable that includes the Bun runtime and all deps.
- **Dev:** `bun run src/index.ts` or `bun --watch run src/index.ts`.

### Distribution

| Path | How |
|---|---|
| `curl -fsSL https://paddelino.example/install.sh \| sh` | Detects arch (x64/arm64), downloads matching binary from GitHub releases, installs to `~/.local/bin/paddelino`, prints next steps. |
| `npm install -g @user/paddelino` | Pulls the npm package; postinstall picks the right prebuilt binary. |
| `git clone && bun install && bun run build` | Source build for hacking. |

For SP1, ship the `install.sh` path first.

### Configuration

Sources, later wins: process env → `~/.paddelino/.env` → `cwd/.env`.

**Required:**
- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_ALLOWED_USERS` (comma-separated Telegram user IDs)
- At least one provider auth: an API key (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `OPENROUTER_API_KEY`, `GEMINI_API_KEY`, etc.) **or** a pi-ai OAuth token (Anthropic and ChatGPT OAuth flows are supported by pi-ai; setup is documented in the README).

**Provider-aware default model.** If `PADDELINO_DEFAULT_MODEL` is not set, paddelino picks a sensible default based on the first available provider, in this priority order (overridable via `PADDELINO_PROVIDER_PRIORITY`, comma-separated):

| Provider | Default model | Rationale |
|---|---|---|
| `anthropic` | `claude-haiku-4-5-20251001` | Fast, cheap, fluent enough for home-assistant chat. |
| `openai` | `gpt-5-mini` (fallback `gpt-4o-mini`) | Newest "mini" tier, low latency, low cost. |
| `openrouter` | `deepseek/deepseek-chat-v3` | Cheap, fast, capable; replaceable via env. |
| `gemini` | `gemini-2.5-flash` | Cheap, fast, vision-capable. |

`/model <provider> <id>` overrides per-conversation; `PADDELINO_DEFAULT_MODEL` overrides per-deployment.

**Optional (with defaults):**
- `PADDELINO_HOME` (default: `~/.paddelino`)
- `PADDELINO_PROVIDER_PRIORITY` (default: `anthropic,openai,openrouter,gemini`)
- `PADDELINO_DEFAULT_PROVIDER` (default: first available in priority order)
- `PADDELINO_DEFAULT_MODEL` (default: from the table above)
- `PADDELINO_DEFAULT_THINKING_LEVEL` (default: `off`; valid: `off | minimal | low | medium | high | xhigh`; silently ignored by providers without extended-thinking support)
- `PADDELINO_RATE_LIMIT_*`
- `PADDELINO_GREETING`

Missing required vars → readable error and exit 1.

### Service install

Ship `systemd/paddelino.service.template`. Documented user-systemd install:

```
~/.config/systemd/user/paddelino.service
systemctl --user enable --now paddelino
journalctl --user -u paddelino -f
```

### Update

`install.sh` is idempotent; `paddelino --update` runs it. Config and `~/.paddelino` data are untouched.

### Testing

| Layer | What |
|---|---|
| **Unit** (`bun test`) | `persistence` (atomic write, corrupted-JSON recovery, roundtrip); `inputs/processors/*`; `router` (default decision; rule ordering when rules exist); `security` (rate limiter); `streaming` (debounced edit consolidation). |
| **Integration** | Uses pi-ai's `registerFauxProvider` (built-in test fixture per pi-ai README) to script LLM responses. Minimal grammY mock. Cases: text → reply; tool call cycles; abort mid-stream; context loads after restart; corrupt context recovers gracefully. |
| **Manual smoke** | Checklist in repo: first run, /start, send text, send photo, restart preserves conversation, /stop interrupts, polite stub for voice. |

### Out of CI/build scope for SP1

- No CI pipeline beyond `bun run check` (typecheck + test).
- No release automation; first releases are manually tagged binary builds.
- No telemetry, no auto-update, no Docker image.

## 10. Porting from `claude-telegram-bot`

The following modules in `claude-telegram-bot/src/` are direct ports (potentially with minor adaptation) and **should be copied with attribution**, not re-derived from scratch:

| Current bot | New location |
|---|---|
| `src/formatting.ts` | `src/channels/telegram/formatting.ts` |
| `src/security.ts` | `src/channels/telegram/security.ts` |
| `src/process-lock.ts` | `src/shared/lock.ts` |
| `src/handlers/streaming.ts` (the consolidation pattern; event names differ) | `src/channels/telegram/streaming.ts` |
| `src/handlers/photo.ts` (media-group buffering with 1s timeout) | `src/channels/telegram/handlers/photo.ts` |
| `src/utils.ts` (audit logging helper) | `src/shared/audit.ts` |
| Parts of `src/handlers/commands.ts` (/start, /status patterns; not project commands) | `src/channels/telegram/handlers/commands.ts` |

The following are intentionally **not** ported (coding-tool specific): `session.ts`, `session-manager.ts`, `project-session.ts`, `project-aliases.ts`, `process-monitor.ts`, `long-run`, `group-links.ts`, `file-forwarder.ts`, `file-sender.ts`, `table-renderer.ts` (revisit in SP2 if the display needs it), `tts-usage.ts`, `voice-profiles.ts`, `voice-mode-state.ts`, document/voice/audio/video handlers (until SP5).

The current `claude-telegram-bot` repo continues to run unchanged during SP1 development and stays as a safety net.

## 11. Open questions

1. **Name.** `paddelino` is the working name. Re-open after SP1 ships if a better name emerges. No npm/PyPI collision known.
2. **OAuth-vs-API-key UX.** pi-ai supports OAuth for Anthropic and ChatGPT. Should `install.sh` guide users through the OAuth flow if no API keys are present, or just point at docs? Lean towards "docs only" for SP1.
3. **Telegram steer acknowledgement.** Decision committed to text reply (`"Added — will fold into the current reply."`) plus standard streaming. Open: should we also send a Telegram reaction (👍) on the steered message for stronger feedback? Defer until manual testing surfaces a real need.
4. **Bootstrap meta-prompt wording.** §5.4 has a draft; will be tuned once we see real outputs during implementation. The shape is locked; the exact instructions are open.
5. **Resolved (kept for historical reference):**
    - Assistant persona — resolved via the first-run bootstrap flow (§5.4): the user describes the assistant in 1–2 sentences and a hardcoded meta-prompt generates the polished system prompt, stored at `~/.paddelino/identity/system-prompt.md`.
    - Default model — resolved via the provider-aware table in §9.
    - `/model` validation — resolved (Levenshtein did-you-mean).
    - Polite-stub copy — resolved in §6 variations table.
    - `~/.paddelino` becomes a git repo? — **no for SP1**, defer to SP3. SP1 keeps it as a flat directory; SP3 introduces `git init` + the agent-write wrapper as a single coherent change.

## 12. Acceptance criteria for SP1

- [ ] `install.sh` on a fresh **Ubuntu 22.04 or 24.04** box installs `paddelino` and runs it after setting `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`, and one provider key (or completing one OAuth flow). Other systemd-based Linux distros (Debian 12+, Fedora 40+) are expected to work but are not part of SP1 acceptance.
- [ ] First message after install triggers the persona bootstrap dialog. After the user replies with a 1–2 sentence description, `~/.paddelino/identity/system-prompt.md` is created with a generated system prompt and shown back to the user.
- [ ] `/persona <new description>` overwrites the system prompt; subsequent replies reflect the new persona.
- [ ] An allowlisted user sending a text message receives a streaming response.
- [ ] An allowlisted user sending a photo (with optional caption) receives a vision-aware response when the configured model supports vision.
- [ ] Conversation continuity survives `paddelino` restart (last conversation continues on the next message).
- [ ] `/stop` interrupts an in-flight response; `/retry` resumes; `/reset` archives and starts fresh.
- [ ] A second message during streaming is steered into the current run; user sees `"Added — will fold into the current reply."`
- [ ] `/model <provider> <typo>` returns a "Did you mean …?" reply.
- [ ] A non-allowlisted user gets silently ignored (audit-logged).
- [ ] Rate-limited users get the `"Slow down …"` reply.
- [ ] Voice / PDF / audio / video / `/raw` send the per-type polite-stub replies.
- [ ] Process exclusivity: a second `paddelino` start fails fast.
- [ ] `bun test` and the integration test suite (with `registerFauxProvider`) pass.
- [ ] Manual smoke checklist completes without surprises.

## 13. Glossary

- **pi** — the agent harness at https://github.com/earendil-works/pi, monorepo containing `pi-agent-core`, `pi-ai`, `pi-web-ui`, `pi-tui`, `pi-coding-agent`.
- **Agent** — pi-agent-core's class representing one conversational state machine: messages + model + tools + system prompt.
- **Assistant identity** — the *shared* personality of the home assistant: system prompt + skill set + tool registry. Distinct from a pi `Agent` instance.
- **Channel** — an input/output transport (Telegram in SP1; display, voice in later sub-projects).
- **InputProcessor** — converts a `RawInput` from a channel into one or more pi `UserMessage`s.
- **Router** — picks `{ identityName, model, tools, thinkingLevel }` for a given input. Trivial defaults in SP1; skill-installable rules in SP5.
- **Skill** — a unit of capability the assistant can use. SP5 concept. Skills can register tools, input processors, routing rules, and identities.
- **Sub-agent** — a delegated task-specific `Agent` spawned by the main assistant for a specific job (e.g. designing a display screen). SP5 concept.
