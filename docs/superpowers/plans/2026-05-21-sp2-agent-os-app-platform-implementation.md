# SP2 Agent OS App Platform Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Agent OS App Platform v1 to `paddelino`: dynamic git-tracked apps, app-aware delegation, draft/preview/promotion workflow, App Verifier, minimal display client, and a weather blueprint app.

**Architecture:** SP2 adds an app subsystem next to SP1's `AgentManager`: `AppWorkspace` owns `~/.paddelino/apps`, `AppRegistry` loads live apps, `AppRuntime` invokes app tools/loaders/jobs, `AppChangeManager` controls drafts and promotion, `AppVerifier` validates apps, and `DisplayServer` exposes a minimal local UI. Apps live outside core code, but the app contract and runtime live in the core `paddelino` repo.

**Tech Stack:** TypeScript, Bun, `bun test`, TypeBox for schemas, Node `fs/path/child_process`, Bun HTTP/WebSocket server, existing pi-agent-core tool wiring, Open-Meteo HTTP API for weather.

**Implementation directory:** `/mnt/Shared/Code/projects/paddelino/`.

**Reference spec:** `/mnt/Shared/Code/projects/claude-telegram-bot/docs/superpowers/specs/2026-05-21-sp2-agent-os-app-platform-design.md`.

**Reading the plan:** Tasks are numbered T01-T26 across 8 phases. Each task is one focused commit. Follow TDD for logic tasks. Commands assume `cd /mnt/Shared/Code/projects/paddelino`.

---

## File Structure

Create these core units in `paddelino`:

```text
src/apps/
  types.ts                 # App contract, structured UI, health, verifier report types
  define-app.ts            # defineApp() helper and app-id validation
  ui-schema.ts             # structured UI validators and smoke-render helpers
  permissions.ts           # app permission validation and runtime checks
  workspace.ts             # ~/.paddelino/apps initialization, git helpers, safe paths
  registry.ts              # live/preview app loading, health tracking, capability catalog
  runtime.ts               # app tool/invoke/loader/job execution facade
  verifier.ts              # App Verifier pipeline and diagnostics
  change-manager.ts        # create draft, validate, preview, promote, rollback
  tools.ts                 # core agent tools for list_apps, invoke_app, app draft flow, display actions
  weather/
    provider.ts            # Open-Meteo provider and mockable provider interface
    app.ts                 # weather blueprint app definition
    types.ts               # weather-specific data types
src/display/
  state.ts                 # DisplayState and contextual restore logic
  server.ts                # Bun HTTP + WebSocket display server
  render.ts                # minimal HTML renderer for dashboard/app/preview/error screens
src/agent/router/
  app-routing.ts           # deterministic app route scoring and delegation helpers
tests/unit/apps/
  define-app.test.ts
  ui-schema.test.ts
  permissions.test.ts
  workspace.test.ts
  verifier.test.ts
  registry.test.ts
  runtime.test.ts
  change-manager.test.ts
  weather.test.ts
tests/unit/display/
  state.test.ts
  render.test.ts
tests/integration/apps/
  app-lifecycle.test.ts
  display-actions.test.ts
```

Modify these existing files:

```text
src/shared/types.ts        # Add app/display config fields
src/config.ts              # Load PADDELINO_APPS_DIR, PADDELINO_DISPLAY_PORT, weather defaults
src/agent/identity.ts      # Include app catalog/delegation guidance in system prompt path
src/agent/manager.ts       # Register app-provided and core app tools with agents
src/agent/router/router.ts # Include app route hints in routing decision path
src/index.ts               # Wire AppWorkspace, AppRegistry, AppRuntime, DisplayServer
.env.example               # Document SP2 env vars
README.md                  # Add short SP2 usage section
```

---

## Phase A - App Contract And Validation (T01-T04)

### T01: App contract types and `defineApp()` helper

**Files:**
- Create: `src/apps/types.ts`
- Create: `src/apps/define-app.ts`
- Test: `tests/unit/apps/define-app.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/unit/apps/define-app.test.ts
import { describe, expect, test } from "bun:test";
import { defineApp, isValidAppId } from "../../../src/apps/define-app";

describe("defineApp", () => {
  test("accepts a minimal valid app", () => {
    const app = defineApp({
      appId: "weather",
      name: "Weather",
      version: "0.1.0",
      description: "Current weather and forecasts.",
      capabilities: { summary: "Shows weather.", examples: ["weather today"] },
      permissions: ["display"],
      display: {
        card: async () => ({ kind: "card", title: "Weather", blocks: [] }),
        screen: async () => ({ kind: "screen", title: "Weather", blocks: [] }),
      },
      healthCheck: async () => ({ status: "ok", message: "ready" }),
    });

    expect(app.appId).toBe("weather");
  });

  test("validates app ids", () => {
    expect(isValidAppId("weather")).toBe(true);
    expect(isValidAppId("weather-alerts")).toBe(true);
    expect(isValidAppId("Weather")).toBe(false);
    expect(isValidAppId("../weather")).toBe(false);
    expect(isValidAppId("weather alerts")).toBe(false);
  });

  test("rejects invalid app ids in defineApp", () => {
    expect(() => defineApp({
      appId: "../bad",
      name: "Bad",
      version: "0.1.0",
      description: "Bad app.",
      capabilities: { summary: "Bad.", examples: [] },
      permissions: ["display"],
      display: {
        card: async () => ({ kind: "card", title: "Bad", blocks: [] }),
        screen: async () => ({ kind: "screen", title: "Bad", blocks: [] }),
      },
      healthCheck: async () => ({ status: "ok", message: "ready" }),
    })).toThrow("Invalid appId");
  });
});
```

- [ ] **Step 2: Run the failing tests**

Run: `bun test tests/unit/apps/define-app.test.ts`

Expected: FAIL with module not found for `src/apps/define-app`.

- [ ] **Step 3: Implement contract types**

```ts
// src/apps/types.ts
export type AppPermission = "display" | "tool" | "job" | "network:http";

export type AppHealthStatus = "ok" | "warning" | "error";

export interface AppHealth {
  status: AppHealthStatus;
  message: string;
  details?: Record<string, unknown>;
}

export type UiBlock =
  | { type: "text"; text: string }
  | { type: "section"; title: string; blocks: UiBlock[] }
  | { type: "keyValue"; items: Array<{ key: string; value: string }> }
  | { type: "list"; items: string[] }
  | { type: "table"; columns: string[]; rows: string[][] }
  | { type: "button"; label: string; action: string; payload?: Record<string, unknown> }
  | { type: "status"; status: AppHealthStatus; message: string }
  | { type: "error"; message: string };

export interface AppCard {
  kind: "card";
  title: string;
  subtitle?: string;
  blocks: UiBlock[];
  actions?: Array<{ label: string; action: string; payload?: Record<string, unknown> }>;
}

export interface AppScreen {
  kind: "screen";
  title: string;
  blocks: UiBlock[];
  actions?: Array<{ label: string; action: string; payload?: Record<string, unknown> }>;
}

export interface AppContext {
  appId: string;
  home: string;
  appPath: string;
  mode: "live" | "preview";
  signal?: AbortSignal;
}

export interface AppToolDefinition {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
  run(input: unknown, context: AppContext): Promise<unknown>;
}

export interface AppRouteHint {
  intent: string;
  examples: string[];
  target: string;
  openDisplay?: boolean;
}

export interface AppJobDefinition {
  name: string;
  intervalMs: number;
  run(context: AppContext): Promise<void>;
}

export interface AppDefinition {
  appId: string;
  name: string;
  version: string;
  description: string;
  capabilities: { summary: string; examples: string[] };
  permissions: AppPermission[];
  display: {
    card(context: AppContext): Promise<AppCard>;
    screen(context: AppContext): Promise<AppScreen>;
  };
  healthCheck(context: AppContext): Promise<AppHealth>;
  tools?: Record<string, AppToolDefinition>;
  routing?: AppRouteHint[];
  invoke?(input: { prompt: string; channel: "telegram" | "display"; userId?: string }, context: AppContext): Promise<unknown>;
  loaders?: Record<string, (input: unknown, context: AppContext) => Promise<unknown>>;
  jobs?: AppJobDefinition[];
  actions?: Record<string, (payload: unknown, context: AppContext) => Promise<unknown>>;
  tests?: Array<(context: AppContext) => Promise<void>>;
}
```

- [ ] **Step 4: Implement `defineApp()`**

```ts
// src/apps/define-app.ts
import type { AppDefinition } from "./types";

const APP_ID_PATTERN = /^[a-z][a-z0-9-]{1,62}[a-z0-9]$/;

export function isValidAppId(appId: string): boolean {
  return APP_ID_PATTERN.test(appId);
}

export function defineApp<T extends AppDefinition>(app: T): T {
  if (!isValidAppId(app.appId)) {
    throw new Error(`Invalid appId: ${app.appId}`);
  }
  if (!app.capabilities.summary.trim()) {
    throw new Error(`App ${app.appId} must include a capability summary`);
  }
  return app;
}
```

- [ ] **Step 5: Verify and commit**

Run: `bun test tests/unit/apps/define-app.test.ts && bun run typecheck`


Expected: PASS and typecheck exits 0.

```bash
git add src/apps/types.ts src/apps/define-app.ts tests/unit/apps/define-app.test.ts
git commit -m "Add app contract and defineApp helper"
```

### T02: Structured UI validation

**Files:**
- Create: `src/apps/ui-schema.ts`
- Test: `tests/unit/apps/ui-schema.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/unit/apps/ui-schema.test.ts
import { describe, expect, test } from "bun:test";
import { validateCard, validateScreen } from "../../../src/apps/ui-schema";

describe("structured UI validation", () => {
  test("accepts supported card blocks", () => {
    expect(validateCard({
      kind: "card",
      title: "Weather",
      blocks: [
        { type: "text", text: "18 C and cloudy" },
        { type: "keyValue", items: [{ key: "Rain", value: "20%" }] },
      ],
    }).ok).toBe(true);
  });

  test("rejects unsupported block types", () => {
    const result = validateScreen({
      kind: "screen",
      title: "Bad",
      blocks: [{ type: "customReactComponent", name: "X" } as any],
    });
    expect(result.ok).toBe(false);
    expect(result.errors[0]).toContain("unsupported block type");
  });
});
```

- [ ] **Step 2: Run the failing tests**

Run: `bun test tests/unit/apps/ui-schema.test.ts`

Expected: FAIL with module not found for `src/apps/ui-schema`.

- [ ] **Step 3: Implement UI validation**

