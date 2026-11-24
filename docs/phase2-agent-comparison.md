# Phase 2 Agent Comparison and Recommendation

This document compares the current `claude-telegram-bot` architecture to a few widely used personal-agent projects:

- OpenClaw
- Hermes Agent
- NanoClaw
- ZeroClaw

The goal is not to copy them wholesale.

The goal is to decide what a lean "Phase 2" version of this project should become:

- keep the strong Telegram UX you already have
- reduce architectural coupling
- improve security boundaries
- support later extension without inheriting framework bloat

## Executive Summary

Your current bot is best understood as:

- a Telegram-native remote console for Claude Code
- with strong UX around formatting, streaming, voice, and multi-project routing
- but with a runtime that is still tightly coupled to Claude Code and Telegram

The best next step is not "become OpenClaw."

The best next step is:

1. Keep Telegram as the only first-class channel for now.
2. Turn the agent runtime into a real abstraction, not just a planned one.
3. Separate transport, session orchestration, and agent backend cleanly.
4. Move from "Claude Code with bypass permissions plus defensive checks" toward a narrower execution boundary.
5. Add other channels later only if the transport layer is already stable.

The closest strategic fit is:

- UX direction from your current bot
- gateway separation ideas from OpenClaw and Hermes
- minimalism and isolation instincts from NanoClaw
- explicit policy levels from ZeroClaw

## 1. What Your Bot Is Today

### Strengths

Your current implementation already has several things that are genuinely good:

- Telegram-first interaction model, instead of treating messaging as an afterthought.
- Streaming status updates and thought visibility.
- Rich Telegram formatting, including table rendering and file fallbacks.
- Voice input and voice output.
- Multi-project routing and session continuity.
- Group-to-project linking, which is a practical primitive for real use.
- Long-running process monitoring and notification back into chat.

These are not trivial features. They are the main reason this project is already useful.

### Local architecture summary

Today the runtime is still centered on `ClaudeSession` and the Anthropic Claude Agent SDK:

- direct Claude Code session management in [src/session.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/session.ts:228)
- full bypass mode in [src/session.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/session.ts:277)
- security model documented in [SECURITY.md](/mnt/Shared/Code/projects/claude-telegram-bot/SECURITY.md:5)
- Telegram formatting pipeline in [src/formatting.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/formatting.ts:86)
- multi-project session manager in [src/session-manager.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/session-manager.ts:38)
- group linking and voice propagation summarized in [GROUP-LINKING-SUMMARY.md](/mnt/Shared/Code/projects/claude-telegram-bot/GROUP-LINKING-SUMMARY.md:5)

### Important architectural reality

There is already an adapter abstraction:

- interface in [src/ai-agent-interface.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/ai-agent-interface.ts:98)
- registry/factory in [src/adapters/index.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/adapters/index.ts:15)

But it is not the real production runtime yet.

The project docs explicitly describe the adapter work as partially complete and still pending integration:

- [ADAPTER_LAYER_SUMMARY.md](/mnt/Shared/Code/projects/claude-telegram-bot/ADAPTER_LAYER_SUMMARY.md:205)

And the non-Claude backends are still templates, not working adapters:

- Codex template throws immediately in [src/adapters/codex-adapter.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/adapters/codex-adapter.ts:46)

So the current system is not yet:

- multi-agent backend
- multi-provider in practice
- transport/runtime decoupled

It is primarily a Telegram shell around Claude Code.

## 2. Comparison With Other Agent Projects

## OpenClaw

Primary sources:

- https://github.com/openclaw/openclaw
- https://github.com/openclaw/openclaw/blob/main/docs/channels/telegram.md

### What OpenClaw is optimized for

OpenClaw is a gateway-centric personal assistant platform.

Its core bet is:

- one control plane
- many messaging channels
- agent routing across channels, groups, topics, and identities
- many built-in capabilities

The official docs position the gateway as the center of the system, with channels, sessions, tools, pairing, and routing flowing through it.

### What it does well

- Extremely strong channel abstraction.
- Production-grade Telegram policy surface.
- Pairing and allowlist models for DMs and groups.
- Routing rules for accounts, groups, and topics.
- Large ecosystem and very broad feature set.

### What it costs

