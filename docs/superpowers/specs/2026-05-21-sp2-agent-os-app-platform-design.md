# SP2 - Agent OS App Platform Design

| | |
|---|---|
| **Status** | Draft v1 - approved in brainstorming, pending written-spec review |
| **Date** | 2026-05-21 |
| **Working name** | Agent OS App Platform v1 |
| **Author** | Chrigarr (with OpenCode in brainstorming mode) |
| **Supersedes** | SP2 section of `2026-05-12-sp1-telegram-on-pi-design.md` |
| **Related** | `docs/superpowers/specs/2026-05-12-sp1-telegram-on-pi-design.md`; `docs/superpowers/plans/2026-05-13-sp1-paddelino-implementation.md` |

## 1. Context

SP1 creates `paddelino`: a Telegram-first household assistant powered by `pi-agent-core`. The original SP1 roadmap described SP2 as a local display channel: a Chromium kiosk on the Surface Book, backed by a local HTTP+WSS endpoint.

During SP2 brainstorming, the product direction changed. The local display is still useful, but it is not the most important next substrate. The next step is the **Agent OS app platform**: a live, git-tracked app system that lets the assistant route to apps, use app-provided tools, preview app changes, and promote approved app edits without restarting `paddelino`.

The display becomes the first visible client of the app system. It stays minimal in SP2: enough to show app cards, open app screens, preview draft apps, approve/reject changes, and restore context after wake. The richer Surface dashboard/chat/artifact experience moves to a later sub-project.

## 2. Scope

### Goals

- Create `~/.paddelino/apps` as the user/agent-owned app workspace, initialized and maintained as a git repo.
- Define a real app contract: metadata, capability summary, display contributions, tools, routing rules, data loaders, and optional background jobs.
- Load app modules dynamically at runtime and hot-reload them without restarting `paddelino`.
- Let the agent propose app changes in a sandbox draft, validate them, preview them, and promote them only after user approval.
- Track every app draft, promotion, and rollback through git.
- Add a trusted App Verifier system capability that validates app drafts and gives human-readable guidance.
- Ship a `weather` app using Open-Meteo as the reference/blueprint app.
- Include a minimal local HTTP display client for dashboard cards, app screens, draft previews, approval/rejection, app health, and contextual wake/restore.
- Keep Telegram as a complete fallback/control path when no browser display is connected.

### Non-goals

- A polished always-on Surface dashboard.
- Voice wake, voice input, or voice output.
- The full memory system.
- Arbitrary self-modification of core `paddelino` code.
- Agent-authored custom frontend code or custom React components.
- Multi-host app sync or an app marketplace.
- Container-grade sandbox isolation.
- Broad subagent orchestration beyond app-aware delegation.

### Roadmap shift

| Sub-project | New focus |
|---|---|
| SP2 | Agent OS app platform, self-extension substrate, minimal display client, weather blueprint app. |
| SP3 | Rich local display shell for Surface Book: dashboard, chat, artifacts, app launcher polish. |
| SP4 | Memory system. |
| SP5 | Voice and broader skills/subagents, subject to later revision. |

## 3. Architecture

SP2 adds an `AppRuntime` layer beside the SP1 `AgentManager`.

```text
Telegram/user request
  -> AgentManager / pi Agent
  -> app-aware routing, app tools, or app delegation
  -> AppRuntime / AppChangeManager
  -> ~/.paddelino/apps git workspace
  -> DisplayServer broadcasts live or preview state
```

### Core components