```ts
// src/apps/ui-schema.ts
import type { AppCard, AppScreen, UiBlock } from "./types";

export interface ValidationResult {
  ok: boolean;
  errors: string[];
}

const BLOCK_TYPES = new Set(["text", "section", "keyValue", "list", "table", "button", "status", "error"]);

export function validateCard(card: AppCard): ValidationResult {
  return validateBlocks(card.blocks, `card:${card.title}`);
}

export function validateScreen(screen: AppScreen): ValidationResult {
  return validateBlocks(screen.blocks, `screen:${screen.title}`);
}

export function validateBlocks(blocks: UiBlock[], path: string): ValidationResult {
  const errors: string[] = [];
  blocks.forEach((block, index) => validateBlock(block, `${path}.blocks[${index}]`, errors));
  return { ok: errors.length === 0, errors };
}

function validateBlock(block: UiBlock, path: string, errors: string[]): void {
  const type = (block as any).type;
  if (!BLOCK_TYPES.has(type)) {
    errors.push(`${path}: unsupported block type ${String(type)}`);
    return;
  }
  if (type === "section") {
    const childBlocks = (block as any).blocks;
    if (!Array.isArray(childBlocks)) {
      errors.push(`${path}: section blocks must be an array`);
      return;
    }
    childBlocks.forEach((child, index) => validateBlock(child, `${path}.blocks[${index}]`, errors));
  }
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/ui-schema.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/ui-schema.ts tests/unit/apps/ui-schema.test.ts
git commit -m "Add structured UI validation"
```

### T03: Permission validation

**Files:**
- Create: `src/apps/permissions.ts`
- Test: `tests/unit/apps/permissions.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/unit/apps/permissions.test.ts
import { describe, expect, test } from "bun:test";
import { assertPermission, validatePermissionConsistency } from "../../../src/apps/permissions";
import { defineApp } from "../../../src/apps/define-app";

const baseApp = defineApp({
  appId: "demo",
  name: "Demo",
  version: "0.1.0",
  description: "Demo app.",
  capabilities: { summary: "Demo.", examples: [] },
  permissions: ["display"],
  display: {
    card: async () => ({ kind: "card", title: "Demo", blocks: [] }),
    screen: async () => ({ kind: "screen", title: "Demo", blocks: [] }),
  },
  healthCheck: async () => ({ status: "ok", message: "ready" }),
});

describe("app permissions", () => {
  test("throws when a permission is missing", () => {
    expect(() => assertPermission(baseApp, "tool")).toThrow("missing permission tool");
  });

  test("reports tools without tool permission", () => {
    const app = { ...baseApp, tools: { ping: { name: "ping", description: "Ping", inputSchema: {}, run: async () => "pong" } } };
    const result = validatePermissionConsistency(app);
    expect(result.ok).toBe(false);
    expect(result.errors[0]).toContain("declares tools but lacks tool permission");
  });
});
```

- [ ] **Step 2: Run the failing tests**

Run: `bun test tests/unit/apps/permissions.test.ts`

Expected: FAIL with module not found for `src/apps/permissions`.

- [ ] **Step 3: Implement permissions**

```ts
// src/apps/permissions.ts
import type { AppDefinition, AppPermission } from "./types";
import type { ValidationResult } from "./ui-schema";

export function assertPermission(app: AppDefinition, permission: AppPermission): void {
  if (!app.permissions.includes(permission)) {
    throw new Error(`App ${app.appId} missing permission ${permission}`);
  }
}

export function validatePermissionConsistency(app: AppDefinition): ValidationResult {
  const errors: string[] = [];
  if (app.tools && !app.permissions.includes("tool")) {
    errors.push(`App ${app.appId} declares tools but lacks tool permission`);
  }
  if (app.jobs && app.jobs.length > 0 && !app.permissions.includes("job")) {
    errors.push(`App ${app.appId} declares jobs but lacks job permission`);
  }
  return { ok: errors.length === 0, errors };
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/permissions.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/permissions.ts tests/unit/apps/permissions.test.ts
git commit -m "Add app permission checks"
```

### T04: SP2 config fields

**Files:**
- Modify: `src/shared/types.ts`
- Modify: `src/config.ts`
- Modify: `.env.example`
- Test: `tests/unit/config.test.ts`

- [ ] **Step 1: Add failing config tests**

Append to `tests/unit/config.test.ts`:

```ts
test("loads SP2 app and display config", () => {
  const config = loadConfig({
    TELEGRAM_BOT_TOKEN: "token",
    TELEGRAM_ALLOWED_USERS: "123",
    ANTHROPIC_API_KEY: "key",
    PADDELINO_HOME: "/tmp/paddelino-test",
    PADDELINO_APPS_DIR: "/tmp/paddelino-apps",
    PADDELINO_DISPLAY_PORT: "4100",
    PADDELINO_WEATHER_LATITUDE: "48.2082",
    PADDELINO_WEATHER_LONGITUDE: "16.3738",
    PADDELINO_WEATHER_LABEL: "Vienna",
  });

  expect(config.paths.apps).toBe("/tmp/paddelino-apps");
  expect(config.display.port).toBe(4100);
  expect(config.weather.defaultLocation).toEqual({ latitude: 48.2082, longitude: 16.3738, label: "Vienna" });
});
```

- [ ] **Step 2: Run the failing test**

Run: `bun test tests/unit/config.test.ts`

Expected: FAIL because `paths.apps`, `display`, and `weather` do not exist.

- [ ] **Step 3: Update shared config type**

Add these fields to `AppConfig` in `src/shared/types.ts`:

```ts
  paths: {
    home: string;
    apps: string;
  };
  display: {
    port: number;
  };
  weather: {
    defaultLocation?: {
      latitude: number;
      longitude: number;
      label: string;
    };
  };
```

- [ ] **Step 4: Update `loadConfig()`**

Replace the returned `paths` part in `src/config.ts` with:

```ts
  const home = env.PADDELINO_HOME ?? join(homedir(), ".paddelino");
  const apps = env.PADDELINO_APPS_DIR ?? join(home, "apps");
  const displayPort = Number(env.PADDELINO_DISPLAY_PORT ?? "4123");

  const weatherLat = env.PADDELINO_WEATHER_LATITUDE;
  const weatherLon = env.PADDELINO_WEATHER_LONGITUDE;
  const weatherLabel = env.PADDELINO_WEATHER_LABEL ?? "Home";
  const defaultLocation = weatherLat && weatherLon
    ? { latitude: Number(weatherLat), longitude: Number(weatherLon), label: weatherLabel }
    : undefined;
```

Return these fields:

```ts
    paths: { home, apps },
    display: { port: displayPort },
    weather: { defaultLocation },
```

- [ ] **Step 5: Document env vars**

Add to `.env.example`:

```bash
# SP2 app platform
# PADDELINO_APPS_DIR=~/.paddelino/apps
# PADDELINO_DISPLAY_PORT=4123
# PADDELINO_WEATHER_LATITUDE=
# PADDELINO_WEATHER_LONGITUDE=
# PADDELINO_WEATHER_LABEL=Home
```

- [ ] **Step 6: Verify and commit**

Run: `bun test tests/unit/config.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/shared/types.ts src/config.ts .env.example tests/unit/config.test.ts
git commit -m "Add SP2 app and display configuration"
```

---

## Phase B - Workspace And App Verifier (T05-T08)

### T05: Git-backed app workspace

**Files:**
- Create: `src/apps/workspace.ts`
- Test: `tests/unit/apps/workspace.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/unit/apps/workspace.test.ts
import { afterEach, beforeEach, describe, expect, test } from "bun:test";
import { existsSync, mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { AppWorkspace } from "../../../src/apps/workspace";

let root: string;

beforeEach(() => { root = mkdtempSync(join(tmpdir(), "paddelino-apps-")); });
afterEach(() => { rmSync(root, { recursive: true, force: true }); });

describe("AppWorkspace", () => {
  test("initializes git-backed live and draft directories", async () => {
    const workspace = new AppWorkspace(root);
    await workspace.init();

    expect(existsSync(join(root, ".git"))).toBe(true);
    expect(existsSync(join(root, "live"))).toBe(true);
    expect(existsSync(join(root, "drafts"))).toBe(true);
  });

  test("rejects unsafe app ids", () => {
    const workspace = new AppWorkspace(root);
    expect(() => workspace.liveAppPath("../bad")).toThrow("Invalid appId");
  });
});
```

- [ ] **Step 2: Run the failing tests**

Run: `bun test tests/unit/apps/workspace.test.ts`

Expected: FAIL with module not found.

- [ ] **Step 3: Implement workspace**

```ts
// src/apps/workspace.ts
import { existsSync, mkdirSync, writeFileSync } from "node:fs";
import { join } from "node:path";
import { spawnSync } from "node:child_process";
import { isValidAppId } from "./define-app";

export class AppWorkspace {
  constructor(public readonly root: string) {}

  async init(): Promise<void> {
    mkdirSync(this.root, { recursive: true });
    mkdirSync(join(this.root, "live"), { recursive: true });
    mkdirSync(join(this.root, "drafts"), { recursive: true });
    if (!existsSync(join(this.root, ".git"))) {
      this.git(["init", "-b", "main"]);
      writeFileSync(join(this.root, ".gitignore"), "node_modules/\n.tmp/\n", "utf8");
      this.git(["add", ".gitignore"]);
      this.git(["commit", "-m", "Initialize paddelino apps workspace"], { allowNoCommit: true });
    }
  }

  liveAppPath(appId: string): string {
    this.assertAppId(appId);
    return join(this.root, "live", appId);
  }

  draftPath(draftId: string): string {
    if (!/^[a-z0-9][a-z0-9-]{1,80}$/.test(draftId)) throw new Error(`Invalid draftId: ${draftId}`);
    return join(this.root, "drafts", draftId);
  }

  git(args: string[], options: { allowNoCommit?: boolean } = {}): string {
    const result = spawnSync("git", args, { cwd: this.root, encoding: "utf8" });
    if (result.status !== 0) {
      const output = `${result.stdout}${result.stderr}`;
      if (options.allowNoCommit && output.includes("nothing to commit")) return output;
      throw new Error(`git ${args.join(" ")} failed: ${output}`);
    }
    return result.stdout;
  }

  statusPorcelain(): string {
    return this.git(["status", "--porcelain"]);
  }

  private assertAppId(appId: string): void {
    if (!isValidAppId(appId)) throw new Error(`Invalid appId: ${appId}`);
  }
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/workspace.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/workspace.ts tests/unit/apps/workspace.test.ts
git commit -m "Add git-backed app workspace"
```