- Large surface area.
- More moving parts.
- Security complexity increases with feature breadth.
- Harder to fully understand end-to-end.

### Relevance to your project

The most relevant OpenClaw ideas are:

- strict separation between channel transport and agent execution
- deterministic routing rules
- explicit access policy per channel/group/topic
- not letting the model choose the delivery channel heuristically

The least relevant thing to copy is the total platform scope.

You do not need:

- 20+ channels
- dashboard sprawl
- giant config surface
- host-wide tool matrix from day one

## Hermes Agent

Primary sources:

- https://github.com/NousResearch/hermes-agent
- https://github.com/NousResearch/hermes-agent/blob/main/gateway/platforms/telegram.py

### What Hermes is optimized for

Hermes is an agent platform, not just a messenger bot.

Its core bets are:

- one gateway for CLI plus messaging
- multi-provider model flexibility
- learning loop and memory growth
- scheduling and subagent delegation
- multiple execution backends

The Hermes README makes the architecture direction very explicit:

- one gateway process for Telegram, Discord, Slack, WhatsApp, Signal, and CLI
- multiple terminal backends like local, Docker, SSH, Daytona, Singularity, and Modal
- built-in cron, memory, skills, and subagents

### What it does well

- Strong separation of messaging from execution backend.
- Multi-provider and multi-runtime flexibility.
- Rich operational model for remote execution.
- Serious long-term memory and skills story.

### What it costs

- Much larger conceptual scope than your current bot.
- More infrastructure than you need for a lean Telegram-first assistant.
- More platform complexity before the core interaction loop is perfect.

### Relevance to your project

The most relevant Hermes ideas are:

- transport should not care which runtime backend is used
- execution backend should be swappable
- messaging and CLI can share command semantics
- background jobs and long-running tasks are first-class

That aligns well with what your bot already does around long-run tasks and resumable sessions.

## NanoClaw

Primary source:

- https://github.com/qwibitai/nanoclaw

### What NanoClaw is optimized for

NanoClaw is optimized for smallness and understandability.

Its strongest opinion is:

- keep the trunk small
- use containers for real isolation
- customize through code and skills instead of endless config

### What it does well

- Clear security story based on container isolation, not only prompt and allowlist controls.
- Small codebase philosophy.
- Good instincts around not shipping every possible channel and provider in the core.
- Good stance that features can be added modularly.

### What it costs

- More opinionated around containerized execution.
- Less immediately aligned with your current "Telegram plus direct Claude Code" flow.
- Built-in channel story is intentionally narrower in trunk.

### Relevance to your project

NanoClaw is probably the most useful corrective influence on your architecture.

Its best lesson for you is:

- keep the product understandable
- do not overbuild the base
- isolate execution more strongly than your current bypass-permissions model

## ZeroClaw

Primary source:

- https://github.com/zeroclaw-labs/zeroclaw

### What ZeroClaw is optimized for

ZeroClaw is optimized for runtime efficiency and broad pluggability.

Its core bets are:

- low-overhead Rust control plane
- provider/channel/tool abstraction through traits
- explicit autonomy levels
- broad deployment flexibility

### What it does well

- Explicit runtime policy model.
- Leaner systems story than large Node-based frameworks.
- Clear "control plane plus pluggable components" direction.

### What it costs

- Broader scope than you need.
- Rust-first runtime would amount to a rewrite, not a refinement.
- It is inspirational architecturally, but not a practical direct migration path from your current codebase.

### Relevance to your project

The biggest idea worth borrowing is not Rust.

It is the policy model:

- read-only
- supervised
- full

That maps well to a future agent runtime where not all sessions get equal power.

## 3. Feature Comparison

| System | Main interaction style | Runtime model | Channel model | Security stance | Best quality |
|---|---|---|---|---|---|
| Your bot | Telegram-first remote operator | Direct Claude Code session | Telegram only | Bypass permissions plus app-level checks | Best Telegram UX and practical project routing |
| OpenClaw | Personal assistant platform | Gateway-centric, tool-rich | Very broad multi-channel | Pairing, allowlists, sandbox options | Strongest channel and routing system |
| Hermes | General agent platform | Gateway plus multiple terminal backends | Multi-channel plus CLI | Messaging controls plus backend isolation options | Best agent-platform flexibility |
| NanoClaw | Minimal personal agent | Small host plus isolated containers | Added modularly | Isolation first | Best minimalism and understandability |
| ZeroClaw | Efficient assistant platform | Lean pluggable control plane | Broad multi-channel | Explicit autonomy and sandbox policy | Best policy framing and systems efficiency |