| Component | Responsibility |
|---|---|
| `AppWorkspace` | Owns `~/.paddelino/apps`, initializes git, tracks live apps and draft app workspaces, and provides safe paths for app writes. |
| `AppRegistry` | Loads live app manifests/modules, validates app contracts, tracks app health, and exposes active app capabilities to the rest of `paddelino`. |
| `AppRuntime` | Executes app-provided loaders, tools, route handlers, display actions, and interval jobs through controlled interfaces. |
| `AppChangeManager` | Handles agent-proposed edits: create draft, edit draft, validate, preview, request approval, promote, and rollback. |
| `AppVerifier` | Trusted system capability that runs the reusable verification pipeline and returns developer-friendly diagnostics. |
| `DisplayServer` | Minimal local HTTP+WSS server for dashboard, app screens, preview screens, approval actions, app health, and display state. |
| `DisplayState` | Tracks active screen, app, artifact, draft preview, and status so wake/reconnect can restore contextually with dashboard fallback. |

### Runtime loading model

- Core `paddelino` code remains static and trusted.
- Live apps are loaded from `~/.paddelino/apps/live/<appId>` or equivalent git-tracked live directories.
- Draft apps are loaded from separate git-backed draft workspaces, implemented as branches or worktrees under the apps repo.
- A draft can be loaded in preview mode without replacing the live app.
- App modules are verified before preview or activation.
- If a promoted app fails to reload, the previous known-good app version remains active or is restored automatically.

### Security boundary

SP2 treats apps as **trusted-but-reviewable local code**, not hardened sandboxed code. Safety comes from:

- Telegram allowlist and existing SP1 user trust boundary.
- App permission declarations.
- Draft-only write paths.
- App Verifier checks before preview and promotion.
- Explicit user approval before promotion.
- Git history for every promoted app change.
- Known-good rollback on load failure.

True process/container sandboxing can be added later without changing the app contract.

## 4. App Contract

Each app is a small TypeScript module plus manifest-like metadata. The contract should be strict enough to validate automatically and simple enough for the agent to author.

### Required fields

- `appId`: stable lowercase ID, e.g. `weather`.
- `name`: human-readable app name.
- `version`: app version.
- `description`: what the app does.
- `capabilities`: concise summary for the main agent, including what the app can answer and what actions it supports.
- `permissions`: declared capabilities, such as `network:http`, `display`, `tool`, or `job`.
- `display`: dashboard card and full-screen view definitions.
- `healthCheck`: app-level check that validates enough config/data access to report whether the app is usable.

### Optional fields

- `tools`: agent-callable functions contributed by the app.
- `routing`: examples/rules that help the main router pick this app.
- `invoke`: generic natural-language app endpoint for app-aware delegation.
- `loaders`: data fetchers used by tools, screens, cards, or jobs.
- `jobs`: simple interval-based background refresh work.
- `actions`: app actions such as refresh, update settings, or open details.
- `artifacts`: structured outputs the app can render or hand to the display.
- `tests`: optional app-specific tests run by the App Verifier.

Conceptual shape:

```ts
export default defineApp({
  appId: "weather",
  name: "Weather",
  version: "0.1.0",
  description: "Current weather and short forecasts.",
  capabilities: {
    summary: "I can show current weather, short forecasts, and rain outlooks.",
    examples: ["what is the weather", "will it rain today"],
  },
  permissions: ["network:http", "display", "tool", "job"],
  display: {
    card: weatherCard,
    screen: weatherScreen,
  },
  loaders: {
    currentWeather: loadCurrentWeather,
  },
  tools: {
    get_current: getCurrentWeatherTool,
  },
  routing: [
    {
      intent: "weather",
      examples: ["what is the weather", "weather tomorrow"],
      target: "weather",
    },
  ],
  invoke: handleWeatherPrompt,
});
```

### Display output

SP2 apps produce **structured UI data**, not arbitrary frontend code. Supported primitives should be small and stable: cards, headings, text, sections, key-value rows, lists, tables, buttons/actions, status blocks, and error blocks. Charts can wait unless needed by the weather app.

Custom app-authored React components are out of scope for SP2. They would require a separate permission and stronger isolation model.

## 5. App-Aware Routing And Delegation

SP2 routing has two layers.

### Deterministic routing

Apps can contribute route examples and intent hints. The central Router remains the final decision-maker, but app routes let it quickly identify likely targets.

A route decision can include:

- `appId`
- preferred tool or action
- preferred display action
- confidence/reason
- whether to open the app screen

### Agentic delegation

The main agent should also know what apps exist. It can get this through an app catalog in context and through tools such as `list_apps`.

If no explicit route matches, the agent can inspect app capabilities and decide to delegate. For example, if a user asks “will it rain before tennis?”, the main agent can see that the `weather` app handles rain outlooks and invoke it even without a direct route match.

Delegation options:

- Call a specific app tool, e.g. `weather.get_forecast`.
- Send the prompt to the app, e.g. `invoke_app({ appId: "weather", prompt })`.
- Open an app screen, e.g. `open_app("weather")`.
- Ask the app for possible actions.
- Return output to Telegram, display, or both based on channel context.

This makes apps real routing targets, not just widgets.

## 6. Tools And Jobs

### App tools

Apps can expose agent-callable tools through the OS. Tool registration is dynamic but permission-gated.

Before an app tool is active:

- The app manifest must declare `tool` permission.
- Tool schemas must validate.
- The app must pass App Verifier checks.
- The app must be live, unless the call is explicitly scoped to a preview draft.

Weather examples:

- `weather.get_current`
- `weather.get_forecast`
- `weather.set_location`

### Background jobs

SP2 supports minimal interval jobs only.

- No cron language in SP2.
- No long-running daemon jobs in SP2.
- Job failures are logged and surfaced in app health.
- Jobs can refresh app state/cache and update display cards.

Weather example: refresh current forecast every 30-60 minutes if a location is configured.

## 7. Draft, Preview, Promotion, Rollback

App editing is a controlled lifecycle. The agent does not mutate live app code directly.

### Lifecycle

1. **Create draft**: `create_app_draft(appId | newAppId)` creates a git-backed draft branch/worktree from the live app or from a template.
2. **Edit draft**: the agent changes files only inside the draft workspace.
3. **Validate draft**: App Verifier runs manifest, type, load, permission, route, tool, display, health, and optional test checks.
4. **Preview draft**: if validation passes, the draft loads as an isolated preview version. The live app remains active.
5. **Approve promotion**: user approves explicitly through Telegram or display. The agent cannot auto-promote in SP2.
6. **Promote**: the draft replaces the live app version and commits to git with app ID, draft ID, approving user, summary, and timestamp.
7. **Reload**: App Registry hot-reloads the live app and notifies display clients.
8. **Rollback**: failed reload or user request restores the previous known-good app commit.

### Important behavior

- Live app remains active while draft is edited.
- Failed draft validation does not affect live app.
- Failed preview does not affect live app.
- Promotion is atomic at app level.
- Drafts can be abandoned without changing live apps; their git history remains available until cleanup.
- If `~/.paddelino/apps` is dirty or conflicted, promotion pauses and asks the user how to proceed.

## 8. App Verifier

SP2 includes a trusted built-in **App Verifier**. It is core/system code, not a mutable user app, because it protects the live app system. It can still appear in the app catalog so the agent knows it exists.

### Responsibilities

- Run the reusable verification pipeline for every draft preview and live reload.
- Return human-readable diagnostics for both user and agent.
- Give development guidance when validation fails.
- Produce checklist-style reports before preview or promotion.
- Expose a tool such as `app_verifier.verify_draft(draftId)`.
- Provide compact Telegram/display reports.
- Document reference rules that teach the agent how to write valid apps.

### Verification stages

1. Manifest/schema check.
2. Permission/capability consistency check.
3. TypeScript typecheck/build check.
4. Module load check.
5. Tool schema check.
6. Routing/capability summary check.
7. Display card/screen smoke render.
8. Health check.
9. Optional app-specific tests.

Example guidance:

- “Your app is missing a capability summary, so the main agent will not know when to delegate to it.”
- “The tool schema for `weather.get_current` is invalid.”
- “The screen renderer returned unsupported block type `customReactComponent`; SP2 supports structured UI blocks only.”

## 9. Minimal Display Client