### T06: App Verifier pipeline

**Files:**
- Create: `src/apps/verifier.ts`
- Test: `tests/unit/apps/verifier.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/unit/apps/verifier.test.ts
import { describe, expect, test } from "bun:test";
import { verifyApp } from "../../../src/apps/verifier";
import { defineApp } from "../../../src/apps/define-app";

describe("AppVerifier", () => {
  test("passes a valid app", async () => {
    const app = defineApp({
      appId: "demo",
      name: "Demo",
      version: "0.1.0",
      description: "Demo app.",
      capabilities: { summary: "Demo.", examples: ["demo"] },
      permissions: ["display"],
      display: {
        card: async () => ({ kind: "card", title: "Demo", blocks: [{ type: "text", text: "ok" }] }),
        screen: async () => ({ kind: "screen", title: "Demo", blocks: [] }),
      },
      healthCheck: async () => ({ status: "ok", message: "ready" }),
    });

    const report = await verifyApp(app, { appId: "demo", home: "/tmp", appPath: "/tmp/demo", mode: "live" });
    expect(report.ok).toBe(true);
    expect(report.stages.map(s => s.name)).toContain("display smoke render");
  });

  test("returns guidance for missing capability summary", async () => {
    const app = defineApp({
      appId: "demo",
      name: "Demo",
      version: "0.1.0",
      description: "Demo app.",
      capabilities: { summary: "x", examples: [] },
      permissions: ["display"],
      display: {
        card: async () => ({ kind: "card", title: "Demo", blocks: [] }),
        screen: async () => ({ kind: "screen", title: "Demo", blocks: [] }),
      },
      healthCheck: async () => ({ status: "ok", message: "ready" }),
    });
    app.capabilities.summary = "";

    const report = await verifyApp(app, { appId: "demo", home: "/tmp", appPath: "/tmp/demo", mode: "live" });
    expect(report.ok).toBe(false);
    expect(report.guidance.join("\n")).toContain("missing a capability summary");
  });
});
```

- [ ] **Step 2: Run the failing tests**

Run: `bun test tests/unit/apps/verifier.test.ts`

Expected: FAIL with module not found.

- [ ] **Step 3: Implement verifier**

```ts
// src/apps/verifier.ts
import type { AppContext, AppDefinition } from "./types";
import { validateCard, validateScreen } from "./ui-schema";
import { validatePermissionConsistency } from "./permissions";
import { isValidAppId } from "./define-app";

export interface VerificationStage {
  name: string;
  ok: boolean;
  errors: string[];
}

export interface VerificationReport {
  ok: boolean;
  appId: string;
  stages: VerificationStage[];
  guidance: string[];
}

export async function verifyApp(app: AppDefinition, context: AppContext): Promise<VerificationReport> {
  const stages: VerificationStage[] = [];
  const guidance: string[] = [];

  addStage(stages, "manifest/schema", isValidAppId(app.appId), isValidAppId(app.appId) ? [] : [`Invalid appId: ${app.appId}`]);

  const capabilityErrors = app.capabilities.summary.trim()
    ? []
    : ["App is missing a capability summary"];
  if (capabilityErrors.length > 0) guidance.push("Your app is missing a capability summary, so the main agent will not know when to delegate to it.");
  addStage(stages, "routing/capability summary", capabilityErrors.length === 0, capabilityErrors);

  const permissions = validatePermissionConsistency(app);
  addStage(stages, "permission consistency", permissions.ok, permissions.errors);

  const card = await app.display.card(context);
  const screen = await app.display.screen(context);
  const cardResult = validateCard(card);
  const screenResult = validateScreen(screen);
  addStage(stages, "display smoke render", cardResult.ok && screenResult.ok, [...cardResult.errors, ...screenResult.errors]);

  const health = await app.healthCheck(context);
  addStage(stages, "health check", health.status !== "error", health.status === "error" ? [health.message] : []);

  if (app.tests) {
    const testErrors: string[] = [];
    for (const test of app.tests) {
      try { await test(context); } catch (err) { testErrors.push((err as Error).message); }
    }
    addStage(stages, "app-specific tests", testErrors.length === 0, testErrors);
  }

  return { ok: stages.every(stage => stage.ok), appId: app.appId, stages, guidance };
}

function addStage(stages: VerificationStage[], name: string, ok: boolean, errors: string[]): void {
  stages.push({ name, ok, errors });
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/verifier.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/verifier.ts tests/unit/apps/verifier.test.ts
git commit -m "Add App Verifier pipeline"
```

### T07: Dynamic app module loading and registry

**Files:**
- Create: `src/apps/registry.ts`
- Test: `tests/unit/apps/registry.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/unit/apps/registry.test.ts
import { afterEach, beforeEach, describe, expect, test } from "bun:test";
import { mkdirSync, mkdtempSync, rmSync, writeFileSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { AppWorkspace } from "../../../src/apps/workspace";
import { AppRegistry } from "../../../src/apps/registry";

let root: string;

beforeEach(() => { root = mkdtempSync(join(tmpdir(), "paddelino-registry-")); });
afterEach(() => { rmSync(root, { recursive: true, force: true }); });

describe("AppRegistry", () => {
  test("loads live apps and exposes catalog", async () => {
    const workspace = new AppWorkspace(root);
    await workspace.init();
    const appDir = workspace.liveAppPath("demo");
    mkdirSync(appDir, { recursive: true });
    writeFileSync(join(appDir, "app.ts"), `
      import { defineApp } from "${process.cwd()}/src/apps/define-app";
      export default defineApp({
        appId: "demo",
        name: "Demo",
        version: "0.1.0",
        description: "Demo app.",
        capabilities: { summary: "Demo capability.", examples: ["demo"] },
        permissions: ["display"],
        display: {
          card: async () => ({ kind: "card", title: "Demo", blocks: [] }),
          screen: async () => ({ kind: "screen", title: "Demo", blocks: [] }),
        },
        healthCheck: async () => ({ status: "ok", message: "ready" }),
      });
    `);

    const registry = new AppRegistry(workspace, "/tmp/paddelino-home");
    await registry.reloadLiveApps();

    expect(registry.getApp("demo")?.definition.name).toBe("Demo");
    expect(registry.catalog()[0]).toMatchObject({ appId: "demo", name: "Demo" });
  });
});
```

- [ ] **Step 2: Run the failing tests**

Run: `bun test tests/unit/apps/registry.test.ts`

Expected: FAIL with module not found.

- [ ] **Step 3: Implement registry**

```ts
// src/apps/registry.ts
import { existsSync, readdirSync } from "node:fs";
import { join } from "node:path";
import type { AppContext, AppDefinition, AppHealth } from "./types";
import type { AppWorkspace } from "./workspace";
import { verifyApp, type VerificationReport } from "./verifier";

export interface LoadedApp {
  definition: AppDefinition;
  path: string;
  mode: "live" | "preview";
  health: AppHealth;
  verification: VerificationReport;
}

export interface AppCatalogEntry {
  appId: string;
  name: string;
  description: string;
  capabilities: AppDefinition["capabilities"];
  permissions: AppDefinition["permissions"];
  routes: AppDefinition["routing"];
}

export class AppRegistry {
  private liveApps = new Map<string, LoadedApp>();
  private previewApps = new Map<string, LoadedApp>();

  constructor(private readonly workspace: AppWorkspace, private readonly home: string) {}

  async reloadLiveApps(): Promise<void> {
    const liveRoot = join(this.workspace.root, "live");
    const next = new Map<string, LoadedApp>();
    if (!existsSync(liveRoot)) {
      this.liveApps = next;
      return;
    }
    for (const entry of readdirSync(liveRoot, { withFileTypes: true })) {
      if (!entry.isDirectory()) continue;
      const loaded = await this.loadAppFromPath(join(liveRoot, entry.name), "live");
      if (loaded.verification.ok) next.set(loaded.definition.appId, loaded);
    }
    this.liveApps = next;
  }

  async loadPreview(draftId: string, appPath: string): Promise<LoadedApp> {
    const loaded = await this.loadAppFromPath(appPath, "preview");
    if (loaded.verification.ok) this.previewApps.set(draftId, loaded);
    return loaded;
  }

  getApp(appId: string): LoadedApp | undefined {
    return this.liveApps.get(appId);
  }

  getPreview(draftId: string): LoadedApp | undefined {
    return this.previewApps.get(draftId);
  }

  catalog(): AppCatalogEntry[] {
    return [...this.liveApps.values()].map(({ definition }) => ({
      appId: definition.appId,
      name: definition.name,
      description: definition.description,
      capabilities: definition.capabilities,
      permissions: definition.permissions,
      routes: definition.routing,
    }));
  }

  private async loadAppFromPath(appPath: string, mode: "live" | "preview"): Promise<LoadedApp> {
    const modulePath = join(appPath, "app.ts");
    const mod = await import(`${modulePath}?t=${Date.now()}-${Math.random()}`);
    const definition = mod.default as AppDefinition;
    const context: AppContext = { appId: definition.appId, home: this.home, appPath, mode };
    const verification = await verifyApp(definition, context);
    const health = verification.ok
      ? await definition.healthCheck(context)
      : { status: "error" as const, message: verification.stages.flatMap(s => s.errors).join("; ") };
    return { definition, path: appPath, mode, health, verification };
  }
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/registry.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/registry.ts tests/unit/apps/registry.test.ts
git commit -m "Add dynamic app registry"
```

### T08: App Runtime invocation facade

**Files:**
- Create: `src/apps/runtime.ts`
- Test: `tests/unit/apps/runtime.test.ts`

- [ ] **Step 1: Write failing runtime tests**

```ts
// tests/unit/apps/runtime.test.ts
import { describe, expect, test } from "bun:test";
import { AppRuntime } from "../../../src/apps/runtime";
import type { LoadedApp } from "../../../src/apps/registry";
import { defineApp } from "../../../src/apps/define-app";

const app = defineApp({
  appId: "demo",
  name: "Demo",
  version: "0.1.0",
  description: "Demo app.",
  capabilities: { summary: "Demo.", examples: ["demo"] },
  permissions: ["display", "tool"],
  display: {
    card: async () => ({ kind: "card", title: "Demo", blocks: [] }),
    screen: async () => ({ kind: "screen", title: "Demo", blocks: [] }),
  },
  healthCheck: async () => ({ status: "ok", message: "ready" }),
  tools: {
    ping: { name: "ping", description: "Ping", inputSchema: {}, run: async () => ({ pong: true }) },
  },
  invoke: async ({ prompt }) => ({ answer: `handled ${prompt}` }),
});

const loaded: LoadedApp = {
  definition: app,
  path: "/tmp/demo",
  mode: "live",
  health: { status: "ok", message: "ready" },
  verification: { ok: true, appId: "demo", stages: [], guidance: [] },
};

describe("AppRuntime", () => {
  test("invokes app tools", async () => {
    const runtime = new AppRuntime({ getApp: () => loaded } as any, "/tmp/home");
    await expect(runtime.runTool("demo", "ping", {})).resolves.toEqual({ pong: true });
  });

  test("delegates prompts to app invoke endpoint", async () => {
    const runtime = new AppRuntime({ getApp: () => loaded } as any, "/tmp/home");
    await expect(runtime.invokeApp("demo", { prompt: "hello", channel: "telegram" })).resolves.toEqual({ answer: "handled hello" });
  });
});
```

