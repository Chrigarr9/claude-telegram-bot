# SP1 — Telegram-on-pi Design

| | |
|---|---|
| **Status** | Draft (awaiting user review) |
| **Date** | 2026-05-12 |
| **Working name** | `pino` (placeholder; easy to swap) |
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
| **SP1** | **Telegram-on-pi** *(this document)* | New repo `pino`. Single Bun process. pi-agent-core as the runtime. Telegram channel via grammY. Per-user pi `Agent` instance, persisted to disk. Built-in `text` and `photo` input processors. Trivial Router. Two stub tools. Single installable package. | nothing |
| SP2 | Local display channel | Chromium kiosk on the Surface Book pointing at a local HTTP+WSS endpoint served by the same `pino` process. Uses `pi-web-ui` (`ChatPanel`, `ArtifactsPanel`, IndexedDB storage). | SP1 |
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
- One installable package that runs on any Linux machine: `install.sh` → `pino` on `$PATH`.
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
- Cost limits, telemetry, automatic updates, Docker images.
- Multi-host sync.

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
| `beforeToolCall` policy / `afterToolCall` audit | Built into `agent/factory.ts` | Policy hook is a no-op (allow everything). Audit hook writes one JSON-per-line to `~/.pino/audit.log`. | SP5 fills policy with the ZeroClaw-style autonomy levels (readonly / workspace-write / supervised / full). |

### 4.5 Future direction (not SP1, but architecturally relevant)