## 4. What Is Unique and Worth Keeping From Your Architecture

Do not lose these.

### Telegram-native presentation

Your Telegram formatting pipeline is a real differentiator:

- HTML-safe message conversion
- better table handling
- file fallbacks for oversized outputs
- voice output path

This is more polished than the average "agent replies in chat" integration.

### Project-oriented sessions

Your bot is not just "one chat to one assistant."

It already understands:

- project aliases
- per-project session continuity
- cross-chat routing to the same project
- group-to-project linking

That is a strong operational model for someone doing real work through Telegram.

### Interrupt and long-run ergonomics

The interrupt model, queueing, and background process notifications are good.

That is exactly the type of real-world behavior that agent frameworks often talk about but users only notice when it fails.

### Personal assistant plus coding agent hybrid

The bot is not pretending Claude Code is only for software development.
That is a useful product direction and already visible in the README and feature set.

## 5. What Needs To Change

## A. Runtime abstraction must become real

Right now the architecture says "adapter-based", but the runtime is still effectively Claude-specific.

Phase 2 should make this true in code, not just in docs:

- transport layer sends normalized requests
- runtime layer executes against Claude Code or another backend
- formatter/presentation layer consumes normalized events

## B. Telegram-specific UX should stop leaking into the agent core

Telegram should remain first-class, but:

- Telegram update handling
- Telegram formatting
- Telegram callback buttons
- Telegram voice/media handling

should live in the transport/presentation boundary, not inside the core session engine.

## C. Security should move from "personal trust plus defensive checks" toward capability boundaries

Current security is honest, but weak for a long-term home-server agent:

- bypass mode is always on
- destructive operations are constrained mainly by prompts and path checks
- runtime power is broad

Phase 2 should move to:

- explicit allowed tool modes
- explicit workspace roots
- optional supervised mode for risky actions
- no general host-admin privileges

## D. Do not broaden channels before stabilizing the architecture

Telegram is enough for now.

If you add Discord or WhatsApp too early, you will increase complexity before the execution model is clean.

## 6. Recommended "Base Version" Architecture

This is the architecture I would recommend for the next real version.

### Layer 1: Channel Adapter

Start with one adapter:

- Telegram

Its job:

- receive inbound messages/media/buttons
- normalize them into a transport-neutral event
- send formatted outbound responses

Later, Discord or WhatsApp can implement the same interface.

### Layer 2: Conversation Router

This should be a small, explicit module.

Its job:

- map chat/group/topic to logical workspace
- decide session key
- load per-chat/per-project settings
- choose runtime backend for this session

This should own:

- DM vs group rules
- project linking
- voice preferences
- session resume rules

### Layer 3: Agent Runtime Interface

Make the adapter contract in `src/ai-agent-interface.ts` the real runtime boundary.

At minimum the runtime should support:

- `sendMessage`
- `stop`
- `resume`
- session metadata
- streaming event emission

But extend the event model slightly so it carries:

- plain text deltas
- reasoning deltas
- tool start/end
- file artifacts
- usage
- task status

### Layer 4: Policy / Capability Layer

Before the runtime executes a task, apply a policy profile.

Suggested profiles:

- `readonly`
- `workspace-write`
- `supervised-exec`
- `full-agent`

That is the ZeroClaw-style idea worth copying.

For Phase 2, most Telegram sessions should probably run in `workspace-write` or `supervised-exec`, not `full-agent`.

### Layer 5: Persistence

Keep this simple.

Use:

- one lightweight DB for session metadata and routing
- files on disk for workspaces, memory, and exported artifacts

Do not build a giant control plane database yet.

### Layer 6: Background Jobs

Keep your long-running task model, but make it backend-neutral.

The router should track:

- active job
- originating chat(s)
- working directory
- notification policy

This is a good place to later add cron without taking on all of Hermes.

## 7. What To Borrow From Each Project

### Borrow from your current bot