- [ ] **Step 2: Run failing tests**

Run: `bun test tests/unit/apps/runtime.test.ts`

Expected: FAIL with module not found.

- [ ] **Step 3: Implement runtime**

```ts
// src/apps/runtime.ts
import type { AppContext } from "./types";
import type { AppRegistry } from "./registry";
import { assertPermission } from "./permissions";

export class AppRuntime {
  constructor(private readonly registry: AppRegistry, private readonly home: string) {}

  async runTool(appId: string, toolName: string, input: unknown): Promise<unknown> {
    const app = this.requireApp(appId);
    assertPermission(app.definition, "tool");
    const tool = app.definition.tools?.[toolName];
    if (!tool) throw new Error(`App ${appId} has no tool ${toolName}`);
    return tool.run(input, this.context(appId, app.path, app.mode));
  }

  async invokeApp(appId: string, input: { prompt: string; channel: "telegram" | "display"; userId?: string }): Promise<unknown> {
    const app = this.requireApp(appId);
    if (!app.definition.invoke) throw new Error(`App ${appId} has no invoke endpoint`);
    return app.definition.invoke(input, this.context(appId, app.path, app.mode));
  }

  async renderCard(appId: string): Promise<unknown> {
    const app = this.requireApp(appId);
    assertPermission(app.definition, "display");
    return app.definition.display.card(this.context(appId, app.path, app.mode));
  }

  async renderScreen(appId: string): Promise<unknown> {
    const app = this.requireApp(appId);
    assertPermission(app.definition, "display");
    return app.definition.display.screen(this.context(appId, app.path, app.mode));
  }

  private requireApp(appId: string) {
    const app = this.registry.getApp(appId);
    if (!app) throw new Error(`Unknown app: ${appId}`);
    return app;
  }

  private context(appId: string, appPath: string, mode: "live" | "preview"): AppContext {
    return { appId, home: this.home, appPath, mode };
  }
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/runtime.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/runtime.ts tests/unit/apps/runtime.test.ts
git commit -m "Add app runtime invocation facade"
```

---

## Phase C - Weather Blueprint App (T09-T11)

### T09: Weather provider

**Files:**
- Create: `src/apps/weather/types.ts`
- Create: `src/apps/weather/provider.ts`
- Test: `tests/unit/apps/weather.test.ts`

- [ ] **Step 1: Write failing provider tests**

```ts
// tests/unit/apps/weather.test.ts
import { describe, expect, test } from "bun:test";
import { createOpenMeteoProvider } from "../../../src/apps/weather/provider";

describe("weather provider", () => {
  test("maps Open-Meteo response to app weather data", async () => {
    const provider = createOpenMeteoProvider(async () => new Response(JSON.stringify({
      current: { temperature_2m: 18.4, weather_code: 3, precipitation_probability: 20 },
      daily: { time: ["2026-05-21"], temperature_2m_max: [21], temperature_2m_min: [12], precipitation_probability_max: [30] },
    })) as any);

    const weather = await provider.getForecast({ latitude: 48.2082, longitude: 16.3738, label: "Vienna" });
    expect(weather.location.label).toBe("Vienna");
    expect(weather.current.temperatureC).toBe(18.4);
    expect(weather.current.condition).toBe("Cloudy");
    expect(weather.daily[0].rainChancePercent).toBe(30);
  });
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/unit/apps/weather.test.ts`

Expected: FAIL with module not found.

- [ ] **Step 3: Implement weather types**

```ts
// src/apps/weather/types.ts
export interface WeatherLocation {
  latitude: number;
  longitude: number;
  label: string;
}

export interface WeatherSnapshot {
  location: WeatherLocation;
  current: {
    temperatureC: number;
    condition: string;
    rainChancePercent?: number;
  };
  daily: Array<{
    date: string;
    minC: number;
    maxC: number;
    rainChancePercent: number;
  }>;
  updatedAt: string;
}

export interface WeatherProvider {
  getForecast(location: WeatherLocation): Promise<WeatherSnapshot>;
}
```

- [ ] **Step 4: Implement Open-Meteo provider**

```ts
// src/apps/weather/provider.ts
import type { WeatherLocation, WeatherProvider, WeatherSnapshot } from "./types";

type FetchLike = typeof fetch;

export function createOpenMeteoProvider(fetchImpl: FetchLike = fetch): WeatherProvider {
  return {
    async getForecast(location: WeatherLocation): Promise<WeatherSnapshot> {
      const url = new URL("https://api.open-meteo.com/v1/forecast");
      url.searchParams.set("latitude", String(location.latitude));
      url.searchParams.set("longitude", String(location.longitude));
      url.searchParams.set("current", "temperature_2m,weather_code,precipitation_probability");
      url.searchParams.set("daily", "temperature_2m_max,temperature_2m_min,precipitation_probability_max");
      url.searchParams.set("timezone", "auto");

      const response = await fetchImpl(url);
      if (!response.ok) throw new Error(`Open-Meteo request failed: ${response.status}`);
      const data = await response.json() as any;
      return {
        location,
        current: {
          temperatureC: data.current.temperature_2m,
          condition: weatherCodeLabel(data.current.weather_code),
          rainChancePercent: data.current.precipitation_probability,
        },
        daily: (data.daily.time as string[]).map((date, index) => ({
          date,
          minC: data.daily.temperature_2m_min[index],
          maxC: data.daily.temperature_2m_max[index],
          rainChancePercent: data.daily.precipitation_probability_max[index],
        })),
        updatedAt: new Date().toISOString(),
      };
    },
  };
}

function weatherCodeLabel(code: number): string {
  if (code === 0) return "Clear";
  if (code <= 3) return "Cloudy";
  if (code >= 51 && code <= 67) return "Rain";
  if (code >= 71 && code <= 77) return "Snow";
  if (code >= 95) return "Thunderstorm";
  return "Unknown";
}
```

- [ ] **Step 5: Verify and commit**

Run: `bun test tests/unit/apps/weather.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/weather/types.ts src/apps/weather/provider.ts tests/unit/apps/weather.test.ts
git commit -m "Add Open-Meteo weather provider"
```

### T10: Weather app definition

**Files:**
- Create: `src/apps/weather/app.ts`
- Modify: `tests/unit/apps/weather.test.ts`

- [ ] **Step 1: Add failing app tests**

Append to `tests/unit/apps/weather.test.ts`:

```ts
import weatherApp, { createWeatherApp } from "../../../src/apps/weather/app";

test("weather app exposes display, tools, routes, and invoke", async () => {
  expect(weatherApp.appId).toBe("weather");
  expect(weatherApp.permissions).toContain("network:http");
  expect(weatherApp.permissions).toContain("tool");
  expect(weatherApp.routing?.[0].examples).toContain("what is the weather");

  const app = createWeatherApp({
    defaultLocation: { latitude: 48.2082, longitude: 16.3738, label: "Vienna" },
    provider: { getForecast: async (location) => ({
      location,
      current: { temperatureC: 18, condition: "Cloudy", rainChancePercent: 20 },
      daily: [{ date: "2026-05-21", minC: 12, maxC: 21, rainChancePercent: 30 }],
      updatedAt: "2026-05-21T10:00:00.000Z",
    }) },
  });

  const card = await app.display.card({ appId: "weather", home: "/tmp", appPath: "/tmp/weather", mode: "live" });
  expect(card.title).toBe("Weather");
  const result = await app.invoke?.({ prompt: "will it rain", channel: "telegram" }, { appId: "weather", home: "/tmp", appPath: "/tmp/weather", mode: "live" });
  expect(String((result as any).text)).toContain("Cloudy");
});
```

- [ ] **Step 2: Run failing tests**

Run: `bun test tests/unit/apps/weather.test.ts`

Expected: FAIL with module not found for `src/apps/weather/app`.

- [ ] **Step 3: Implement weather app**

```ts
// src/apps/weather/app.ts
import { defineApp } from "../define-app";
import type { AppDefinition } from "../types";
import { createOpenMeteoProvider } from "./provider";
import type { WeatherLocation, WeatherProvider, WeatherSnapshot } from "./types";

export interface WeatherAppOptions {
  defaultLocation?: WeatherLocation;
  provider?: WeatherProvider;
}

const fallbackLocation: WeatherLocation = { latitude: 48.2082, longitude: 16.3738, label: "Vienna" };

export function createWeatherApp(options: WeatherAppOptions = {}): AppDefinition {
  const provider = options.provider ?? createOpenMeteoProvider();
  const getLocation = () => options.defaultLocation ?? fallbackLocation;
  const load = () => provider.getForecast(getLocation());

  return defineApp({
    appId: "weather",
    name: "Weather",
    version: "0.1.0",
    description: "Current weather, short forecast, and rain outlook.",
    capabilities: {
      summary: "I can show current weather, short forecasts, and rain outlooks for the configured home location.",
      examples: ["what is the weather", "will it rain today", "weather tomorrow"],
    },
    permissions: ["display", "tool", "job", "network:http"],
    display: {
      card: async () => weatherCard(await load()),
      screen: async () => weatherScreen(await load()),
    },
    healthCheck: async () => {
      try {
        await load();
        return { status: "ok", message: "Weather provider reachable" };
      } catch (err) {
        return { status: "error", message: (err as Error).message };
      }
    },
    tools: {
      get_current: {
        name: "get_current",
        description: "Get current weather for the configured location.",
        inputSchema: { type: "object", properties: {}, additionalProperties: false },
        run: async () => load(),
      },
      get_forecast: {
        name: "get_forecast",
        description: "Get short weather forecast for the configured location.",
        inputSchema: { type: "object", properties: {}, additionalProperties: false },
        run: async () => load(),
      },
    },
    routing: [{ intent: "weather", examples: ["what is the weather", "weather tomorrow", "will it rain today"], target: "weather", openDisplay: true }],
    invoke: async () => {
      const weather = await load();
      return { text: `${weather.location.label}: ${weather.current.temperatureC} C, ${weather.current.condition}, rain chance ${weather.current.rainChancePercent ?? "unknown"}%.` };
    },
    jobs: [{ name: "refresh", intervalMs: 60 * 60 * 1000, run: async () => { await load(); } }],
  });
}

function weatherCard(weather: WeatherSnapshot) {
  return {
    kind: "card" as const,
    title: "Weather",
    subtitle: weather.location.label,
    blocks: [
      { type: "text" as const, text: `${weather.current.temperatureC} C and ${weather.current.condition}` },
      { type: "keyValue" as const, items: [
        { key: "Rain", value: `${weather.current.rainChancePercent ?? "unknown"}%` },
        { key: "Updated", value: weather.updatedAt },
      ] },
    ],
  };
}

function weatherScreen(weather: WeatherSnapshot) {
  return {
    kind: "screen" as const,
    title: `Weather - ${weather.location.label}`,
    blocks: [
      { type: "text" as const, text: `${weather.current.temperatureC} C and ${weather.current.condition}` },
      { type: "table" as const, columns: ["Date", "Min", "Max", "Rain"], rows: weather.daily.map(day => [day.date, `${day.minC} C`, `${day.maxC} C`, `${day.rainChancePercent}%`]) },
    ],
  };
}

export default createWeatherApp();
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/weather.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/weather/app.ts tests/unit/apps/weather.test.ts
git commit -m "Add weather blueprint app"
```