**Traceability and undo for agent-authored data.** `~/.pino` should become a git repo. Writes initiated by the agent (memories in SP3, skills in SP5, notes generally) go through a wrapper: `write → git add → git commit -m "agent: <op> [user=<id>, session=<id>]"`. Conversations stay outside git (too noisy — they're high-frequency, low-value to version). Result: every agent-driven change has a commit, so wrong writes are recoverable with `git revert`, and "why does the assistant believe X about me?" is answerable with `git log -- memory/<userId>`. SP3 will add the wrapper. SP1 keeps the door open by treating `~/.pino` as a flat directory and avoiding designs that would make later git tracking awkward.

## 5. Architecture

### 5.1 Process diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                          pino (one process)                            │
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
│                          │ ~/.pino/users/<id>/      │                  │
│                          │   conversations/         │                  │
│                          │     default.json         │                  │
│                          └──────────────────────────┘                  │
└────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Repo layout

```
pino/
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
│   └── systemd/pino.service.template
├── tests/
│   ├── unit/                          # bun test
│   └── integration/                   # using pi-ai's registerFauxProvider
├── .env.example
├── package.json
├── tsconfig.json
└── README.md
```

All sources under `src/` so `bun build --compile src/index.ts --outfile dist/pino` produces a single executable.

### 5.3 Module responsibilities

| Module | Responsibility |
|---|---|
| `config.ts` | Parse env + optional `.env` files (process env → `~/.pino/.env` → cwd `.env`, later wins). Validate required vars. Expose typed `Config`. Fail fast at startup with readable errors. |
| `agent/identity.ts` | `getSystemPrompt()`, `getTools()`. SP1 returns hardcoded values. The seam where SP5's skill registry plugs in. |
| `agent/factory.ts` | `createAgent(userId, config, loadedMessages) → pi.Agent`. Builds the Agent with identity, configured model, configured thinking level, `beforeToolCall` stub, `afterToolCall` audit, and pre-populated messages from disk. |
| `agent/manager.ts` | `AgentManager` singleton. `getAgent(userId): Promise<Agent>`. Caches Agents. Subscribes to `agent_end` to persist Context. Evicts on inactivity (default 24h). |
| `agent/persistence.ts` | `load(userId)`/`save(userId, context)`. Atomic write (tmp file in same dir → fsync → rename). Robust to crashes. Handles corrupted JSON by archiving and starting fresh with a user-facing warning. |
| `agent/tools/current-time.ts` | TypeBox-defined tool: returns the current date-time in a given timezone. |
| `agent/tools/note.ts` | TypeBox-defined tool: writes a text note to `~/.pino/users/<id>/notes/<iso-ts>.md`. SP3 will replace with a real memory tool while keeping the schema stable. |
| `agent/inputs/registry.ts` | Registers `InputProcessor`s by `RawInput.kind`. `process(input)` dispatches to the matching processor and returns a `UserMessage[]` (pi-compatible). |
| `agent/inputs/processors/text.ts` | Pass-through: wraps text into `{ role: "user", content: text, timestamp }`. |
| `agent/inputs/processors/photo.ts` | Downloads the photo, base64-encodes, wraps as `{ role: "user", content: [{ type: "text", text: caption }, { type: "image", data, mimeType }], timestamp }`. Handles Telegram media groups by buffering with a 1-second timeout (port the pattern from the current bot). |
| `agent/router/router.ts` | `Router` class. `register(rule)`, `decide(input, userId, ctx) → RoutingDecision`. In SP1 the rules list is empty; `decide()` always returns the configured default. |
| `channels/ChannelInterface.ts` | `interface Channel { start(): Promise<void>; stop(): Promise<void>; }`. |
| `channels/telegram/bot.ts` | Sets up grammY + grammyjs/runner, registers handlers, wires `StreamingState` factory. |
| `channels/telegram/handlers/*` | One handler per input type. Build `RawInput`, call `InputProcessorRegistry`, call `Router`, get Agent from `AgentManager`, set up `StreamingState`, call `agent.prompt(...)`. Unhandled types return polite stubs. |
| `channels/telegram/streaming.ts` | Consolidated status callback. Subscribes to a pi Agent for one prompt; consolidates `message_update.text_delta` events into one streamed Telegram message edited every ~500ms; reflects `tool_execution_*` as status lines; finalizes on `agent_end`; unsubscribes. |
| `channels/telegram/formatting.ts` | Markdown → Telegram-safe HTML. Ported from current bot. |
| `channels/telegram/security.ts` | Allowlist check; token-bucket rate limiter per user. Ported from current bot. |
| `shared/audit.ts` | `audit(eventKind, userId, details)` appends one JSON line to `~/.pino/audit.log`. |
| `shared/lock.ts` | PID lock file at `~/.pino/.lock`. Reclaims if stale (PID not running). Ported from current bot's `process-lock.ts`. |

### 5.4 Key types (sketch)

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
        │   sends initial "💭 Thinking..." Telegram message
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
        │    tool_execution_start  → status appended ("🔧 calling X...")
        │      [beforeToolCall: SP1 no-op]
        │      tool.execute(...)
        │      [afterToolCall: audit log entry]
        │    tool_execution_end    → status updated ("✓ X")
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
| Text | Goes through `text` processor (above). |
| Photo / media group | `photo` processor: downloads, base64-encodes, builds a user message with text + image content blocks. Pi handles vision-capable models transparently. |
| Voice, audio, document, video, video-note | Polite stub: "🎙 I can't process voice/audio/PDF/video yet — that's coming as a skill in a later version. Send me a text message." Audited. |
| `/start` | Greeting; ensures user dir exists. |
| `/reset` | Archives `default.json` → `default.json.archived-<ts>`; evicts cached Agent. |
| `/stop` | `agent.abort()` for that user's current run. Partial content preserved. |
| `/retry` | `agent.continue()`. Continues from current context without a new user message. |
| `/status` | Reports model, provider, message count, last activity, cache state. |
| `/model <provider> <id>` | `agent.state.model = getModel(provider, id)`. Validation deferred to pi-ai on next prompt. |

## 7. Persistence

### Layout

```
~/.pino/                              # base; configurable via PINO_HOME
├── config.toml                       # reserved (deferred — env is enough for SP1)
├── audit.log                         # append-only, JSON-per-line
├── .lock                             # PID lock
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
    "pinoVersion": "0.1.0",
    "piVersion": "x.y.z",
    "savedAt": "<iso>",
    "context": { ... pi Context ... }
  }
  ```
- **Atomic write:** write to `default.json.tmp` in same dir → `fsync` → `rename`. Linux guarantees same-FS rename atomicity.
- **Process exclusivity:** one PID per `~/.pino`. New starts fail fast.

### Load semantics

- **Lazy.** First message from a user triggers load. Missing file → start fresh. Corrupt file → archive to `default.json.corrupt-<ts>` and start fresh with a user-facing warning.

### Audit log

- Append-only, JSON-per-line. Event kinds: `auth_denied`, `rate_limited`, `prompt_received`, `tool_call`, `tool_result`, `error`, `agent_end`, `restart`. No rotation in SP1.

## 8. Errors and abort

| Failure | Source | SP1 response |
|---|---|---|
| Not on allowlist | auth | Silent. Audit logged. |
| Rate limit | local | Reply: "⏱ slow down — try again in `<n>`s". |
| Provider error | pi-ai `error` event | Message becomes "❌ provider issue: `<errorMessage>`. Try `/retry` or `/reset`." Partial content (if any) appended. Context saved with `stopReason: "error"`. |
| Provider context-length exceeded | same | Reply: "🪨 context window full. Run `/reset` to start fresh." (SP3 replaces with auto-compaction.) |
| User abort | `/stop` → `agent.abort()` | Reply: "⏹ stopped." Partial assistant content preserved. |
| Tool error | tool throws | pi → `isError: true` tool result. LLM decides. Audited. |
| Persistence error | `persistence.save()` | Reply: "⚠ couldn't save state — your reply is above but next message will resume from before it." Agent stays alive; next `agent_end` retries the save. |
| Telegram API 400/429 | grammY | Chunk long messages; fall back to sending as document; use grammY auto-retry for 429s. |
| Crash | OOM/SIGKILL | systemd restart, lock reclaim, lazy reload. |
| Multiple instances | startup | PID lock errors out. |

### Abort semantics

`agent.abort()` cancels the in-flight provider stream and emits `error` with `reason: "aborted"`. Partial assistant message preserved with `stopReason: "aborted"`. After abort: cached Agent stays alive; the next prompt is normal; `/retry` calls `agent.continue()` to resume the LLM call without a new user message.

### Not handled in SP1

- Pi runtime crashes inside the process — uncaught exception, systemd restarts, lazy reload.
- Mid-tool-call cancellation correctness — depends on each tool honoring its `signal`. SP1's two tools are fast and ignore signals.
- Cost limits — pi-ai reports usage; we don't enforce.

## 9. Build, distribution, testing

### Build

- **Runtime:** Bun (matches current bot).
- **Compile:** `bun build --compile src/index.ts --outfile dist/pino` produces a single ~80–100 MB executable that includes the Bun runtime and all deps.
- **Dev:** `bun run src/index.ts` or `bun --watch run src/index.ts`.

### Distribution

| Path | How |
|---|---|
| `curl -fsSL https://pino.example/install.sh \| sh` | Detects arch (x64/arm64), downloads matching binary from GitHub releases, installs to `~/.local/bin/pino`, prints next steps. |
| `npm install -g @user/pino` | Pulls the npm package; postinstall picks the right prebuilt binary. |
| `git clone && bun install && bun run build` | Source build for hacking. |

For SP1, ship the `install.sh` path first.

### Configuration

Sources, later wins: process env → `~/.pino/.env` → `cwd/.env`.

Required:
- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_ALLOWED_USERS` (comma-separated Telegram user IDs)
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, or any pi-ai provider key

Optional (defaults):
- `PINO_HOME` (default: `~/.pino`)
- `PINO_DEFAULT_PROVIDER` (default: `anthropic`)
- `PINO_DEFAULT_MODEL` (default: `claude-sonnet-4-5-20250929`)
- `PINO_DEFAULT_THINKING_LEVEL` (default: `off`)
- `PINO_RATE_LIMIT_*`
- `PINO_GREETING`

Missing required vars → readable error and exit 1.

### Service install

Ship `systemd/pino.service.template`. Documented user-systemd install:

```
~/.config/systemd/user/pino.service
systemctl --user enable --now pino
journalctl --user -u pino -f
```

### Update

`install.sh` is idempotent; `pino --update` runs it. Config and `~/.pino` data are untouched.

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

1. **Default model.** `claude-sonnet-4-5-20250929` is the current placeholder. May change before SP1 implementation if a more cost-effective default is preferred.
2. **Name.** `pino` is a placeholder. Re-open after SP1 ships if a better name emerges.
3. **`/model` validation.** SP1 defers to pi-ai's runtime error. We may want a friendlier "did you mean…?" experience; not committed.
4. **Polite-stub copy.** Exact wording of the "I can't handle voice/PDF yet" messages will be tuned during implementation; the contract is "say something useful, not silent."
5. **Whether `~/.pino` becomes a git repo in SP1.** Decision: **no — defer to SP3.** SP1 keeps `~/.pino` as a flat directory. SP3 introduces both `git init` and the agent-write wrapper as a single coherent change. An empty git repo with no commits adds confusion without value.

## 12. Acceptance criteria for SP1

- [ ] `install.sh` on a fresh Ubuntu 22.04/24.04 box installs `pino` and runs it after setting `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`, and one provider key.
- [ ] An allowlisted user sending a text message to the bot receives a streaming response.
- [ ] An allowlisted user sending a photo (with optional caption) receives a vision-aware response when the configured model supports vision.
- [ ] Conversation continuity survives `pino` restart (last conversation continues on the next message).
- [ ] `/stop` interrupts an in-flight response; `/retry` resumes; `/reset` archives and starts fresh.
- [ ] A non-allowlisted user gets silently ignored (with an audit log entry).
- [ ] Rate-limited users get the "slow down" reply.
- [ ] Voice/PDF/audio/video send polite-stub replies.
- [ ] Process exclusivity: a second `pino` start fails fast.
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