- Telegram UX
- voice and media handling
- project routing
- group linking
- thoughtful formatting
- long-run notifications

### Borrow from OpenClaw

- deterministic routing
- explicit DM/group policy
- channel adapter separation
- pairing and allowlist discipline

### Borrow from Hermes

- shared semantics across CLI and messaging
- backend flexibility
- background execution as a first-class concept
- future path to subagents only when truly needed

### Borrow from NanoClaw

- smaller core
- stronger execution isolation
- resist feature bloat
- use modular additions instead of packing everything into core

### Borrow from ZeroClaw

- explicit autonomy levels
- policy-first thinking
- clean separation of control plane and pluggable backends

## 8. What Not To Borrow

Do not copy these yet:

- OpenClaw-scale channel matrix
- Hermes-scale learning loop and self-improving skill system
- ZeroClaw-scale platform breadth
- NanoClaw's "everything is a skill-installed fork mutation" workflow

Those are all valid product choices, but they are not the shortest path to a strong Phase 2 for this codebase.

## 9. Recommended Scope for Phase 2

Phase 2 should aim for this:

### Required

- Telegram remains the only production channel
- real runtime abstraction integrated into the live path
- Claude Code backend remains supported first
- optional second backend later, after the abstraction is proven
- project/session router extracted cleanly
- security profiles introduced
- artifacts and status streaming standardized

### Optional

- local CLI entrypoint that reuses the same router/runtime
- cron scheduling
- lightweight web status view

### Explicitly out of scope

- WhatsApp
- Discord
- dashboard-heavy management UI
- multi-agent swarms
- autonomous self-modification
- direct Docker or host-control powers

## 10. Suggested Implementation Sequence

1. Extract a real `ChannelAdapter` interface and keep Telegram as the only implementation.
2. Replace direct `ClaudeSession` ownership with the runtime adapter interface in the main flow.
3. Introduce a `ConversationRouter` module that owns chat-to-project/session mapping.
4. Introduce policy profiles for runtime capability.
5. Standardize streaming events and file artifact delivery.
6. Add one optional second runtime backend only after the new flow is stable.
7. Add another channel only after the router and adapter interfaces stop changing.

## 11. Final Recommendation

The best "easy AI agent that does the most important things well" is not:

- a smaller OpenClaw clone
- a Hermes clone without the learning loop
- a Rust rewrite

It is:

- your current Telegram UX
- plus a real transport/runtime split
- plus a small policy layer
- plus optional backend portability

That gives you something much stronger than the current bot without inheriting the full overhead of the larger systems.

If I had to name the target architecture in one sentence:

`Telegram-native remote agent shell with pluggable runtimes, project-aware routing, and explicit capability levels.`

## Sources

### Local project

- [README.md](/mnt/Shared/Code/projects/claude-telegram-bot/README.md:6)
- [src/session.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/session.ts:228)
- [src/ai-agent-interface.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/ai-agent-interface.ts:98)
- [src/adapters/index.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/adapters/index.ts:15)
- [src/adapters/codex-adapter.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/adapters/codex-adapter.ts:46)
- [src/formatting.ts](/mnt/Shared/Code/projects/claude-telegram-bot/src/formatting.ts:86)
- [SECURITY.md](/mnt/Shared/Code/projects/claude-telegram-bot/SECURITY.md:5)
- [GROUP-LINKING-SUMMARY.md](/mnt/Shared/Code/projects/claude-telegram-bot/GROUP-LINKING-SUMMARY.md:5)
- [ADAPTER_LAYER_SUMMARY.md](/mnt/Shared/Code/projects/claude-telegram-bot/ADAPTER_LAYER_SUMMARY.md:205)

### External official sources

- OpenClaw README: https://github.com/openclaw/openclaw
- OpenClaw Telegram docs: https://github.com/openclaw/openclaw/blob/main/docs/channels/telegram.md
- Hermes Agent README: https://github.com/NousResearch/hermes-agent
- Hermes Telegram adapter: https://github.com/NousResearch/hermes-agent/blob/main/gateway/platforms/telegram.py
- NanoClaw README: https://github.com/qwibitai/nanoclaw
- ZeroClaw README: https://github.com/zeroclaw-labs/zeroclaw