### T11: Seed bundled weather app into app workspace

**Files:**
- Modify: `src/apps/workspace.ts`
- Test: `tests/unit/apps/workspace.test.ts`

- [ ] **Step 1: Add failing seed test**

Append to `tests/unit/apps/workspace.test.ts`:

```ts
test("seeds weather app into live workspace", async () => {
  const workspace = new AppWorkspace(root);
  await workspace.init();
  await workspace.seedBundledApp("weather", `export default { appId: "weather" };\n`);
  expect(existsSync(join(root, "live", "weather", "app.ts"))).toBe(true);
  expect(workspace.statusPorcelain()).toBe("");
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/unit/apps/workspace.test.ts`

Expected: FAIL because `seedBundledApp` does not exist.

- [ ] **Step 3: Implement seeding**

Add to `AppWorkspace`:

```ts
  async seedBundledApp(appId: string, appSource: string): Promise<void> {
    const appPath = this.liveAppPath(appId);
    mkdirSync(appPath, { recursive: true });
    const target = join(appPath, "app.ts");
    if (!existsSync(target)) {
      writeFileSync(target, appSource, "utf8");
      this.git(["add", `live/${appId}/app.ts`]);
      this.git(["commit", "-m", `Seed ${appId} app`], { allowNoCommit: true });
    }
  }
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/workspace.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/workspace.ts tests/unit/apps/workspace.test.ts
git commit -m "Seed bundled apps into workspace"
```

---

## Phase D - Routing, Tools, And Agent Integration (T12-T15)

### T12: App-aware route scoring

**Files:**
- Create: `src/agent/router/app-routing.ts`
- Test: `tests/unit/router.test.ts`

- [ ] **Step 1: Add failing routing tests**

Append to `tests/unit/router.test.ts`:

```ts
import { scoreAppRoutes } from "../../src/agent/router/app-routing";

test("scores app routes from examples", () => {
  const matches = scoreAppRoutes("will it rain today?", [{
    appId: "weather",
    name: "Weather",
    description: "Weather app",
    capabilities: { summary: "Rain and forecasts", examples: ["will it rain today"] },
    permissions: ["display", "tool"],
    routes: [{ intent: "weather", examples: ["what is the weather", "will it rain today"], target: "weather" }],
  }]);

  expect(matches[0]).toMatchObject({ appId: "weather" });
  expect(matches[0].score).toBeGreaterThan(0);
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/unit/router.test.ts`

Expected: FAIL because `app-routing` does not exist.

- [ ] **Step 3: Implement app route scorer**

```ts
// src/agent/router/app-routing.ts
import type { AppCatalogEntry } from "../../apps/registry";

export interface AppRouteScore {
  appId: string;
  score: number;
  reason: string;
  openDisplay: boolean;
}

export function scoreAppRoutes(message: string, catalog: AppCatalogEntry[]): AppRouteScore[] {
  const normalized = normalize(message);
  return catalog
    .map(app => {
      const examples = [...app.capabilities.examples, ...(app.routes ?? []).flatMap(route => route.examples)];
      const matched = examples.find(example => normalized.includes(normalize(example)) || overlap(normalized, normalize(example)) >= 2);
      return matched ? { appId: app.appId, score: matched.length, reason: `matched example: ${matched}`, openDisplay: Boolean(app.routes?.some(route => route.openDisplay)) } : null;
    })
    .filter((score): score is AppRouteScore => Boolean(score))
    .sort((a, b) => b.score - a.score);
}

function normalize(value: string): string {
  return value.toLowerCase().replace(/[^a-z0-9 ]+/g, " ").replace(/\s+/g, " ").trim();
}

function overlap(a: string, b: string): number {
  const left = new Set(a.split(" ").filter(word => word.length > 2));
  return b.split(" ").filter(word => left.has(word)).length;
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/router.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/agent/router/app-routing.ts tests/unit/router.test.ts
git commit -m "Add app-aware route scoring"
```

### T13: Core app tools

**Files:**
- Create: `src/apps/tools.ts`
- Test: `tests/unit/apps/runtime.test.ts`

- [ ] **Step 1: Add failing tool tests**

Append to `tests/unit/apps/runtime.test.ts`:

```ts
import { createAppTools } from "../../../src/apps/tools";

test("core app tools list apps and invoke apps", async () => {
  const runtime = new AppRuntime({ getApp: () => loaded, catalog: () => [{ appId: "demo", name: "Demo", description: "Demo", capabilities: app.capabilities, permissions: app.permissions, routes: [] }] } as any, "/tmp/home");
  const tools = createAppTools({ runtime, registry: { catalog: () => [{ appId: "demo", name: "Demo", description: "Demo", capabilities: app.capabilities, permissions: app.permissions, routes: [] }] } as any });

  expect(await tools.list_apps.run({}, {} as any)).toEqual([{ appId: "demo", name: "Demo", description: "Demo", capabilities: app.capabilities, permissions: app.permissions, routes: [] }]);
  expect(await tools.invoke_app.run({ appId: "demo", prompt: "hi", channel: "telegram" }, {} as any)).toEqual({ answer: "handled hi" });
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/unit/apps/runtime.test.ts`

Expected: FAIL because `src/apps/tools.ts` does not exist.

- [ ] **Step 3: Implement app tools**

```ts
// src/apps/tools.ts
import type { AppRegistry } from "./registry";
import type { AppRuntime } from "./runtime";
import type { AppToolDefinition } from "./types";

export function createAppTools(deps: { registry: AppRegistry; runtime: AppRuntime }): Record<string, AppToolDefinition> {
  return {
    list_apps: {
      name: "list_apps",
      description: "List installed apps and their capabilities.",
      inputSchema: { type: "object", properties: {}, additionalProperties: false },
      run: async () => deps.registry.catalog(),
    },
    invoke_app: {
      name: "invoke_app",
      description: "Send a natural-language prompt to an app.",
      inputSchema: {
        type: "object",
        required: ["appId", "prompt", "channel"],
        properties: {
          appId: { type: "string" },
          prompt: { type: "string" },
          channel: { enum: ["telegram", "display"] },
        },
        additionalProperties: false,
      },
      run: async (input) => {
        const payload = input as { appId: string; prompt: string; channel: "telegram" | "display"; userId?: string };
        return deps.runtime.invokeApp(payload.appId, payload);
      },
    },
  };
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/runtime.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/tools.ts tests/unit/apps/runtime.test.ts
git commit -m "Add core app agent tools"
```

### T14: Register app tools with AgentManager

**Files:**
- Modify: `src/agent/manager.ts`
- Test: `tests/unit/manager.test.ts`

- [ ] **Step 1: Inspect current tool wiring**

Read `src/agent/manager.ts` and find where default tools are created for pi agents.

- [ ] **Step 2: Add failing test for extra tools**

Add a test to `tests/unit/manager.test.ts` that constructs `AgentManager` with `extraTools` and asserts those tools are included in the factory input. If the existing tests mock `createAgent`, extend that mock assertion with a tool named `list_apps`.

Use this test shape:

```ts
test("includes extra app tools when creating agents", async () => {
  const tools = {
    list_apps: { name: "list_apps", description: "List apps", inputSchema: {}, run: async () => [] },
  };
  const manager = createTestManager({ extraTools: tools });
  await manager.getOrCreate("123");
  expect(lastCreateAgentOptions.tools.map((tool: any) => tool.name)).toContain("list_apps");
});
```

- [ ] **Step 3: Run failing test**

Run: `bun test tests/unit/manager.test.ts`

Expected: FAIL because `extraTools` is not supported.

- [ ] **Step 4: Implement `extraTools` support**

In `src/agent/manager.ts`, add an optional constructor field:

```ts
extraTools?: Record<string, AppToolDefinition>;
```

Merge it into the existing default tool list:

```ts
const tools = {
  ...getDefaultTools(...),
  ...(this.options.extraTools ?? {}),
};
```

Import `AppToolDefinition` from `../apps/types`.

- [ ] **Step 5: Verify and commit**

Run: `bun test tests/unit/manager.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/agent/manager.ts tests/unit/manager.test.ts
git commit -m "Register app tools with agents"
```

### T15: Add app catalog guidance to identity prompt

**Files:**
- Modify: `src/agent/identity.ts`
- Test: `tests/unit/identity.test.ts`

- [ ] **Step 1: Add failing identity test**

Append to `tests/unit/identity.test.ts`:

```ts
test("system prompt includes app-aware delegation guidance", () => {
  const identity = createIdentity(testHome);
  const prompt = identity.systemPrompt({
    appCatalog: [{ appId: "weather", name: "Weather", description: "Weather", capabilities: { summary: "Rain outlook", examples: ["will it rain"] }, permissions: ["display", "tool"], routes: [] }],
  });
  expect(prompt).toContain("Installed apps");
  expect(prompt).toContain("weather");
  expect(prompt).toContain("invoke_app");
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/unit/identity.test.ts`

Expected: FAIL because identity prompt does not accept app catalog.

- [ ] **Step 3: Implement app catalog prompt support**

Update `createIdentity()` so `systemPrompt()` accepts optional context:

```ts
interface SystemPromptContext {
  appCatalog?: Array<{ appId: string; name: string; description: string; capabilities: { summary: string; examples: string[] } }>;
}
```

Append this section when apps exist:

```ts
function appCatalogPrompt(appCatalog: SystemPromptContext["appCatalog"]): string {
  if (!appCatalog || appCatalog.length === 0) return "";
  const lines = appCatalog.map(app => `- ${app.appId} (${app.name}): ${app.capabilities.summary}. Examples: ${app.capabilities.examples.join(", ")}`);
  return `\n\nInstalled apps:\n${lines.join("\n")}\n\nWhen a user request fits an installed app, use list_apps or invoke_app instead of handling it as generic chat. If an app exposes a more specific tool, prefer that tool.`;
}
```

- [ ] **Step 4: Pass catalog from manager**

Where `AgentManager` builds system prompts, pass `appCatalog: this.options.appCatalog?.()` and add this optional constructor field:

```ts
appCatalog?: () => AppCatalogEntry[];
```

- [ ] **Step 5: Verify and commit**

Run: `bun test tests/unit/identity.test.ts tests/unit/manager.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/agent/identity.ts src/agent/manager.ts tests/unit/identity.test.ts tests/unit/manager.test.ts
git commit -m "Teach agents about installed apps"
```

---

## Phase E - Draft, Preview, Promotion (T16-T18)

### T16: Create and validate app drafts

**Files:**
- Create: `src/apps/change-manager.ts`
- Test: `tests/unit/apps/change-manager.test.ts`

- [ ] **Step 1: Write failing draft tests**

```ts
// tests/unit/apps/change-manager.test.ts
import { afterEach, beforeEach, describe, expect, test } from "bun:test";
import { existsSync, mkdtempSync, mkdirSync, rmSync, writeFileSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { AppWorkspace } from "../../../src/apps/workspace";
import { AppRegistry } from "../../../src/apps/registry";
import { AppChangeManager } from "../../../src/apps/change-manager";

let root: string;
beforeEach(() => { root = mkdtempSync(join(tmpdir(), "paddelino-change-")); });
afterEach(() => { rmSync(root, { recursive: true, force: true }); });

describe("AppChangeManager", () => {
  test("creates draft from live app", async () => {
    const workspace = new AppWorkspace(root);
    await workspace.init();
    const live = workspace.liveAppPath("weather");
    mkdirSync(live, { recursive: true });
    writeFileSync(join(live, "app.ts"), "export default {};\n");

    const manager = new AppChangeManager(workspace, new AppRegistry(workspace, "/tmp/home"));
    const draft = await manager.createDraft({ appId: "weather", userId: "123" });

    expect(draft.appId).toBe("weather");
    expect(existsSync(join(draft.path, "app.ts"))).toBe(true);
  });
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/unit/apps/change-manager.test.ts`

Expected: FAIL with module not found.

- [ ] **Step 3: Implement draft creation**

```ts
// src/apps/change-manager.ts
import { cpSync, existsSync, mkdirSync, rmSync } from "node:fs";
import { basename, join } from "node:path";
import type { AppRegistry } from "./registry";
import type { AppWorkspace } from "./workspace";
import type { VerificationReport } from "./verifier";

export interface AppDraft {
  draftId: string;
  appId: string;
  path: string;
  userId: string;
}

export class AppChangeManager {
  constructor(private readonly workspace: AppWorkspace, private readonly registry: AppRegistry) {}

  async createDraft(input: { appId: string; userId: string }): Promise<AppDraft> {
    const draftId = `${input.appId}-${Date.now()}`;
    const draftPath = this.workspace.draftPath(draftId);
    mkdirSync(draftPath, { recursive: true });
    const livePath = this.workspace.liveAppPath(input.appId);
    if (existsSync(livePath)) {
      cpSync(livePath, draftPath, { recursive: true });
    } else {
      mkdirSync(draftPath, { recursive: true });
    }
    this.workspace.git(["add", relativeToRoot(this.workspace.root, draftPath)]);
    this.workspace.git(["commit", "-m", `Create draft ${draftId}`], { allowNoCommit: true });
    return { draftId, appId: input.appId, path: draftPath, userId: input.userId };
  }

  async validateDraft(draft: AppDraft): Promise<VerificationReport> {
    const loaded = await this.registry.loadPreview(draft.draftId, draft.path);
    return loaded.verification;
  }

  abandonDraft(draft: AppDraft): void {
    rmSync(draft.path, { recursive: true, force: true });
    this.workspace.git(["add", "drafts"]);
    this.workspace.git(["commit", "-m", `Abandon draft ${draft.draftId}`], { allowNoCommit: true });
  }
}

function relativeToRoot(root: string, path: string): string {
  return path.slice(root.length + 1) || basename(path);
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/change-manager.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/change-manager.ts tests/unit/apps/change-manager.test.ts
git commit -m "Add app draft creation and validation"
```

### T17: Preview and promote drafts

**Files:**
- Modify: `src/apps/change-manager.ts`
- Test: `tests/unit/apps/change-manager.test.ts`

- [ ] **Step 1: Add failing promotion tests**

Append to `tests/unit/apps/change-manager.test.ts`:

```ts
test("promotes draft to live app and commits change", async () => {
  const workspace = new AppWorkspace(root);
  await workspace.init();
  const registry = new AppRegistry(workspace, "/tmp/home");
  const manager = new AppChangeManager(workspace, registry);
  const draft = await manager.createDraft({ appId: "weather", userId: "123" });
  writeFileSync(join(draft.path, "app.ts"), "export default {};\n");

  await manager.promoteDraft(draft, { approvingUserId: "123", summary: "Test promotion" });

  expect(existsSync(join(root, "live", "weather", "app.ts"))).toBe(true);
  expect(workspace.statusPorcelain()).toBe("");
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/unit/apps/change-manager.test.ts`

Expected: FAIL because `promoteDraft` does not exist.

- [ ] **Step 3: Implement promotion and rollback marker**

Add to `AppChangeManager`:

```ts
  async promoteDraft(draft: AppDraft, input: { approvingUserId: string; summary: string }): Promise<void> {
    if (this.workspace.statusPorcelain().trim()) {
      throw new Error("Apps workspace has uncommitted changes; resolve before promotion");
    }
    const livePath = this.workspace.liveAppPath(draft.appId);
    const backupRef = this.currentHead();
    rmSync(livePath, { recursive: true, force: true });
    mkdirSync(livePath, { recursive: true });
    cpSync(draft.path, livePath, { recursive: true });
    this.workspace.git(["add", "live", "drafts"]);
    this.workspace.git(["commit", "-m", `Promote ${draft.appId} draft ${draft.draftId}: ${input.summary} [approved-by=${input.approvingUserId}]`], { allowNoCommit: true });
    try {
      await this.registry.reloadLiveApps();
    } catch (err) {
      this.workspace.git(["reset", "--hard", backupRef]);
      await this.registry.reloadLiveApps();
      throw err;
    }
  }

  rollbackTo(ref: string): void {
    this.workspace.git(["reset", "--hard", ref]);
  }

  private currentHead(): string {
    return this.workspace.git(["rev-parse", "HEAD"]).trim();
  }
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/change-manager.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/change-manager.ts tests/unit/apps/change-manager.test.ts
git commit -m "Add draft promotion and rollback"
```

### T18: Draft workflow tools

**Files:**
- Modify: `src/apps/tools.ts`
- Test: `tests/unit/apps/runtime.test.ts`

- [ ] **Step 1: Add failing draft tool tests**

Append to `tests/unit/apps/runtime.test.ts`:

```ts
test("core app tools expose draft workflow", async () => {
  const changeManager = {
    createDraft: async () => ({ draftId: "weather-1", appId: "weather", path: "/tmp/draft", userId: "123" }),
    validateDraft: async () => ({ ok: true, appId: "weather", stages: [], guidance: [] }),
    promoteDraft: async () => undefined,
  };
  const tools = createAppTools({ runtime: {} as any, registry: { catalog: () => [] } as any, changeManager: changeManager as any });
  expect(await tools.create_app_draft.run({ appId: "weather", userId: "123" }, {} as any)).toMatchObject({ draftId: "weather-1" });
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/unit/apps/runtime.test.ts`

Expected: FAIL because draft tools do not exist.

- [ ] **Step 3: Extend `createAppTools()`**

Add optional dependency:

```ts
changeManager?: AppChangeManager;
```

Add tools:

```ts
    create_app_draft: {
      name: "create_app_draft",
      description: "Create a git-backed draft for editing an app.",
      inputSchema: { type: "object", required: ["appId", "userId"], properties: { appId: { type: "string" }, userId: { type: "string" } }, additionalProperties: false },
      run: async (input) => {
        if (!deps.changeManager) throw new Error("App change manager is not configured");
        return deps.changeManager.createDraft(input as { appId: string; userId: string });
      },
    },
    verify_app_draft: {
      name: "verify_app_draft",
      description: "Run App Verifier for a draft.",
      inputSchema: { type: "object", required: ["draftId", "appId", "path", "userId"], properties: { draftId: { type: "string" }, appId: { type: "string" }, path: { type: "string" }, userId: { type: "string" } }, additionalProperties: false },
      run: async (input) => {
        if (!deps.changeManager) throw new Error("App change manager is not configured");
        return deps.changeManager.validateDraft(input as any);
      },
    },
    promote_app_draft: {
      name: "promote_app_draft",
      description: "Promote an approved app draft to live.",
      inputSchema: { type: "object", required: ["draft", "approvingUserId", "summary"], properties: { draft: { type: "object" }, approvingUserId: { type: "string" }, summary: { type: "string" } }, additionalProperties: false },
      run: async (input) => {
        if (!deps.changeManager) throw new Error("App change manager is not configured");
        const payload = input as any;
        await deps.changeManager.promoteDraft(payload.draft, { approvingUserId: payload.approvingUserId, summary: payload.summary });
        return { ok: true };
      },
    },
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/apps/runtime.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/tools.ts tests/unit/apps/runtime.test.ts
git commit -m "Add app draft workflow tools"
```

---

## Phase F - Minimal Display Client (T19-T21)

### T19: Display state

**Files:**
- Create: `src/display/state.ts`
- Test: `tests/unit/display/state.test.ts`

- [ ] **Step 1: Write failing display state tests**