The SP2 display is a basic local client for the app platform.

### Features

- Local HTTP server, e.g. `http://localhost:<port>`.
- WebSocket or SSE live updates.
- Dashboard fallback screen.
- App cards from live apps.
- Open a live app full-screen.
- Open a draft preview screen.
- Approve or reject draft promotion.
- Show app health and validation errors.
- Contextual wake/restore: restore active app/task/preview when available, otherwise dashboard.
- Telegram fallback for summaries, approvals, and errors when no display is connected.

### Non-goals

- Rich visual design.
- Custom app-authored frontend components.
- Multi-device display sync.
- Voice wake.
- Complex artifact gallery.
- Always-on requirement.

## 10. Weather Blueprint App

The `weather` app is the reference implementation for future agent-created apps.

### Data source

Use Open-Meteo because it is free, does not require an API key, and exposes simple HTTP JSON APIs. The provider remains swappable behind app loaders so the architecture does not depend on Open-Meteo.

Tests must use a mock weather provider and not depend on network access.

### Demonstrated capabilities

- Manifest with permissions: `network:http`, `display`, `tool`, and optional `job`.
- Capability summary: current weather, forecast, rain outlook, and weather dashboard card.
- Dashboard card: current temperature, condition, rain chance, and last updated.
- Full screen: current conditions plus short forecast.
- Tools: `weather.get_current`, `weather.get_forecast`, and possibly `weather.set_location`.
- Generic invocation: `invoke_app({ appId: "weather", prompt })` can answer natural weather prompts.
- Loader: fetches current and forecast data from Open-Meteo.
- Config: location from app settings or environment, with setup-time/default fallback.
- Job: optional periodic forecast refresh.
- Health check: verifies location config and Open-Meteo access.
- Error state: displays “weather unavailable” without breaking registry or display.
- Draft editability: card layout, forecast fields, location handling, and copy can be changed in draft and previewed before promotion.

The weather app should be small enough for the agent to copy as a template, but complete enough to exercise the whole lifecycle: load, route, invoke, render, edit, verify, preview, promote, and rollback.

## 11. Error Handling

- Invalid app manifests are rejected and surfaced in app health.
- Failed live app reload keeps or restores the previous known-good version.
- Failed draft validation blocks preview but preserves draft files for further edits.
- Failed draft preview does not affect the live app.
- Failed promotion rolls back automatically to the previous app commit.
- Tool and job failures are isolated to the app and surfaced in app health.
- Display disconnects do not block app operations.
- If the apps git repo is dirty or conflicted during promotion, promotion pauses and asks for user intervention.

## 12. Testing And Verification

### Unit tests

- Manifest validation.
- App registry load/reload behavior.
- Permission checks.
- Tool registration.
- Routing and app-aware delegation helpers.
- Structured UI validation.
- App Verifier diagnostics.

### Integration tests

- Load the weather app.
- Invoke weather tools with a mock provider.
- Render weather card/screen data.
- Handle weather provider failure.
- Create draft.
- Reject invalid draft.
- Preview valid draft.
- Promote draft.
- Roll back after reload failure.
- Roll back manually.
- Generate display dashboard state.
- Approve/reject draft through the display action path.

### Required project checks

- `bun run typecheck`
- `bun test`
- App Verifier pipeline for bundled apps.

## 13. Open Implementation Decisions

These are intentionally left for the implementation plan, not product ambiguity:

- Whether drafts use git branches only or git worktrees internally.
- Whether the display transport is WebSocket or SSE.
- Exact structured UI block schema.
- Exact location configuration UX for the weather app.
- Exact route-matching heuristic before agentic delegation.

## 14. Design Commitments

- SP2 is the app/self-extension substrate first and display second.
- Apps live outside core code under `~/.paddelino/apps`.
- Agent-authored app changes happen through drafts, not live mutation.
- User approval is required before promotion.
- Git tracks every promoted app version.
- App Verifier is a first-class system capability.
- Weather is the blueprint app proving the platform end to end.