```ts
// tests/unit/display/state.test.ts
import { describe, expect, test } from "bun:test";
import { DisplayState } from "../../../src/display/state";

describe("DisplayState", () => {
  test("restores active app before dashboard", () => {
    const state = new DisplayState();
    expect(state.current().screen).toBe("dashboard");
    state.openApp("weather");
    expect(state.current()).toMatchObject({ screen: "app", appId: "weather" });
  });

  test("tracks draft preview", () => {
    const state = new DisplayState();
    state.previewDraft("draft-1", "weather");
    expect(state.current()).toMatchObject({ screen: "preview", draftId: "draft-1", appId: "weather" });
  });
});
```

- [ ] **Step 2: Run failing tests**

Run: `bun test tests/unit/display/state.test.ts`

Expected: FAIL with module not found.

- [ ] **Step 3: Implement display state**

```ts
// src/display/state.ts
export type DisplaySnapshot =
  | { screen: "dashboard"; status?: string }
  | { screen: "app"; appId: string; status?: string }
  | { screen: "preview"; draftId: string; appId: string; status?: string }
  | { screen: "artifact"; artifactId: string; status?: string };

export class DisplayState {
  private snapshot: DisplaySnapshot = { screen: "dashboard" };

  current(): DisplaySnapshot {
    return this.snapshot;
  }

  openDashboard(): void {
    this.snapshot = { screen: "dashboard", status: this.snapshot.status };
  }

  openApp(appId: string): void {
    this.snapshot = { screen: "app", appId, status: this.snapshot.status };
  }

  previewDraft(draftId: string, appId: string): void {
    this.snapshot = { screen: "preview", draftId, appId, status: this.snapshot.status };
  }

  setStatus(status: string): void {
    this.snapshot = { ...this.snapshot, status };
  }
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/display/state.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/display/state.ts tests/unit/display/state.test.ts
git commit -m "Add minimal display state"
```

### T20: Minimal HTML renderer

**Files:**
- Create: `src/display/render.ts`
- Test: `tests/unit/display/render.test.ts`

- [ ] **Step 1: Write failing renderer tests**

```ts
// tests/unit/display/render.test.ts
import { describe, expect, test } from "bun:test";
import { renderDashboard, renderUiBlocks } from "../../../src/display/render";

describe("display renderer", () => {
  test("renders dashboard cards", () => {
    const html = renderDashboard([{ kind: "card", title: "Weather", blocks: [{ type: "text", text: "18 C" }] }]);
    expect(html).toContain("Weather");
    expect(html).toContain("18 C");
  });

  test("escapes text", () => {
    expect(renderUiBlocks([{ type: "text", text: "<script>" }])).toContain("&lt;script&gt;");
  });
});
```

- [ ] **Step 2: Run failing tests**

Run: `bun test tests/unit/display/render.test.ts`

Expected: FAIL with module not found.

- [ ] **Step 3: Implement renderer**

```ts
// src/display/render.ts
import type { AppCard, AppScreen, UiBlock } from "../apps/types";

export function renderDashboard(cards: AppCard[]): string {
  return page("Paddelino", `<h1>Paddelino</h1><main class="cards">${cards.map(renderCard).join("\n")}</main>`);
}

export function renderScreen(screen: AppScreen): string {
  return page(screen.title, `<main><h1>${escapeHtml(screen.title)}</h1>${renderUiBlocks(screen.blocks)}</main>`);
}

export function renderCard(card: AppCard): string {
  return `<section class="card"><h2>${escapeHtml(card.title)}</h2>${card.subtitle ? `<p>${escapeHtml(card.subtitle)}</p>` : ""}${renderUiBlocks(card.blocks)}</section>`;
}

export function renderUiBlocks(blocks: UiBlock[]): string {
  return blocks.map(block => {
    switch (block.type) {
      case "text": return `<p>${escapeHtml(block.text)}</p>`;
      case "section": return `<section><h3>${escapeHtml(block.title)}</h3>${renderUiBlocks(block.blocks)}</section>`;
      case "keyValue": return `<dl>${block.items.map(item => `<dt>${escapeHtml(item.key)}</dt><dd>${escapeHtml(item.value)}</dd>`).join("")}</dl>`;
      case "list": return `<ul>${block.items.map(item => `<li>${escapeHtml(item)}</li>`).join("")}</ul>`;
      case "table": return `<table><thead><tr>${block.columns.map(col => `<th>${escapeHtml(col)}</th>`).join("")}</tr></thead><tbody>${block.rows.map(row => `<tr>${row.map(cell => `<td>${escapeHtml(cell)}</td>`).join("")}</tr>`).join("")}</tbody></table>`;
      case "button": return `<button data-action="${escapeHtml(block.action)}">${escapeHtml(block.label)}</button>`;
      case "status": return `<p class="status ${block.status}">${escapeHtml(block.message)}</p>`;
      case "error": return `<p class="error">${escapeHtml(block.message)}</p>`;
    }
  }).join("\n");
}

function page(title: string, body: string): string {
  return `<!doctype html><html><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>${escapeHtml(title)}</title><style>body{font-family:system-ui;margin:2rem;background:#111;color:#eee}.card{border:1px solid #444;border-radius:12px;padding:1rem;margin:1rem 0}table{border-collapse:collapse}td,th{border:1px solid #444;padding:.4rem}</style></head><body>${body}</body></html>`;
}

function escapeHtml(value: string): string {
  return value.replaceAll("&", "&amp;").replaceAll("<", "&lt;").replaceAll(">", "&gt;").replaceAll('"', "&quot;");
}
```

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/unit/display/render.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/display/render.ts tests/unit/display/render.test.ts
git commit -m "Add minimal display renderer"
```

### T21: Display server and display tools

**Files:**
- Create: `src/display/server.ts`
- Modify: `src/apps/tools.ts`
- Test: `tests/integration/apps/display-actions.test.ts`

- [ ] **Step 1: Write failing display integration test**

```ts
// tests/integration/apps/display-actions.test.ts
import { describe, expect, test } from "bun:test";
import { DisplayState } from "../../../src/display/state";
import { createAppTools } from "../../../src/apps/tools";

describe("display actions", () => {
  test("open_app and set_display_status update display state", async () => {
    const displayState = new DisplayState();
    const tools = createAppTools({ runtime: {} as any, registry: { catalog: () => [] } as any, displayState });

    await tools.open_app.run({ appId: "weather" }, {} as any);
    await tools.set_display_status.run({ status: "Checking weather" }, {} as any);

    expect(displayState.current()).toMatchObject({ screen: "app", appId: "weather", status: "Checking weather" });
  });
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/integration/apps/display-actions.test.ts`

Expected: FAIL because display tools do not exist.

- [ ] **Step 3: Implement display server**

```ts
// src/display/server.ts
import type { AppRegistry } from "../apps/registry";
import type { AppRuntime } from "../apps/runtime";
import { renderDashboard, renderScreen } from "./render";
import type { DisplayState } from "./state";

export class DisplayServer {
  private server?: ReturnType<typeof Bun.serve>;

  constructor(private readonly deps: { port: number; state: DisplayState; registry: AppRegistry; runtime: AppRuntime }) {}

  start(): void {
    if (this.server) return;
    this.server = Bun.serve({
      port: this.deps.port,
      fetch: async (request) => this.handle(request),
    });
  }

  stop(): void {
    this.server?.stop();
    this.server = undefined;
  }

  private async handle(request: Request): Promise<Response> {
    const url = new URL(request.url);
    if (url.pathname.startsWith("/app/")) {
      const appId = url.pathname.split("/")[2];
      const screen = await this.deps.runtime.renderScreen(appId);
      return html(renderScreen(screen as any));
    }
    const cards = [];
    for (const app of this.deps.registry.catalog()) {
      cards.push(await this.deps.runtime.renderCard(app.appId) as any);
    }
    return html(renderDashboard(cards));
  }
}

function html(body: string): Response {
  return new Response(body, { headers: { "content-type": "text/html; charset=utf-8" } });
}
```

- [ ] **Step 4: Add display tools**

Extend `createAppTools()` dependency type with:

```ts
displayState?: DisplayState;
```

Add tools:

```ts
    open_app: {
      name: "open_app",
      description: "Open an app on the local display.",
      inputSchema: { type: "object", required: ["appId"], properties: { appId: { type: "string" } }, additionalProperties: false },
      run: async (input) => {
        if (!deps.displayState) throw new Error("Display state is not configured");
        deps.displayState.openApp((input as { appId: string }).appId);
        return { ok: true };
      },
    },
    set_display_status: {
      name: "set_display_status",
      description: "Set short status text on the local display.",
      inputSchema: { type: "object", required: ["status"], properties: { status: { type: "string" } }, additionalProperties: false },
      run: async (input) => {
        if (!deps.displayState) throw new Error("Display state is not configured");
        deps.displayState.setStatus((input as { status: string }).status);
        return { ok: true };
      },
    },
```

- [ ] **Step 5: Verify and commit**

Run: `bun test tests/integration/apps/display-actions.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/display/server.ts src/apps/tools.ts tests/integration/apps/display-actions.test.ts
git commit -m "Add minimal display server and display tools"
```

---

## Phase G - Wiring And End-to-End Lifecycle (T22-T24)

### T22: Wire SP2 runtime in `src/index.ts`

**Files:**
- Modify: `src/index.ts`
- Test: `tests/integration/apps/app-lifecycle.test.ts`

- [ ] **Step 1: Write lifecycle integration test**

```ts
// tests/integration/apps/app-lifecycle.test.ts
import { afterEach, beforeEach, describe, expect, test } from "bun:test";
import { mkdtempSync, mkdirSync, rmSync, writeFileSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { AppWorkspace } from "../../../src/apps/workspace";
import { AppRegistry } from "../../../src/apps/registry";
import { AppRuntime } from "../../../src/apps/runtime";
import { AppChangeManager } from "../../../src/apps/change-manager";

let root: string;
beforeEach(() => { root = mkdtempSync(join(tmpdir(), "paddelino-e2e-")); });
afterEach(() => { rmSync(root, { recursive: true, force: true }); });

describe("app lifecycle", () => {
  test("load, invoke, draft, preview, promote", async () => {
    const workspace = new AppWorkspace(root);
    await workspace.init();
    const appDir = workspace.liveAppPath("demo");
    mkdirSync(appDir, { recursive: true });
    writeFileSync(join(appDir, "app.ts"), `
      import { defineApp } from "${process.cwd()}/src/apps/define-app";
      export default defineApp({
        appId: "demo", name: "Demo", version: "0.1.0", description: "Demo app.",
        capabilities: { summary: "Demo.", examples: ["demo"] }, permissions: ["display", "tool"],
        display: { card: async () => ({ kind: "card", title: "Demo", blocks: [] }), screen: async () => ({ kind: "screen", title: "Demo", blocks: [] }) },
        healthCheck: async () => ({ status: "ok", message: "ready" }),
        invoke: async () => ({ text: "demo" }),
      });
    `);
    workspace.git(["add", "live/demo/app.ts"]);
    workspace.git(["commit", "-m", "Add demo app"]);

    const registry = new AppRegistry(workspace, "/tmp/home");
    await registry.reloadLiveApps();
    const runtime = new AppRuntime(registry, "/tmp/home");
    expect(await runtime.invokeApp("demo", { prompt: "demo", channel: "telegram" })).toEqual({ text: "demo" });

    const manager = new AppChangeManager(workspace, registry);
    const draft = await manager.createDraft({ appId: "demo", userId: "123" });
    const report = await manager.validateDraft(draft);
    expect(report.ok).toBe(true);
    await manager.promoteDraft(draft, { approvingUserId: "123", summary: "No-op" });
    expect(workspace.statusPorcelain()).toBe("");
  });
});
```

- [ ] **Step 2: Run integration test**

Run: `bun test tests/integration/apps/app-lifecycle.test.ts`

Expected: PASS if earlier tasks are complete.

- [ ] **Step 3: Wire runtime in `src/index.ts`**

Add imports:

```ts
import { AppWorkspace } from "./apps/workspace";
import { AppRegistry } from "./apps/registry";
import { AppRuntime } from "./apps/runtime";
import { AppChangeManager } from "./apps/change-manager";
import { createAppTools } from "./apps/tools";
import { DisplayState } from "./display/state";
import { DisplayServer } from "./display/server";
```

Before `AgentManager` construction:

```ts
  const appWorkspace = new AppWorkspace(config.paths.apps);
  await appWorkspace.init();
  const appRegistry = new AppRegistry(appWorkspace, config.paths.home);
  await appRegistry.reloadLiveApps();
  const appRuntime = new AppRuntime(appRegistry, config.paths.home);
  const appChangeManager = new AppChangeManager(appWorkspace, appRegistry);
  const displayState = new DisplayState();
  const appTools = createAppTools({ registry: appRegistry, runtime: appRuntime, changeManager: appChangeManager, displayState });
```

Pass into `AgentManager`:

```ts
    extraTools: appTools,
    appCatalog: () => appRegistry.catalog(),
```

After Telegram channel setup:

```ts
  const display = new DisplayServer({ port: config.display.port, state: displayState, registry: appRegistry, runtime: appRuntime });
  display.start();
```

In signal handlers, call `display.stop()` if it has been created. The weather app seed is added in T23, after the seed-source helper exists.

- [ ] **Step 4: Verify and commit**

Run: `bun test tests/integration/apps/app-lifecycle.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/index.ts tests/integration/apps/app-lifecycle.test.ts
git commit -m "Wire app platform runtime"
```

### T23: Add robust weather seed source

**Files:**
- Create: `src/apps/weather/seed.ts`
- Modify: `src/index.ts`
- Test: `tests/unit/apps/weather.test.ts`

- [ ] **Step 1: Add failing seed source test**

Append to `tests/unit/apps/weather.test.ts`:

```ts
import { createWeatherAppSeedSource } from "../../../src/apps/weather/seed";

test("weather seed source imports core weather app by absolute path", () => {
  const source = createWeatherAppSeedSource("/repo/paddelino", { latitude: 48.2082, longitude: 16.3738, label: "Vienna" });
  expect(source).toContain("/repo/paddelino/src/apps/weather/app");
  expect(source).toContain("createWeatherApp");
  expect(source).toContain("label: \"Vienna\"");
});
```

- [ ] **Step 2: Run failing test**

Run: `bun test tests/unit/apps/weather.test.ts`

Expected: FAIL because seed module does not exist.

- [ ] **Step 3: Implement seed source**

```ts
// src/apps/weather/seed.ts
import type { WeatherLocation } from "./types";

export function createWeatherAppSeedSource(coreRoot: string, location: WeatherLocation): string {
  const appModulePath = `${coreRoot.replaceAll("\\", "/")}/src/apps/weather/app`;
  return `
import { createWeatherApp } from "${appModulePath}";

export default createWeatherApp({
  defaultLocation: { latitude: ${location.latitude}, longitude: ${location.longitude}, label: ${JSON.stringify(location.label)} },
});
`;
}
```

- [ ] **Step 4: Update `src/index.ts` seed import**

Replace text import with:

```ts
import { createWeatherAppSeedSource } from "./apps/weather/seed";
```

Replace seed call with:

```ts
  const weatherSeedLocation = config.weather.defaultLocation ?? { latitude: 48.2082, longitude: 16.3738, label: "Vienna" };
  await appWorkspace.seedBundledApp("weather", createWeatherAppSeedSource(process.cwd(), weatherSeedLocation));
  await appRegistry.reloadLiveApps();
```

- [ ] **Step 5: Verify and commit**

Run: `bun test tests/unit/apps/weather.test.ts && bun run typecheck`

Expected: PASS and typecheck exits 0.

```bash
git add src/apps/weather/seed.ts src/index.ts tests/unit/apps/weather.test.ts
git commit -m "Add weather app seed source"
```

### T24: Full SP2 check

**Files:**
- Modify only files required by failures from the commands below.

- [ ] **Step 1: Run complete test suite**

Run: `bun test`

Expected: all tests PASS.

- [ ] **Step 2: Run typecheck**

Run: `bun run typecheck`

Expected: exit 0.

- [ ] **Step 3: Run app startup smoke test without Telegram polling**

If no dry-run mode exists, add `PADDELINO_DRY_RUN=1` in `src/index.ts` so the app initializes config, workspace, registry, and display server, logs success, then exits before Telegram polling.

Expected command:

```bash
PADDELINO_DRY_RUN=1 TELEGRAM_BOT_TOKEN=test TELEGRAM_ALLOWED_USERS=123 ANTHROPIC_API_KEY=test PADDELINO_HOME=/tmp/paddelino-sp2-smoke bun run src/index.ts
```

Expected output contains: `paddelino dry run ok`.

- [ ] **Step 4: Commit smoke-mode changes if needed**

```bash
git add src/index.ts tests
git commit -m "Add SP2 startup smoke coverage"
```

---

## Phase H - Documentation And Operational UX (T25-T26)

### T25: README and smoke checklist

**Files:**
- Modify: `README.md`
- Modify: `docs/smoke-checklist.md`

- [ ] **Step 1: Add README SP2 section**

Add:

```markdown
## Agent OS Apps (SP2)

paddelino can load local apps from `~/.paddelino/apps`. Apps are git-tracked TypeScript modules that can contribute display cards, screens, tools, route hints, prompt delegation, and simple interval jobs.

The bundled blueprint app is `weather`, seeded into the app workspace on first run. It uses Open-Meteo and requires no API key. Configure the default location with:

```bash
PADDELINO_WEATHER_LATITUDE=48.2082
PADDELINO_WEATHER_LONGITUDE=16.3738
PADDELINO_WEATHER_LABEL=Vienna
```

The local display is available at `http://localhost:4123` by default. It is intentionally minimal in SP2 and supports dashboard cards, app screens, draft previews, and app health/errors.

Agent-authored app changes use drafts: create draft, edit draft, verify, preview, approve, promote. Live app files are not modified directly.
```

- [ ] **Step 2: Add smoke checklist entries**

Add to `docs/smoke-checklist.md`:

```markdown
## SP2 App Platform Smoke Check

- [ ] Start paddelino with a temporary `PADDELINO_HOME`.
- [ ] Confirm `~/.paddelino/apps` is a git repo with `live/weather/app.ts`.
- [ ] Open `http://localhost:4123` and see the dashboard.
- [ ] Confirm the Weather card renders or shows a clear provider error.
- [ ] Ask Telegram: "what is the weather" and confirm the agent uses the weather app or `invoke_app`.
- [ ] Ask the agent to create a draft change to the weather app copy.
- [ ] Run App Verifier and confirm a pass/fail report appears.
- [ ] Preview the draft without changing the live weather app.
- [ ] Approve promotion and confirm git contains a promotion commit.
- [ ] Roll back the weather app and confirm the previous version reloads.
```

- [ ] **Step 3: Verify and commit**

Run: `bun run typecheck`

Expected: exit 0.

```bash
git add README.md docs/smoke-checklist.md
git commit -m "Document SP2 app platform"
```

### T26: Final verification and handoff

**Files:**
- No code changes expected.

- [ ] **Step 1: Run final checks**

```bash
bun run typecheck
bun test
PADDELINO_DRY_RUN=1 TELEGRAM_BOT_TOKEN=test TELEGRAM_ALLOWED_USERS=123 ANTHROPIC_API_KEY=test PADDELINO_HOME=/tmp/paddelino-sp2-final bun run src/index.ts
```

Expected: typecheck exits 0, tests pass, dry run prints `paddelino dry run ok`.

- [ ] **Step 2: Inspect git status**

Run: `git status --short`

Expected: no uncommitted changes.

- [ ] **Step 3: Record completion notes**

Add a short implementation summary to the final response:

```text
Implemented SP2 app platform: app contract, workspace, verifier, registry, runtime, weather blueprint app, draft/preview/promotion workflow, app tools, minimal display server, docs, and smoke checklist. Verification: bun run typecheck, bun test, dry-run startup.
```

---

## Self-Review

Spec coverage:
- `~/.paddelino/apps` git workspace: T05, T11, T16, T17.
- App contract and structured UI: T01-T03.
- Dynamic load/hot reload: T07, T17, T22.
- Draft/preview/promotion/rollback: T16-T18.
- App Verifier: T06 plus draft tool in T18.
- Weather blueprint app: T09-T11.
- App-aware routing/delegation/tools/jobs: T12-T15, T10 jobs field.
- Minimal display client: T19-T21.
- Telegram fallback/control path: core tools in T13/T18 and existing Telegram channel integration in T14/T22.
- Tests and verification: T01-T26.

Implementation decisions made:
- Drafts are git-backed directories under `drafts/` with commits, not separate worktrees in v1. This satisfies git tracking throughout and can be refactored to worktrees later.
- Display transport starts with plain HTTP. WebSocket/SSE live updates are deferred to the first point where polling becomes insufficient; the display state/tools are designed so live updates can be added without changing app contracts.
- Structured UI schema is a small TypeScript union validated by runtime helpers.
- Weather location defaults to env config with Vienna fallback in seed code; later setup UX can improve this.
- Route matching starts with simple example overlap; agentic delegation through app catalog and `invoke_app` handles non-exact matches.
